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

### ARM PIC模式下的符号地址获取
#### 汇编代码
```
.text:00001C28 loc_1C28                                ; CODE XREF: JNI_OnLoad+30↑j
.text:00001C28                 SUB     R5, SP, #8
.text:00001C2C                 MOV     SP, R5
.text:00001C30                 LDR     R0, =(_GLOBAL_OFFSET_TABLE_ - 0x1C48)
.text:00001C34                 LDR     R1, =(sub_16A4 - 0x5FBC)
.text:00001C38                 MOV     R3, #0
.text:00001C3C                 STR     R8, [R5]
.text:00001C40                 ADD     R0, PC, R0      ; _GLOBAL_OFFSET_TABLE_
.text:00001C44                 ADD     R2, R1, R0      ; sub_16A4
.text:00001C48                 ADD     R0, R9, R0      ; unk_6290
.text:00001C4C                 MOV     R1, #0
.text:00001C50                 LDR     R7, [R0,#(off_62B4 - 0x6290)]
.text:00001C54                 SUB     R0, R11, #-var_20
.text:00001C58                 BLX     R7
.text:00001C5C                 BL      sub_17F4
.text:00001C60                 LDR     R0, [R4]
.text:00001C64                 MOV     R6, #4
.text:00001C68                 MOV     R1, R5
.text:00001C6C                 ORR     R6, R6, #0x10000
.text:00001C70                 MOV     R2, R6
.text:00001C74                 LDR     R3, [R0,#0x18]
.text:00001C78                 MOV     R0, R4
.text:00001C7C                 BLX     R3
.text:00001C80                 CMP     R0, #0
.text:00001C84                 MOVNE   R6, #0xFFFFFFFF
.text:00001C88                 MOV     R0, R6
.text:00001C8C                 SUB     SP, R11, #0x18
.text:00001C90                 POP     {R4-R9,R11,PC}
```
PIC截取片段：
```
LDR R0, =(_GLOBAL_OFFSET_TABLE_ - K)   ; 取“GOT 相对 PC 的偏移”这个常量
ADD R0, PC, R0                         ; R0 ← 运行时的 GOT 基址
LDR R1, =(sub_16A4 - C)                ; R1 ← 链接时放进字面池的“相对偏移”
ADD R2, R1, R0                         ; R2 ← GOT基址 + 偏移 = sub_16A4 的真实地址
```

* 第一步：`LDR R0, =(_GLOBAL_OFFSET_TABLE_ - 0x1C48) + ADD R0, PC, R0`—— 这会把 `_GLOBAL_OFFSET_TABLE_ `的运行时基址（GOT base）计算到 R0。这是典型的加载 GOT 基址的 idiom。
* 第二步：`LDR R1, =(sub_16A4 - 0x5FBC) `—— 这不是直接把`sub_16A4 - 0x5FBC`当作最终地址；而是把一个编译/链接阶段生成的常量（通常是“符号相对于某个参考基址的偏移”）加载到 R1。
* 第三步：`ADD R2, R1, R0 `—— 把上面的偏移（R1）和运行时的 GOT/基址（R0）相加，得到 `sub_16A4` 的实际运行时地址（放到 R2）。

#### 解释：
**静态** 指的是目标相对于so的地址
**动态** 指的是so链接到程序后，运行时的地址
* **_GLOBAL_OFFSET_TABLE_** 静态的GOT地址
* **0x1C48**  静态PC的值
* **LDR R0, =(_GLOBAL_OFFSET_TABLE_ - 0x1C48)**  静态的GOT - 静态PC的值 => GOT相当于PC的偏移
* **ADD R0, PC, R0** GOT相对于PC的偏移 + 运行时PC值 =>  动态的GOT地址
* **0x5FBC** _GLOBAL_OFFSET_TABLE_  GOT表(全局偏移表)相对于so的地址
* **LDR R1, =(sub_16A4 - 0x5FBC)** 计算静态符号距离静态GOT偏移
* **ADD R2, R1, R0**  R1: 动态GOT地址， R0:sub_16A4 相对于GOT的偏移 

