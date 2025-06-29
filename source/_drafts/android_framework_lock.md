title: android 锁
author: nuoen
tags: []
categories:
  - cpp 
  - android
  - framework
date: 2025-03-29 21:39:00

## AutoMutex
场景：
```cpp
extern Mutex gDefaultServiceManagerLock;

sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != nullptr) return gDefaultServiceManager;

    {
        AutoMutex _l(gDefaultServiceManagerLock);
        while (gDefaultServiceManager == nullptr) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(nullptr));
            if (gDefaultServiceManager == nullptr)
                sleep(1);
        }
    }

    return gDefaultServiceManager;
}
```
图示：
```
          Multiple Threads Call defaultServiceManager()
                         ↓
         ┌──────────────────────────────────────┐
         │ if (gDefaultServiceManager != nullptr)│  ← 已初始化就直接返回
         └───────────────┬──────────────────────┘
                         │
                         ▼
         ┌──────────────────────────────────────┐
         │ AutoMutex _l(gDefaultServiceManagerLock); │ ← 所有线程都要在此排队加锁
         └───────────────┬──────────────────────┘
                         │
        ┌────────────────┴────────────────────────────┐
        │                                             │
┌───────▼───────┐                          ┌──────────▼─────────┐
│Thread A 先获得锁│                          │Thread B、C 等待锁 │
└───────┬───────┘                          └────────────────────┘
        │
        │ gDefaultServiceManager == nullptr? 是
        ▼
┌──────────────────────────────────────────────┐
│Thread A 尝试创建 ServiceManager 实例          │
│gDefaultServiceManager = interface_cast(...)  │
└──────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────┐
│ 初始化成功，释放锁 AutoMutex 析构自动 unlock │
└──────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────┐
│ Thread B/C 被唤醒，加锁后检查变量不为 null   │
│ if (gDefaultServiceManager != nullptr) return│
└──────────────────────────────────────────────┘
```
•	这是典型的线程安全 单例懒加载（lazy initialization）；
•	用锁保护了“只初始化一次”的语义；
•	整体非常适合用于系统服务的获取场景，如 IServiceManager。

实现原理
system/core/include/utils/Mutex.h
```cpp
typedef Mutex::Autolock AutoMutex;

    class SCOPED_CAPABILITY Autolock {
      public:
        inline explicit Autolock(Mutex& mutex) ACQUIRE(mutex) : mLock(mutex) { mLock.lock(); }
        inline explicit Autolock(Mutex* mutex) ACQUIRE(mutex) : mLock(*mutex) { mLock.lock(); }
        inline ~Autolock() RELEASE() { mLock.unlock(); }

      private:
        Mutex& mLock;
        // Cannot be copied or moved - declarations only
        Autolock(const Autolock&);
        Autolock& operator=(const Autolock&);
    };

class CAPABILITY("mutex") Mutex {
  public:
    enum {
        PRIVATE = 0,
        SHARED = 1
    };

    Mutex();
    explicit Mutex(const char* name);
    explicit Mutex(int type, const char* name = nullptr);
    ~Mutex();

    // lock or unlock the mutex
    status_t lock() ACQUIRE();
    void unlock() RELEASE();

        pthread_mutex_t mMutex;

}

inline status_t Mutex::lock() {
    return -pthread_mutex_lock(&mMutex);
}

inline Mutex::~Mutex() {
    pthread_mutex_destroy(&mMutex);
}
```
调用逻辑就是：
1. 声明一个 AutoMutex 变量，也就是Autolock -> Mutex& mLock;
2. Mutex& mLock 变量，mLock->pthread_mutex_t 
3. AutoMutex _l(gDefaultServiceManagerLock); 构造时，调用pthread_mutex_lock 加锁；
4. 变量 _l,在作用域结束后，会调用其析构函数,从而调用pthread_mutex_destroy ，从而自动解锁；
附录：
## 作用域

✅ 什么是作用域？
	•	在 C++ 中，作用域是 { ... } 大括号包围的代码块。
	•	变量（比如 AutoMutex _l）的生命周期限定在这个代码块内。
	•	一旦出了这个作用域，变量会被自动析构（即释放资源）。

⸻

🎯 AutoMutex 的作用域在哪？
```cpp
{
    AutoMutex _l(gDefaultServiceManagerLock);
    ...
}
```
	•	开始：AutoMutex _l(...) 这一行
	•	结束：紧接着的 } 结束大括号
	•	这个作用域内加锁，退出作用域时自动解锁

⸻

🔄 生命周期过程图解

defaultServiceManager()
┌────────────────────────────────────────────┐
│ if (gDefaultServiceManager != nullptr) ...│ ← 已初始化直接返回
├────────────────────────────────────────────┤
│ {                                          │ ← 进入作用域（加锁）
│   AutoMutex _l(lock);                      │ ← 构造，加锁
│   while (...) { ... }                      │
│ }                                          │ ← 退出作用域（析构 -> 解锁）
├────────────────────────────────────────────┤
│ return gDefaultServiceManager;            │
└────────────────────────────────────────────┘



⸻

🚦 为什么不整个函数都锁住？
	•	如果把 AutoMutex _l(...) 放在函数顶部，即使变量已初始化，也会加锁等待，浪费性能。
	•	通过作用域限制，只在 gDefaultServiceManager 为 nullptr 时才加锁，更高效。

⸻

✅ 总结表格
问题	解答
AutoMutex 的作用域是？	包围其定义的大括号 { ... }
何时开始和结束？	从 AutoMutex _l(...) 开始，到其对应的 } 结束
为何用作用域包住？	自动加解锁，确保线程安全，同时提高性能
退出作用域后会发生什么？	调用 ~AutoMutex() → 自动调用 unlock() 解锁
⸻