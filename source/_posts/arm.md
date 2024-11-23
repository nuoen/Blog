title: ARM64
---

### CPSR 
实现金丝雀机制，防止栈溢出
libnativelib.so:0000007E929D3B90 MRS             X8, #3, c13, c0, #2
libnativelib.so:0000007E929D3B94 LDR             X8, [X8,#0x28]
libnativelib.so:0000007E929D3B98 STUR            X8, [X29,#var_8]



libnativelib.so:0000007E929D3CBC MRS             X8, #3, c13, c0, #2
libnativelib.so:0000007E929D3CC0 LDR             X8, [X8,#0x28]
libnativelib.so:0000007E929D3CC4 LDUR            X9, [X29,#var_8]
libnativelib.so:0000007E929D3CC8 SUBS            X8, X8, X9
libnativelib.so:0000007E929D3CCC B.NE            loc_7E929D3CE0

//https://cataloc.gitee.io/blog/2021/04/24/Android%E9%80%86%E5%90%91%E4%B8%AD%E7%9A%84Canary%E6%9C%BA%E5%88%B6/

### LDR
`LDR` 和 `LDUR` 都是ARM汇编中用于加载（Load）数据的指令，但它们有一些重要的区别：

1. LDR（Load Register）：
   - `LDR` 指令用于从内存中加载数据，并将其存储到通用寄存器中。
   - `LDR` 指令可以加载各种数据类型，包括字节、半字、字、双字以及浮点数等。
   - `LDR` 指令执行标准的内存访问，根据所加载数据的大小，可能会引发数据对齐异常（例如，尝试加载未对齐的数据）。
   - `LDR` 指令通常用于访问常规内存，如堆、栈或全局数据。

2. LDUR（Load Register Unprivileged）：
   - `LDUR` 指令是ARM64架构引入的一种指令，用于无权限加载数据，它通常在用户模式下使用。
   - `LDUR` 指令具有无权限访问内存的特性，因此即使在特权级别较低（如用户模式）也可以使用。
   - `LDUR` 指令在加载数据时会自动处理数据对齐，不会引发数据对齐异常，这是 `LDUR` 与 `LDR` 的一个重要区别。
   - `LDUR` 指令也可以加载各种数据类型，但它在无权限访问的情况下更有用。

总结来说，主要区别在于权限和数据对齐处理：

- `LDR` 是标准的加载指令，用于加载数据到通用寄存器，通常在特权级别较高的情况下使用。
- `LDUR` 是一种无权限加载指令，它可以在用户模式下使用，且具有自动数据对齐的功能，因此更适合无权限访问的情况。

选择使用哪个指令取决于特定的应用场景和访问内存的权限要求。

`LDRB` 是ARM汇编中的指令，用于从内存中加载一个字节（8位数据）并将其存储到通用寄存器中。`LDRB` 的名称表示 "Load Byte"，它执行以下操作：

1. 从指定的内存地址读取一个字节的数据。
2. 将读取的字节数据零扩展到32位，然后将其存储在目标通用寄存器中。

`LDRB` 指令的通用格式如下：

```assembly
LDRB Rd, [Rn, #offset]
```

其中：
- `Rd` 是目标通用寄存器，用于存储加载的字节数据。
- `Rn` 是基址寄存器，包含了内存地址的基址。
- `offset` 是一个偏移量，指定要加载的内存地址相对于基址寄存器 `Rn` 的偏移。

举个例子，如果要从内存地址 `R0 + 8` 处加载一个字节，然后将其存储到通用寄存器 `R1` 中，可以使用以下指令：

```assembly
LDRB R1, [R0, #8]
```

这条指令将加载地址为 `R0 + 8` 处的一个字节，将其零扩展为32位，然后将其存储在 `R1` 寄存器中。`LDRB` 通常用于加载单个字节的数据，例如字符或字节型的数据。

### MRS


# ARM64 学习：

## 一、基础知识
### 1.ARMv8特色：
    * (1) 超大的物理地址空间(Large Physical Address)，提供超过4GB物理内存的访问；
        (2) 64位宽的虚拟地址空间(64-bit Virtual Addresing);
        (3) 提供31个64位宽的通用寄存器，可以减少对栈的访问，从而提高性能；
        (4) 提供16KB和64KB的页面，有助于降低TLB的未命中率(miss rate);
        (5) 全新的异常处理模型，有助于降低操作系统和虚拟化的实现复杂度；
        (6)全新的加载-获取，存储-释放指令(Load-Acquire, Store-Release Instructions)。专门为C++11，C11以及Java内存模型设计；

    2 执行状态
        AArch64: 64位的执行状态：
        (1) 提供31个64位通用寄存器；
        (2) 提供64位的程序计数器寄存器PC、栈指针寄存器SP以及异常链接寄存器；
        (3) 提供A64指令集；
        (4)定义ARMv8异常模型，支持4个异常等级，EL0~EL3;
        (5)提供64位的内存模型；
        (6)定义一组处理器状态(PSTATE)用来保存PE的状态；
        AArch32: 32位的执行状态：
        (1) 提供13个632位通用寄存器, 32位的程序计数寄存器PC、栈指针寄存器SP、链接寄存器；
        (2) 提供A32,T32指令集；
        (3)定义ARMv7异常模型，基于PE模式并映射到ARMv8的异常模型中；
        (4)提供32位虚拟内存访问机制；
        (5)定义一组处理器状态(PSTATE)用来保存PE的状态；

    3 ARMv8包含的寄存器
        3.1 ARMv8包含31个通用寄存器
            AArch64运行状态支持31个通用寄存器X0~X30，AArch32状态支持16个32位通用寄存器；
            X0~X30：通用寄存器；
            SP: 栈指针寄存器；
            PC：程序计数寄存器；
            其中：x30寄存器（lr寄存器）
                x30寄存器是链接寄存器，用于保存函数的返回地址，当ret指令执行时，会寻找x30寄存器中的值，作为返回地址；
                x29寄存器是fp寄存器，用于保存函数的栈帧指针，指向函数的栈帧起始地址，即栈底地址；

        3.2 系统寄存器
            系统寄存器提供控制和状态，在AArch64状态下，很多系统寄存器根据不同的异常等级提供不同的变种寄存器：
            <reg_name>_ELx, x is 0,1,2, or 3
            比如SP_EL0表示在EL0下的栈指针寄存器；

        3.3 SIMD/FP寄存器：
            支持128bit寄存器

        3.4 特殊寄存器
            XZE

    4.数据类型
        Byte: 8bit
        Halfword: 16bit
        Word: 32bit
        Doubleworld: 64bit
        Quadword: 128bit

    5.异常模型
        Exception Levels确定了处理器当前运行的特权级别，类似ARMv7架构中的特权等级
        EL0: 用户特权，用于运行普通应用程序；
        EL1: 系统特权，通常用于运行操作系统；
        EL2: 运行虚拟化扩展的虚拟监控程序(Hypervisor);
        EL3: 运行安全世界中的安全监控器(Secure Monitor);
        异常级别	应用场景	安全世界
        EL0	APP	Secure OS
        EL1	Guest OS	sECURE OS
        EL2	Hypervisor	Secure Hypervisor
        EL3	Secure—>	–>monitor

    6.A64汇编指令介绍：
        1.A64指令集只能运行在aarch64环境中；
        2.所有的A64汇编指令都是32bit宽；
        3.A64支持全部大写或小写书写方式；
        4.寄存器命名,
        Name	size	Encoding	Descriptio
        Wn	32bits	0~30	General-purpose register0~30
        Xn	64bit	0~30	General-purpose register0~30
        WZR	32bits	31	Zero register
        XZR	64bit	31	Zero register
        WSP	32bits	31	Current stack pointer
        SP	64bit	31	Current stack pointer
    
    7.A64指令分类
        内存加载和存储指令：
        多字节内存加载和存储：
        算术和移位指令：
        位操作指令：
        条件操作：
        跳转指令：
        独占访存指令：
        内存屏障指令：
        异常处理指令：
        系统寄存器访问指令：

## 二、ARM 直接编译.s 文件

cd /Users/nuoen/Library/Android/sdk/ndk/25.2.9519653/toolchains/llvm/prebuilt/
目录更新：
/Users/nuoen/Library/Android/sdk/ndk/27.0.12077973/toolchains/llvm/prebuilt/darwin-x86_64/bin

直接编译.s 文件，/Users/nuoen/Documents/AndroidSecurity/fridaScript/arm64.s
android 8.1 :
```
./clang -target aarch64-linux-android26 -v  ~/Documents/AndroidSecurity/fridaScript/arm64.s -o ~/Documents/AndroidSecurity/fridaScript/arm64android --static -ffunction-sections -fdata-sections -Wl,--gc-sections 
```
android 13 :
```
./clang -target aarch64-linux-android31 -v  ~/Documents/AndroidSecurity/fridaScript/arm64.s -o ~/Documents/AndroidSecurity/fridaScript/arm64android --static
```
其中：
--static 主要解决 : ld: error: relocation R_AARCH64_ABS64 cannot be used against local symbol; recompile with -fPIC
-ffunction-sections -fdata-sections -Wl,--gc-sections 主要解决:
executable's TLS segment is underaligned: alignment is 8, needs to be at least 64 for ARM64 Bionic
https://github.com/termux/termux-packages/issues/8273


直接编译.cpp 文件 为可执行文件
./clang++  -target aarch64-linux-android32   ~/Documents/AndroidSecurity/fridaScript/cppLearn/cpptest.cpp  -o ~/Documents/AndroidSecurity/fridaScript/cppLearn/cpptest

编译.cpp 文件为 .s文件
./clang++  -target aarch64-linux-android32  -S  ~/Documents/AndroidSecurity/fridaScript/cppLearn/cpptest.cpp  -o ~/Documents/AndroidSecurity/fridaScript/cppLearn/cpptest.s
利用lldb远程调试
手机端：
```
./lldb-server p --server --listen unix-abstract:///data/local/tmp/debug.sock
```
调试端：
```
$ lldb-<version>
$ platform list  # 查看lldb可以连接的平台
$ platform select remote-android
$ platform status # 查看平台状态
$ platform connect unix-abstract-connect:///data/local/tmp/debug.sock
```
</details>

A64的存储和加载指令
ldr和str指令
ARMv8也是基于指令加载和存储的架构，即不能直接操作内存
```
LDR <reg_dst>,<addr> //把存储器地址的数据加载到目的寄存器中
STR <reg_src>,<addr> //把原寄存器的值，存储到内存中
```
ld指令寻址1：地址偏移模式
```
ldr Xd,[Xn,$offset]
```


std:string内存结构