这里的R2 就是sub_16A4运行时的地址

#### 比喻：
把 sub_16A4 - 0x5FBC 想成书架上的“书的编号相对于书柜左边缘的位置”，GOT_base/module base 就是“书柜的左边缘在房间的真实坐标”。要拿到书的房间内真实坐标，你要把“书相对书柜的编号（偏移）”加上“书柜在房间里的坐标（基址）”。IDA 把“书的编号写成 book - 0x5FBC”来表示它是个相对量。

#### PS
```
.text:00001C48                 ADD     R0, R9, R0      ; unk_6290
.text:00001C50                 LDR     R7, [R0,#(off_62B4 - 0x6290)]
```
1C48这里，R0 = unk_6290相对于GOT的静态偏移 + GOT的动态地址
1C50这里，R7 = R0 一定是unk_6290的动态地址 + 偏移off_62B4相对于unk_6290的静态偏移 => off_62B4的动态地址
#### 名词：
1) 什么是 “link-time 常量（相对某个基址的偏移）”
	•	link-time 常量：由链接器在链接时计算并写入到代码/字面池/重定位记录中的一个数值。它经常不是“绝对地址”，而是一个表达式的结果，例如“符号相对某个锚点（基址）的偏移”。在 ELF 里，这由重定位条目（relocation）描述：运行时装载器会按“符号值 + 加数（addend）+（可选）PC/基址”的公式把它补全/应用到目标位置。 ￼
	•	在 ARM 上，LDR Rd, =expr 这种伪指令常把 32 位常量放进字面池（literal pool），再用 PC 相对的 LDR 取出来。这个 expr 可以是“符号 - 锚点”这样的相对量，IDA 就会把它渲染成你看到的样子。 ￼

直观理解：链接器先把“某函数相对于某基准的偏移”记到常量区；到运行时，再把“偏移 + （运行时算出来的）基址”相加，得到真实地址。

⸻

2) 什么是 GOT / _GLOBAL_OFFSET_TABLE_ 的“基址”
	•	GOT（Global Offset Table，全局偏移表）是共享库/PIE 用来在运行时保存“需要绝对地址的全局数据/函数地址”的一个表。动态链接器会把它填好或按需填好（PLT/GOT-PLT）。.got/.got.plt 常配合 PLT 处理外部符号调用。 ￼
	•	GOT 基址就是该表在当前模块被装载后的起始地址。PIC 代码会先用一小段 PC 相对序列把 GOT 基址算进某寄存器（你片段里是通过 LDR … =(_GLOBAL_OFFSET_TABLE_ - …) + ADD PC, … 这类序列得到的），然后再用“GOT 基址 + 偏移”去拿到真正的目标地址或到 GOT 中继续取绝对地址。 ￼
	•	在 ARM AAPCS/EABI 模型下，某些平台/编译器会把 R9 指定为 SB（static base） 来当做“位置无关数据的静态基址”，本质也是“先拿到一个基址，再加偏移”。（不同平台对 R9 的角色有差异。） ￼

⸻

3) 为什么要用 “link-time 偏移 + GOT 基址” 来算地址
	•	支持位置无关 / ASLR：共享库或 PIE 进程每次装载到的绝对地址可能不同，因此不能在指令里硬编码“绝对地址”。把“相对偏移”写进常量区/重定位里，运行时再与“当前装载得到的基址”相加，就能在任意基址下得到正确的目标地址。 ￼
	•	ELF 重定位模型：ELF 里普遍采用“S + A (+ P)”一类的重定位计算（S=符号值，A=加数/偏移，P=当前指令位置等），把链接时给出的加数（偏移）与运行时可知的符号/基址组合得到最终值。这正是你看到 “sub_16A4 - 0x5FBC + 运行时基址” 的来源。 ￼
	•	ARM 汇编生成习惯：编译器用 LDR Rd, =expr 伪指令+字面池存偏移，再通过 ADD 把它与 PC/GOT 基址相加，得到最终地址；IDA 会把字面池里的常量显示成 符号 - 某常量 的形式，容易让人误以为要“真的去减”，其实它只是被选作的锚点表达。 ￼

