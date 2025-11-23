

## 环境和编译
见 aosp_kernel.md

## ebpf运行
sudo apt install libbfd-dev clang llvm libcap-dev

android 编译
aosp 编译


### 准备工作
1. https://github.com/facebookexperimental/ExtendedAndroidTools
2. busybox 
```shell
adb push busybox /data/local/tmp/
adb shell
cd /data/local/tmp

# 1) 给执行权限
chmod +x busybox

# 2) 先建存放软链接的目录
mkdir -p /data/local/tmp/busybox-bin

# 3) 正确调用【注意前缀是 ./busybox，而不是 /busybox】
./busybox --install -s /data/local/tmp/busybox-bin

# 4) 把它临时加入 PATH
export PATH=$PATH:/data/local/tmp/busybox-bin

# 5) 验证
which ls
ls --help | head -n 1
busybox --list | head
```
3. 开启kptr_restrict 
moderntimes:/ # cat /proc/sys/kernel/kptr_restrict     0 1 2    
是一个 内核 sysctl 开关，用来限制内核把内核指针地址（kernel pointers）打印/导出到用户空间，以减少信息泄露、避免被用来绕过 KASLR 等防护。
* 0:不限制 允许在日志、/proc、/sys 等处显示真实内核地址。
* 1:受权限限制 只有拥有 CAP_SYSLOG（部分内核用 CAP_SYS_ADMIN 代替）的进程可以看到真实地址；其他进程看到被屏蔽/掩码后的地址。
* 2：严格隐藏 无论是否有特权，都隐藏真实地址（打印为 0 或掩码/哈希）。这是很多 Android user build 的默认。


4. 指定头文件                                 
需要开启 CONFIG_IKHEADERS=y ，在/data/local/tmp里找到kheaders.tar.xz

5.  开启ftrace的输出功能
```shell
cat /sys/kernel/tracing/tracing_on
echo 1 > /sys/kernel/tracing/tracing_on
```

两种不同的生态 BCC 和 libbpf
## BCC (BPF Compiler Collection)
* 一个基于 Python/C++ 的工具集，内部自带了一份 简化版的 libbpf API（所以会有 bcc/libbpf.h 这样的头文件）。
* 它屏蔽了很多底层细节，方便用 Python + embedded C 写 eBPF 程序。
* 如果你是用 BCC 工具（如 trace.py、funclatency.py），那你会看到 bcc/libbpf.h。




## libbpf (官方库)
* 来自 Linux 内核 tools/lib/bpf/，现在是 官方推荐 的用户态库。
* 头文件路径通常是 #include <bpf/libbpf.h>。
* API 更底层，但功能强大（CO-RE、BTF、map/prog 管理等）。
* 这是目前社区和内核开发者推荐的写法，未来也会取代 BCC 的内部接口。


## 关键词
### 探针
```shell
sudo bpftrace -l "kprobe:*" | wc -l
68651
```
•kprobe = Kernel Probe（内核探针）
•是 Linux 内核提供的一种动态追踪机制，可以在任意内核函数入口/出口插入探针，而无需重启或重新编译内核。
•常用于 性能分析、问题排查、内核调试。
#### eBPF Probe 类型对比表

| Probe 类型   | 作用位置              | 目标对象            | 特点/用途                                                                 | 示例（bpftrace） |
|--------------|----------------------|---------------------|---------------------------------------------------------------------------|-----------------|
| **kprobe**   | 内核函数入口 (entry) | 任意内核函数        | - 动态插桩，不用修改内核源码<br>- 可查看函数参数<br>- 依赖 `kallsyms`     | `kprobe:do_sys_open` |
| **kretprobe**| 内核函数返回 (return)| 任意内核函数        | - 捕获函数返回值<br>- 可计算函数耗时（配合 kprobe）                       | `kretprobe:do_sys_open` |
| **tracepoint**| 内核静态埋点位置     | 预定义事件点        | - 内核开发者定义，稳定性高<br>- 接口固定，不随内核版本轻易变化            | `tracepoint:syscalls:sys_enter_openat` |
| **uprobe**   | 用户态函数入口       | 用户程序/共享库函数 | - 用于跟踪用户态应用（ELF 二进制/动态库）<br>- 类似 kprobe 但针对用户空间 | `uprobe:/bin/bash:readline` |
| **uretprobe**| 用户态函数返回       | 用户程序/共享库函数 | - 捕获用户态函数返回值<br>- 可分析函数执行结果或耗时                       | `uretprobe:/bin/bash:readline` |

