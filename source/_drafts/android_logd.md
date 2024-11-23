为了深入理解 LOG(INFO) << "init first stage started!" 的执行流程，我们需要从宏的解析和执行流程一步步分析，直到最终消息被记录在日志中。

1. 解析 LOG(INFO) << "init first stage started!"

LOG(INFO) << "init first stage started!" 是一个宏调用，首先解析 LOG 宏定义：
```cpp
#define LOG(severity) LAZY_STREAM(LOG_STREAM(severity), LOG_IS_ON(severity))
```
根据这句定义，LOG(INFO) 会展开为：
```cpp
LAZY_STREAM(LOG_STREAM(INFO), LOG_IS_ON(INFO))
```
接下来需要依次解析 LAZY_STREAM、LOG_STREAM 和 LOG_IS_ON。

2. 宏解析顺序

2.1 LOG_IS_ON 宏解析

首先解析 LOG_IS_ON(INFO)，其定义如下：
```cpp
#define LOG_IS_ON(severity) (::logging::ShouldCreateLogMessage(::logging::LOG_##severity))
```
因为 INFO 对应的日志级别常量是 LOG_INFO，所以 LOG_IS_ON(INFO) 会进一步展开为：
```cpp
::logging::ShouldCreateLogMessage(::logging::LOG_INFO)
```
ShouldCreateLogMessage 是一个用于判断是否需要记录日志的函数，其实现如下：
```cpp
bool ShouldCreateLogMessage(int severity) {
  if (severity < g_min_log_level)
    return false;
  return g_logging_destination != LOG_NONE || log_message_handler || severity >= kAlwaysPrintErrorLevel;
}
```
如果当前日志级别满足输出条件，这个函数会返回 true。

2.2 LOG_STREAM 宏解析

如果 LOG_IS_ON(INFO) 返回 true，我们会继续解析 LOG_STREAM(INFO)：
```cpp
#define LOG_STREAM(severity) COMPACT_GOOGLE_LOG_##severity.stream()
```
这里，COMPACT_GOOGLE_LOG_##severity 将拼接成 COMPACT_GOOGLE_LOG_INFO，所以 LOG_STREAM(INFO) 最终展开为：
```cpp
COMPACT_GOOGLE_LOG_INFO.stream()
```
2.3 COMPACT_GOOGLE_LOG_INFO 宏解析

COMPACT_GOOGLE_LOG_INFO 被定义为：
```cpp
#define COMPACT_GOOGLE_LOG_INFO COMPACT_GOOGLE_LOG_EX_INFO(LogMessage)
```
这个宏调用了 COMPACT_GOOGLE_LOG_EX_INFO，所以我们继续展开。

2.4 COMPACT_GOOGLE_LOG_EX_INFO 宏解析

COMPACT_GOOGLE_LOG_EX_INFO 的定义如下：
```cpp
#define COMPACT_GOOGLE_LOG_EX_INFO(ClassName, ...) \
  ::logging::ClassName(__FILE__, __LINE__, ::logging::LOG_INFO, ##__VA_ARGS__)
```
这会创建一个 LogMessage 类的对象，传递当前文件名、行号和日志级别。因此 COMPACT_GOOGLE_LOG_INFO.stream() 会展开为：
```cpp
::logging::LogMessage(__FILE__, __LINE__, ::logging::LOG_INFO).stream()
```
到这里，宏解析完成。LogMessage 类的构造函数会初始化日志的文件、行号和日志级别，并返回 stream()，我们可以将消息内容通过 << 发送到流中。

3. 执行 LogMessage 类构造与日志记录

3.1 LogMessage 构造函数初始化

LogMessage 的构造函数会将文件、行号和日志级别存储到对象中，并执行 Init 函数来初始化日志流：
```cpp
LogMessage::LogMessage(const char* file, int line, LogSeverity severity)
    : severity_(severity), file_(file), line_(line) {
  Init(file, line);
}
```
3.2 Init 函数格式化日志头

在 Init 函数中，日志消息会添加头部信息（包括时间戳、进程和线程 ID 等），例如：
```cpp
void LogMessage::Init(const char* file, int line) {
  stream_ << '[';
  if (g_log_process_id)
    stream_ << CurrentProcessId() << ':';
  if (g_log_thread_id)
    stream_ << base::PlatformThread::CurrentId() << ':';
  if (g_log_timestamp) {
    struct timeval tv;
    gettimeofday(&tv, nullptr);
    time_t t = tv.tv_sec;
    struct tm local_time;
    localtime_r(&t, &local_time);
    stream_ << std::setfill('0')
            << std::setw(2) << 1 + local_time.tm_mon
            << std::setw(2) << local_time.tm_mday
            << '/'
            << std::setw(2) << local_time.tm_hour
            << std::setw(2) << local_time.tm_min
            << std::setw(2) << local_time.tm_sec
            << '.'
            << std::setw(6) << tv.tv_usec
            << ':';
  }
  if (g_log_tickcount)
    stream_ << TickCount() << ':';
  stream_ << log_severity_name(severity_) << ":" << file << "(" << line << ")] ";
  message_start_ = stream_.str().length();
}
```
此代码会将日志头部信息输出到 stream_ 中。

3.3 ~LogMessage 析构函数输出日志

当 LogMessage 对象析构时，~LogMessage 会调用日志输出流程：
```cpp
LogMessage::~LogMessage() {
  std::string str_newline(stream_.str());
  if (log_message_handler &&
      log_message_handler(severity_, file_, line_,
                          message_start_, str_newline)) {
    return;
  }
  if ((g_logging_destination & LOG_TO_SYSTEM_DEBUG_LOG) != 0) {
#if defined(OS_ANDROID)
    __android_log_write(ANDROID_LOG_INFO, "chromium", str_newline.c_str());
#else
    ignore_result(fwrite(str_newline.data(), str_newline.size(), 1, stderr));
    fflush(stderr);
#endif
  }
  if ((g_logging_destination & LOG_TO_FILE) != 0) {
    LoggingLock logging_lock;
    if (InitializeLogFileHandle()) {
      ignore_result(fwrite(str_newline.data(), str_newline.size(), 1, g_log_file));
      fflush(g_log_file);
    }
  }
  if (severity_ == LOG_FATAL) {
    base::debug::BreakDebugger();
  }
}
```
	•	在 Android 上，使用 __android_log_write 写入系统日志。
	•	在其他平台上，会根据配置写入 stderr 或日志文件。

总结

	1.	宏展开：LOG(INFO) << "init first stage started!" 依次展开为 LAZY_STREAM、LOG_STREAM、COMPACT_GOOGLE_LOG_INFO，最终生成一个 LogMessage 对象并调用其 stream()。
	2.	构造与格式化：LogMessage 对象在构造时会调用 Init 函数，格式化头部信息。
	3.	日志输出：对象析构时，~LogMessage 检查配置，选择写入到系统日志、stderr 或文件中。