⸻


### Thumb 切到 ARM ,tunk/跳板
```
text:00086B08 ; =============== S U B R O U T I N E =======================================
.text:00086B08
.text:00086B08 ; Attributes: thunk
.text:00086B08
.text:00086B08 sub_86B08                               ; CODE XREF: sub_16238+A↑p
.text:00086B08                                         ; sub_19F4C+A↑p ...
.text:00086B08                 BX      PC
.text:00086B08 ; ---------------------------------------------------------------------------
.text:00086B0A                 ALIGN 4
.text:00086B0C                 CODE32
.text:00086B0C
.text:00086B0C loc_86B0C                               ; CODE XREF: sub_86B08↑j
.text:00086B0C                 LDR     R12, =(sub_11244 - 0x86B18)
.text:00086B10                 ADD     PC, R12, PC     ; sub_11244
.text:00086B10 ; ---------------------------------------------------------------------------
.text:00086B14 off_86B14       DCD sub_11244 - 0x86B18 ; DATA XREF: sub_86B08:loc_86B0C↑r
.text:00086B14 ; End of function sub_86B08
.text:00086B14
.text:00086B18                 CODE16
.text:00086B18 ; [00000010 BYTES: COLLAPSED FUNCTION j___umodsi3. PRESS CTRL-NUMPAD+ TO EXPAND]
.text:00086B28
.text:00086B28 ; =============== S U B R O U T I N E =======================================

// attributes: thunk
int __fastcall sub_86B08(int a1)
{
  return sub_11244(a1);
}
```
跳板解释：
```
.text:00086B08 sub_86B08          ; 被很多地方 BL 调用的“存根”
.text:00086B08    BX      PC      ; 在 Thumb 态执行。BX PC 会“切换指令集”，跳到当前 PC 指向处但把 T 位翻转
.text:00086B0A    ALIGN 4         ; 接下来按 4 字节对齐（ARM 指令对齐）
.text:00086B0C    CODE32          ; 这里开始用 ARM 指令集
.text:00086B0C loc_86B0C:
.text:00086B0C    LDR     R12, =(sub_11244 - 0x86B18)  ; 从字面池取“到 sub_11244 的相对偏移”
.text:00086B10    ADD     PC, R12, PC                  ; PC ← PC + R12  => 跳去 sub_11244
.text:00086B14 off_86B14 DCD sub_11244 - 0x86B18       ; 字面池，存的就是上面要加载的相对偏移
.text:00086B18    CODE16
; 这里 IDA 注释：j___umodsi3（被折叠的另一段 Thumb 代码）
```
#### 为什么需要这样绕一下
	•	指令集切换（ARM ↔ Thumb）：调用点（比如 BL sub_86B08）可能在 Thumb 态，但真正的目标函数 sub_11244 在 ARM 态（或距离/可达性问题）。直接跨态跳转需要符合 ABI 规则，linker 会插入这种 veneer 来保证安全切换。
	•	位置无关 / 远距跳转：即使同一态，有时因为距离超出分支立即数范围或出于 PIC/ASLR 的需要，linker 也会生成用字面池+ADD PC 的 veneer 来“拼”出实际目标地址。
	•	复用存根：同一个 veneer 可能被多个调用点引用（你也看到上面注释里：sub_16238+A、sub_19F4C+A 都引用了它）。


#### Veneer vs Thunk 对比
# Veneer vs Thunk 对比
# Veneer vs Thunk 对照表（完整版）

> 速记：**Veneer = 链接器的“跳板”**（修补指令集/距离/重定位），**Thunk = 编译器的“语义适配器”**（修补 this/调用约定/语言特性）。