---

#### 使用场景举例
- **kprobe**：排查内核函数参数，比如谁在调用 `tcp_connect`。  
- **kretprobe**：测量内核函数执行耗时，比如一次 `vfs_read` 花了多久。  
- **tracepoint**：稳定的 syscall 监控，比如跟踪所有文件 open 调用。  
- **uprobe**：分析用户态程序调用，比如 `glibc` 的 `malloc`。  
- **uretprobe**：获取用户态函数返回值，比如 `open()` 返回的 fd。  

---
### 🧩 Linux `perf` 全面介绍

#### 一、什么是 perf

`perf` 是 Linux 内核自带的性能分析框架（Performance Counters for Linux，简称 PCL 或 perf events）。
它可以采集 **CPU 硬件事件**（如 cycles、cache misses）、
**内核事件**（如调度、上下文切换、syscall），以及
**用户程序的采样数据**，帮助分析性能瓶颈。

它有两部分组成：

| 组件 | 说明 |
|------|------|
| **perf 内核子系统** | 内核模块，提供 `/sys/kernel/debug/tracing` 与 `/proc/sys/kernel/perf_event_paranoid` 等接口，用于收集事件。 |
| **perf 用户态工具 (`perf`)** | 通常安装在 `/usr/bin/perf`，是 CLI 前端，用于配置、采样、记录与分析。 |

---

#### 二、核心功能模块

| 命令 | 功能说明 |
|------|-----------|
| `perf list` | 列出可用的性能事件（硬件、软件、tracepoint、eBPF）。 |
| `perf stat` | 统计事件计数（整体性能指标，如 cycles、instructions）。 |
| `perf top` | 实时显示占 CPU 比例最高的函数（类似 top + profiler）。 |
| `perf record` | 采样并记录事件数据到 perf.data 文件。 |
| `perf report` | 对 perf.data 进行离线分析和火焰图样式展示。 |
| `perf annotate` | 展示热点函数的汇编指令级采样。 |
| `perf trace` | 类似 strace，追踪系统调用与事件。 |
| `perf probe` | 动态注入 kprobe/uprobes 探针。 |
| `perf script` | 把 perf.data 转换成人类可读文本，供脚本进一步分析。 |

---

#### 三、常见使用场景与命令示例

##### 1️⃣ 整体性能统计
```bash
# 对命令整体性能统计
perf stat ls /tmp
```
## eBPF 指令集
https://www.kernel.org/doc/html/v6.1/bpf/instruction-set.html
https://github.com/iovisor/bpf-docs/blob/master/eBPF.md
## eBPF 验证器(verifier)
什么是 eBPF 验证器
* eBPF 验证器（verifier）是 Linux 内核中负责静态检查一个要加载到内核的 eBPF 程序是否安全、合法的一段逻辑。  ￼
* 它的目标是 防止错误 /不安全 /恶意的 BPF 程序进入内核，避免崩溃、挂起、非法访问内核内存、无限循环、越界访问、资源滥用等问题。  ￼
* 在用户空间调用 bpf() 系统调用加载 BPF 程序时，程序首先要通过验证器才能被接受进入内核。


## eBPF Map
什么是eBPF Map
	•	Map 是 eBPF 程序与用户态通信的桥梁。
	•	它是一个由内核维护的 键值存储 (key-value store)，所有 eBPF 程序都可以访问它。
	•	用户态进程也可以通过 bpf() 系统调用来读写这个 map。

`BPF_MAP_CREATE` 用于在内核中创建 eBPF Map 对象，作为 eBPF 程序与用户态交互的桥梁。

- **跨 eBPF 程序 / 用户态进程共享数据**
  - 用户态写入参数 → eBPF 程序读取
  - eBPF 程序统计计数 → 用户态读取结果
- 示例：
  ```c
  int map_fd = bpf_create_map(BPF_MAP_TYPE_HASH, sizeof(int), sizeof(long), 1024);