| 维度 | **Veneer（贴面/跳板）** | **Thunk（桩/适配桩）** |
|---|---|---|
| **定义** | 由 **链接器**自动生成的小段中转代码，用于保证能“正确跳到目标地址”。 | 由 **编译器**自动生成的小段中转代码，用于“正确完成一次调用的语义”。 |
| **主要动机** | ① ARM↔Thumb **指令集切换**（interworking）<br>② **分支距离**超出 `B/BL` 可达范围（long branch）<br>③ **位置无关/PIC/ASLR** 下需要 **PC/GOT 相对**拼地址 | ① C++ **虚函数/多继承**导致的 **this 指针调整**（adjustor thunk）<br>② **调用约定/ABI** 差异（如 `cdecl`↔`stdcall`）<br>③ **闭包/lambda** 捕获桥接、延迟求值等语言特性 |
| **生成阶段/工具** | **Linker 阶段**（ld/gold/lld 等） | **Compiler 阶段**（clang/gcc/msvc 等） |
| **典型指令形态（ARM32）** | `LDR r12, =target - anchor` + `ADD pc, r12, pc`；<br>`BX <reg>` / `BLX <reg>` 完成 **切态**；有时引入 **literal pool**。 | `ADD reg, #imm` 调整 `this` / 参数，随后 `B/BL` 或 `JMP` 到真实实现；通常不切换指令集，仅做 **寄存器/栈** 轻微改动。 |
| **是否修改参数** | 一般 **不改参数**，只构造正确的控制流目标（可能切态）。 | 常 **调整参数/this**，或改变调用约定以匹配真实实现。 |
| **是否有栈帧** | 通常 **无**（极短，无 prologue/epilogue）。 | 可能有或无，但经常有 **轻量调整**（不一定建完整栈帧）。 |
| **位置/命名特征** | 常出现在 **代码段边界/对齐处**；可能被多个调用点共用；ID A 里常标注 “**Attributes: thunk**”（但其语义更接近 veneer）。 | 靠近类/方法定义或虚表附近；可能被标注为 thunk，并带有带修饰名的符号（C++ 名字整形）。 |
| **与符号/重定位** | 强依赖 **PC 相对**、**GOT/PLT**、**literal pool**；链接时/装载时完成地址拼接。 | 与 **语言语义** 与 **对象模型/ABI** 紧密关联，较少依赖 GOT。 |
| **平台分布** | ARM/Thumb、AArch64（长跳 veneer）、RISC-V（远距分支）、PowerPC 等常见。 | 几乎所有支持 OOP/多 ABI 的平台（x86/x64/ARM/ObjC/Swift…）。 |
| **如何识别（逆向）** | ① 极短、**只做取地址+跳转/切态**；② 包含 `BX/BLX` 或 `ADD pc, reg, pc`；③ 紧邻 **字面池**（`DCD target - anchor`）。 | ① 有 **this/参数** 调整痕迹；② 紧跟一个真正实现；③ 符号/交叉引用指向某个类成员实现。 |
| **调试/验证要点** | 单步看 **PC 变化** 与 **T 位（Thumb bit）**；观察 literal pool 值是否等于 `target - anchor`；确认跳转后是否落到目标函数入口。 | 单步看 **寄存器/栈** 的调整；确认最终 `jmp/call` 到真实实现；检查虚表项是否先落到该 thunk。 |

---

## 示例 A：Veneer（ARM32 Interworking & 远距跳转）

```asm
; Thumb 态入口（IDA 可能显示为 CODE16）
sub_86B08:
    BX      PC              ; 切到 ARM 态（interworking）
    ALIGN   4
    CODE32
loc_86B0C:
    LDR     r12, =(sub_11244 - 0x86B18)  ; 从字面池取“目标相对偏移”
    ADD     pc, r12, pc                  ; PC ← PC + r12 ⇒ 跳到 sub_11244
off_86B14 DCD sub_11244 - 0x86B18        ; 字面池 / 重定位项


