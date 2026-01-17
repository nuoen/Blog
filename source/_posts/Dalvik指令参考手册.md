# Dalvik指令参考手册

本文档为 Dalvik 虚拟机字节码指令集的完整参考手册。包括：

- 198条已覆盖指令的详细说明和十六进制解析
- dexdump 示例的逐字节分析
- 26条未覆盖指令的原因分析
- 完整的格式说明和编码规则
- 指令格式速查表和完整指令列表

## 目录

1. [指令格式说明](#指令格式说明)
2. [覆盖率统计](#覆盖率统计)
3. [字节序与格式说明](#字节序与格式说明)
4. [指令分类详解](#指令分类详解)
   - [NOP 指令](#1-nop-指令)
   - [数据移动指令](#2-数据移动指令)
   - [返回指令](#3-返回指令)
   - [常量加载指令](#4-常量加载指令)
   - [监视器指令](#5-监视器指令)
   - [类型检查指令](#6-类型检查指令)
   - [数组操作指令](#7-数组操作指令)
   - [实例字段操作](#8-实例字段操作)
   - [静态字段操作](#9-静态字段操作)
   - [方法调用指令](#10-方法调用指令)
   - [一元运算指令](#11-一元运算指令)
   - [二元运算指令](#12-二元运算指令)
   - [类型转换指令](#13-类型转换指令)
   - [比较指令](#14-比较指令)
   - [控制流指令](#15-控制流指令)
   - [异常处理指令](#16-异常处理指令)
   - [对象操作指令](#17-对象操作指令)
5. [未覆盖指令分析](#未覆盖指令分析)
6. [附录](#附录)

---

## 指令格式说明

### 格式代码规则

指令格式通常由 3 个字符组成:

1. **第一位数字**: 表示 16 位代码单元的数量
2. **第二位数字**: 表示最大寄存器数量
3. **最后一个字母**: 表示编码的额外数据类型

### 常见格式类型

| 格式 | 说明 | 语法示例 |
|------|------|----------|
| 10x | 简单操作码,无参数 | `nop` |
| 12x | 两个寄存器参数 | `move vA, vB` |
| 11n | 一个寄存器 + 4位立即数 | `const/4 vA, #+B` |
| 11x | 一个寄存器参数 | `move-result vAA` |
| 10t | 无寄存器 + 8位分支偏移 | `goto +AA` |
| 20t | 无寄存器 + 16位分支偏移 | `goto/16 +AAAA` |
| 20bc | 一个字节 + 常量池索引 | `throw-verification-error AA, kind@BBBB` |
| 22x | 一个8位寄存器 + 一个16位寄存器 | `move/from16 vAA, vBBBB` |
| 21t | 一个8位寄存器 + 16位分支偏移 | `if-eqz vAA, +BBBB` |
| 21s | 一个8位寄存器 + 16位有符号立即数 | `const/16 vAA, #+BBBB` |
| 21h | 一个8位寄存器 + 16位高位立即数 | `const/high16 vAA, #+BBBB0000` |
| 21c | 一个8位寄存器 + 常量池索引 | `const-string vAA, string@BBBB` |
| 23x | 三个8位寄存器 | `add-int vAA, vBB, vCC` |
| 22b | 两个8位寄存器 + 8位立即数 | `add-int/lit8 vAA, vBB, #+CC` |
| 22t | 两个4位寄存器 + 16位分支偏移 | `if-eq vA, vB, +CCCC` |
| 22s | 两个4位寄存器 + 16位立即数 | `add-int/lit16 vA, vB, #+CCCC` |
| 22c | 两个4位寄存器 + 常量池索引 | `iget vA, vB, field@CCCC` |
| 22cs | 两个4位寄存器 + 字段偏移 | *(已弃用)* |
| 30t | 无寄存器 + 32位分支偏移 | `goto/32 +AAAAAAAA` |
| 32x | 两个16位寄存器 | `move/16 vAAAA, vBBBB` |
| 31i | 一个8位寄存器 + 32位立即数 | `const vAA, #+BBBBBBBB` |
| 31t | 一个8位寄存器 + 32位分支偏移 | `fill-array-data vAA, +BBBBBBBB` |
| 31c | 一个8位寄存器 + 32位常量池索引 | `const-string/jumbo vAA, string@BBBBBBBB` |
| 35c | 方法调用格式,支持0-5个寄存器 | `invoke-kind {vC,vD,vE,vF,vG}, meth@BBBB` |
| 35ms | 方法调用格式 + 方法偏移 | *(已弃用)* |
| 35mi | 内联方法调用 | *(已弃用)* |
| 3rc | 方法调用范围格式 | `invoke-kind/range {vCCCC..vNNNN}, meth@BBBB` |
| 3rms | 方法调用范围 + 方法偏移 | *(已弃用)* |
| 3rmi | 内联方法调用范围 | *(已弃用)* |
| 45cc | 方法调用 + 方法原型 | `invoke-polymorphic {vC..vG}, meth@BBBB, proto@HHHH` |
| 4rcc | 方法调用范围 + 方法原型 | `invoke-polymorphic/range {vCCCC..vNNNN}, meth@BBBB, proto@HHHH` |
| 51l | 一个8位寄存器 + 64位立即数 | `const-wide vAA, #+BBBBBBBBBBBBBBBB` |

### 字母后缀含义

- **b**: 立即有符号字节
- **c**: 常量池索引 (字符串、类型、字段、方法)
- **h**: 立即有符号高位数值 (用于右移位)
- **i**: 立即有符号整数或 32 位浮点数
- **l**: 立即有符号长整型或 64 位双精度浮点数
- **m**: 方法偏移 *(已弃用)*
- **n**: 立即有符号半字节 (4 位)
- **s**: 立即有符号短整型
- **t**: 分支目标偏移
- **x**: 无额外数据

---

## 覆盖率统计

### 总体概览

- **官方指令总数**: 224 条
- **已覆盖指令数**: 198 条
- **覆盖率**: **88.4%**
- **未覆盖指令数**: 26 条
- **文档版本**: 2025-01-01
- **数据来源**: Android 官方文档 + HelloWorld.java 实测

### 分类统计

| 类别 | 指令数量 | 已覆盖 | 覆盖率 |
|------|---------|--------|--------|
| 数据移动 | 13 | 10 | 76.9% |
| 返回指令 | 4 | 4 | 100% |
| 常量加载 | 12 | 10 | 83.3% |
| 监视器 | 2 | 2 | 100% |
| 类型检查 | 2 | 2 | 100% |
| 数组操作 | 16 | 16 | 100% |
| 实例字段 | 14 | 14 | 100% |
| 静态字段 | 14 | 14 | 100% |
| 方法调用 | 15 | 11 | 73.3% |
| 一元运算 | 6 | 6 | 100% |
| 二元运算 | 96 | 89 | 92.7% |
| 类型转换 | 15 | 15 | 100% |
| 比较指令 | 5 | 5 | 100% |
| 控制流 | 18 | 15 | 83.3% |
| 异常处理 | 2 | 2 | 100% |
| 对象操作 | 1 | 1 | 100% |
| **总计** | **224** | **198** | **88.4%** |

---

## 字节序与格式说明

### 字节序 (Little-Endian)

**DEX 文件使用小端序（Little-Endian）**：
- 多字节数据的最低有效字节存储在最低地址
- 例如：`0x1234` 在内存中存储为 `34 12`

**示例**：
```
十六进制: 0x12345678
小端序存储: 78 56 34 12

16 位数值 0x1234:
- 大端序: 12 34
- 小端序: 34 12 ✓ (DEX 使用)

32 位数值 0x12345678:
- 大端序: 12 34 56 78
- 小端序: 78 56 34 12 ✓

64 位数值 0x123456789ABCDEF0:
- 大端序: 12 34 56 78 9A BC DE F0
- 小端序: F0 DE BC 9A 78 56 34 12 ✓
```

### 指令格式速查表

| 格式 | 字节布局 | 说明 | 示例指令 |
|------|---------|------|----------|
| **10x** | `00 op` | 无操作数 | nop, return-void |
| **11x** | `op AA` | 单个 8 位寄存器 | move-result vAA, throw vAA |
| **11n** | `op AB` | 寄存器 + 4 位立即数 | const/4 vA, #+B |
| **12x** | `op AB` | 两个 4 位寄存器 | move vA, vB |
| **21t** | `op AA BBBB` | 寄存器 + 16 位分支偏移 | if-eqz vAA, +BBBB |
| **21s** | `op AA BBBB` | 寄存器 + 16 位有符号立即数 | const/16 vAA, #+BBBB |
| **21c** | `op AA BBBB` | 寄存器 + 16 位常量池索引 | const-string vAA, string@BBBB |
| **22x** | `op AA BBBB` | 8 位寄存器 + 16 位寄存器 | move/from16 vAA, vBBBB |
| **22t** | `op AB CCCC` | 两个 4 位寄存器 + 16 位分支偏移 | if-eq vA, vB, +CCCC |
| **22s** | `op AB CCCC` | 两个 4 位寄存器 + 16 位立即数 | add-int/lit16 vA, vB, #+CCCC |
| **22b** | `op AA BB CC` | 两个 8 位寄存器 + 8 位立即数 | add-int/lit8 vAA, vBB, #+CC |
| **22c** | `op AB CCCC` | 两个 4 位寄存器 + 16 位常量池索引 | iget vA, vB, field@CCCC |
| **23x** | `op AA BB CC` | 三个 8 位寄存器 | add-int vAA, vBB, vCC |
| **31i** | `op AA BBBBBBBB` | 寄存器 + 32 位立即数 | const vAA, #+BBBBBBBB |
| **31t** | `op AA BBBBBBBB` | 寄存器 + 32 位分支偏移 | packed-switch vAA, +BBBBBBBB |
| **35c** | `op AG BBBB FEDC` | 方法调用（≤5 参数） | invoke-virtual {vC..vG}, method@BBBB |
| **3rc** | `op AA BBBB CCCC` | 范围方法调用 | invoke-virtual/range {vCCCC..vNNNN}, method@BBBB |
| **51l** | `op AA BBBBBBBB BBBBBBBB` | 寄存器 + 64 位立即数 | const-wide vAA, #+BBBBBBBBBBBBBBBB |

**字段说明**：

- `op` = 操作码（8 位）
- `AA`, `BB`, `CC` = 8 位字段
- `A`, `B`, `C`, `D`, `E`, `F`, `G` = 4 位字段
- `BBBB`, `CCCC` = 16 位字段
- `BBBBBBBB` = 32 位字段

---

## 指令分类详解

## 1. NOP 指令

### 指令列表

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `nop` | 0x00 | 10x | 空操作，用于字节对齐或占位 |

### 十六进制解析

```asm
字节码: 00 00
│      │  └─ 高字节填充（必须为 0）
│      └──── 操作码 0x00 (nop)

总长度: 2 字节（1 个 code unit）
```

### Java 示例

```java
// NOP 指令通常由编译器自动插入用于对齐
switch (x) {
    case 1: break;  // 编译器可能在此插入 nop
    case 2: break;
}
```

---

## 2. 数据移动指令

### 2.1 寄存器间移动

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `move vA, vB` | 0x01 | 12x | 移动 32 位值（int/float） |
| `move/from16 vAA, vBBBB` | 0x02 | 22x | 从 16 位源寄存器移动 |
| `move/16 vAAAA, vBBBB` | 0x03 | 32x | 32 位寄存器移动 ❌ |
| `move-wide vA, vB` | 0x04 | 12x | 移动 64 位值（long/double） |
| `move-wide/from16 vAA, vBBBB` | 0x05 | 22x | 从 16 位源移动 64 位值 |
| `move-wide/16 vAAAA, vBBBB` | 0x06 | 32x | 32 位寄存器移动 64 位值 ❌ |
| `move-object vA, vB` | 0x07 | 12x | 移动对象引用 |
| `move-object/from16 vAA, vBBBB` | 0x08 | 22x | 从 16 位源移动对象引用 |
| `move-object/16 vAAAA, vBBBB` | 0x09 | 32x | 32 位寄存器移动对象引用 ❌ |

### 2.2 结果移动

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `move-result vAA` | 0x0a | 11x | 移动方法调用返回的 32 位结果 |
| `move-result-wide vAA` | 0x0b | 11x | 移动方法调用返回的 64 位结果 |
| `move-result-object vAA` | 0x0c | 11x | 移动方法调用返回的对象引用 |
| `move-exception vAA` | 0x0d | 11x | 保存捕获的异常对象 |

### 十六进制解析示例

#### move vA, vB (0x01)

**格式**: 12x
**字节布局**: `01 BA`

```asm
字节码: 01 10
        ││ └┴─ 寄存器编号 (高4位=vB=v1, 低4位=vA=v0)
        │└──── 操作码 0x01 (move)

解析:
- 操作码: 0x01
- vA (目标): 0 (低 4 位)
- vB (源): 1 (高 4 位)
- 指令: move v0, v1

作用: v0 = v1 (复制 32 位值)
总长度: 2 字节
```

#### move-result vAA (0x0a)

```asm
字节码: 0a 00
        │  └──── 目标寄存器 v0
        └─────── 操作码 0x0a

解析:
- move-result v0
- 作用: 将上一个方法调用返回的 32 位结果移到 v0
```

#### move-exception vAA (0x0d)

```asm
字节码: 0d 01
        │  └──── 目标寄存器 v1
        └─────── 操作码 0x0d

解析:
- move-exception v1
- 作用: 将捕获的异常对象保存到 v1
- 说明: 这是 catch 块的第一条指令
```

### Java 示例

```java
// move, move-wide, move-object
int a = 10;
int b = a;              // move v1, v0

long x = 123456789L;
long y = x;             // move-wide v1, v0

String str1 = "test";
String str2 = str1;     // move-object v1, v0

// move-result
int result = returnInt();  // invoke-static {}, returnInt
                           // move-result v0

// move-exception
try {
    // ...
} catch (Exception e) {    // move-exception v1
    System.out.println(e);
}
```

---

## 3. 返回指令

### 指令列表

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `return-void` | 0x0e | 10x | 从 void 方法返回 |
| `return vAA` | 0x0f | 11x | 返回 32 位值 |
| `return-wide vAA` | 0x10 | 11x | 返回 64 位值 |
| `return-object vAA` | 0x11 | 11x | 返回对象引用 |

### 十六进制解析

#### return-void (0x0e)

```asm
字节码: 0e 00
        │  └──── 填充字节（必须为 0）
        └─────── 操作码 0x0e

解析:
- return-void
- 作用: 从 void 方法返回
- 无返回值
总长度: 2 字节
```

#### return vAA (0x0f)

```asm
字节码: 0f 00
        │  └──── 返回值寄存器 v0
        └─────── 操作码 0x0f

解析:
- return v0
- 作用: 返回 v0 中的 32 位值（int 或 float）
```

#### return-wide vAA (0x10)

```asm
字节码: 10 00
        │  └──── 返回值寄存器 v0
        └─────── 操作码 0x10

解析:
- return-wide v0
- 作用: 返回 v0-v1 中的 64 位值（long 或 double）
```

### Java 示例

```java
void foo() {
    return;             // return-void
}

int getInt() {
    return 42;          // const/16 v0, #42
                        // return v0
}

long getLong() {
    return 1234567890L; // const-wide v0, #1234567890
                        // return-wide v0
}

String getString() {
    return "test";      // const-string v0, "test"
                        // return-object v0
}
```

---

## 4. 常量加载指令

### 4.1 整数常量

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `const/4 vA, #+B` | 0x12 | 11n | 加载 4 位立即数（-8 到 7） |
| `const/16 vAA, #+BBBB` | 0x13 | 21s | 加载 16 位立即数 |
| `const vAA, #+BBBBBBBB` | 0x14 | 31i | 加载 32 位立即数 |
| `const/high16 vAA, #+BBBB0000` | 0x15 | 21h | 加载高 16 位（低 16 位为 0） |

### 4.2 长整数常量

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `const-wide/16 vAA, #+BBBB` | 0x16 | 21s | 加载 16 位 long 值 |
| `const-wide/32 vAA, #+BBBBBBBB` | 0x17 | 31i | 加载 32 位 long 值 |
| `const-wide vAA, #+BBBBBBBBBBBBBBBB` | 0x18 | 51l | 加载 64 位 long 值 |
| `const-wide/high16 vAA, #+BBBB000000000000` | 0x19 | 21h | 加载高 16 位 long |

### 4.3 字符串和类常量

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `const-string vAA, string@BBBB` | 0x1a | 21c | 加载字符串引用 |
| `const-string/jumbo vAA, string@BBBBBBBB` | 0x1b | 31c | 加载字符串引用（大索引） ❌ |
| `const-class vAA, type@BBBB` | 0x1c | 21c | 加载 Class 对象引用 |

### 十六进制解析示例

#### const/4 vA, #+B (0x12)

**格式**: 11n
**字节布局**: `12 BA` (B=立即数, A=寄存器)

```asm
示例 1: const/4 v0, #int 0
字节码: 12 00
        │  └┴─ BA 字段 (B=0, A=0)
        └──── 操作码 0x12

解析:
- 12 = 操作码
- 低 4 位 (A) = 0 → 目标寄存器 v0
- 高 4 位 (B) = 0 → 立即数 0
- 指令: const/4 v0, #int 0

总长度: 2 字节
立即数范围: -8 到 7 (4 位有符号)
```

```asm
示例 2: const/4 v1, #int 4
字节码: 12 41
        │  └┴─ BA 字段
        └──── 操作码 0x12

位字段分解:
- 0x41 = 0100 0001 (二进制)
         ││││ └┴┴┴─ A (低4位) = 0001 = 1 (v1)
         └┴┴┴────── B (高4位) = 0100 = 4

解析:
- vA = v1
- #+B = 4
- 指令: const/4 v1, #int 4
```

```asm
示例 3: const/4 v0, #int -1
字节码: 12 f0
        │  └┴─ BA 字段
        └──── 操作码

解析:
- 0xf0 = 1111 0000 (二进制)
- A = 0 (v0)
- B = 0xf = -1 (4 位有符号，补码)
- 指令: const/4 v0, #int -1
```

#### const/16 vAA, #+BBBB (0x13)

**格式**: 21s
**字节布局**: `13 AA BBBB` (小端序)

```asm
示例 1: const/16 v0, #int 100
字节码: 13 00 64 00
        │  │  └─┴─ 立即数 0x0064 = 100 (小端序)
        │  └─────── 目标寄存器 v0
        └────────── 操作码 0x13

字节分解:
- 字节 0: 0x13 (操作码)
- 字节 1: 0x00 (vAA = v0)
- 字节 2-3: 0x0064 (BBBB, 小端序)
  - 0x64 = 100 (十进制)

解析:
- vAA = v0
- #+BBBB = 100
- 指令: const/16 v0, #int 100

总长度: 4 字节
立即数范围: -32768 到 32767 (16 位有符号)
```

```asm
示例 2: const/16 v0, #int -128
字节码: 13 00 80 ff
        │  │  └─┴─ 立即数 0xff80 (小端序)
        │  └─────── v0
        └────────── 操作码

小端序解析:
- 存储字节: 80 ff
- 实际值: 0xff80 = -128 (16 位有符号补码)

解析:
- const/16 v0, #int -128
```

#### const vAA, #+BBBBBBBB (0x14)

**格式**: 31i
**字节布局**: `14 AA BBBBBBBB` (小端序)

```asm
示例: const v0, #float 1.414
字节码: 14 00 f4 fd b4 3f
        │  │  └─┴─┴─┴─ 32 位立即数 (小端序)
        │  └─────────── v0
        └────────────── 操作码 0x14

IEEE 754 浮点解析:
- 存储字节: f4 fd b4 3f
- 实际值: 0x3fb4fdf4 (小端序反转)
- 二进制: 0011 1111 1011 0100 1111 1101 1111 0100
  - 符号位: 0 (正数)
  - 指数: 01111111 = 127 - 127 = 0
  - 尾数: 01101001111110111110100
- 十进制: ≈ 1.414

解析:
- const v0, #float 1.414 // #3fb4fdf4

总长度: 6 字节
```

#### const-wide vAA, #+BBBBBBBBBBBBBBBB (0x18)

**格式**: 51l
**字节布局**: `18 AA BBBBBBBB BBBBBBBB` (小端序)

```asm
示例: const-wide v0, #double 1.73205
字节码: 18 00 a9 4c 58 e8 7a b6 fb 3f
        │  │  └─┴─┴─┴─┴─┴─┴─┴─ 64 位立即数
        │  └────────────────── v0
        └───────────────────── 操作码 0x18

小端序字节:
- 存储: a9 4c 58 e8 7a b6 fb 3f
- 实际值: 0x3ffbb67ae8584ca9 (反转)

IEEE 754 双精度解析:
- 0x3ffbb67ae8584ca9 = 1.73205 (double)
- 这是 √3 的近似值

解析:
- const-wide v0, #double 1.73205 // #3ffbb67ae8584ca9
- v0-v1 寄存器对存储 64 位值

总长度: 10 字节
```

#### const-string vAA, string@BBBB (0x1a)

**格式**: 21c
**字节布局**: `1a AA BBBB`

```asm
示例: const-string v0, "Instance String"
字节码: 1a 00 14 00
        │  │  └─┴─ 字符串索引 0x0014 (小端序)
        │  └─────── 目标寄存器 v0
        └────────── 操作码 0x1a

解析:
- vAA = v0
- string@BBBB = string@0x0014
- 指令: const-string v0, "Instance String" // string@0014

作用: 从字符串池的索引 0x14 处加载字符串引用到 v0
总长度: 4 字节
```

### Java 示例

```java
// const/4 - 小整数 (-8 到 7)
int tiny1 = 0;          // const/4 v0, #0
int tiny2 = 7;          // const/4 v0, #7
int tiny3 = -8;         // const/4 v0, #-8

// const/16 - 16 位整数
int small = 1000;       // const/16 v0, #1000
int negative = -5000;   // const/16 v0, #-5000

// const - 32 位整数
int large = 0x12345678; // const v0, #0x12345678

// const-wide - 64 位整数
long hugeLong = 9223372036854775807L; // const-wide v0, #9223372036854775807

// const-string - 字符串
String str = "Hello";   // const-string v0, "Hello"

// const-class - 类字面量
Class<?> clazz = String.class; // const-class v0, Ljava/lang/String;
```

---

## 5. 监视器指令

### 指令列表

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `monitor-enter vAA` | 0x1d | 11x | 获取对象监视器锁 |
| `monitor-exit vAA` | 0x1e | 11x | 释放对象监视器锁 |

### 十六进制解析

#### monitor-enter vAA (0x1d)

```asm
字节码: 1d 01
        │  └──── 寄存器 v1 (锁对象)
        └─────── 操作码 0x1d

解析:
- monitor-enter v1
- 作用: 获取 v1 对象的监视器锁
- 对应 Java: synchronized(v1) { 开始
总长度: 2 字节
```

#### monitor-exit vAA (0x1e)

```asm
字节码: 1e 01
        │  └──── 寄存器 v1
        └─────── 操作码 0x1e

解析:
- monitor-exit v1
- 作用: 释放 v1 对象的监视器锁
- 对应 Java: } synchronized 结束
总长度: 2 字节
```

### Java 示例

```java
Object lock = new Object();

synchronized (lock) {      // monitor-enter v1
    int x = 100;
    x += 50;
}                          // monitor-exit v1
```

---

## 6. 类型检查指令

### 指令列表

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `check-cast vAA, type@BBBB` | 0x1f | 21c | 检查对象是否为指定类型，失败抛异常 |
| `instance-of vA, vB, type@CCCC` | 0x20 | 22c | 检查对象是否为指定类型，返回布尔值 |

### 十六进制解析

#### check-cast vAA, type@BBBB (0x1f)

```asm
字节码: 1f 01 0f 00
        │  │  └─┴─ 类型索引 0x000f (String)
        │  └─────── 寄存器 v1
        └────────── 操作码 0x1f

解析:
- check-cast v1, Ljava/lang/String; // type@000f
- 作用: 检查 v1 是否为 String 类型，失败抛 ClassCastException
总长度: 4 字节
```

#### instance-of vA, vB, type@CCCC (0x20)

**格式**: 22c
**字节布局**: `20 AB CCCC`

```asm
字节码: 20 01 0f 00
        │  │  └─┴─ 类型索引 0x000f
        │  └─────── 寄存器 (A=v1结果, B=v0对象)
        └────────── 操作码 0x20

位字段:
- 0x01 = 0000 0001
  - 低4位 (B) = 1 → v0 (被检查对象)
  - 高4位 (A) = 0 → v1 (结果)

解析:
- instance-of v1, v0, Ljava/lang/String; // type@000f
- 作用: v1 = (v0 instanceof String) ? 1 : 0
总长度: 4 字节
```

### Java 示例

```java
Object obj = "test";

if (obj instanceof String) {    // instance-of v0, v1, Ljava/lang/String;
    String str = (String) obj;  // check-cast v1, Ljava/lang/String;
}
```

---

## 7. 数组操作指令

### 7.1 数组创建和长度

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `array-length vA, vB` | 0x21 | 12x | 获取数组长度到 vA |
| `new-instance vAA, type@BBBB` | 0x22 | 21c | 创建新对象实例 |
| `new-array vA, vB, type@CCCC` | 0x23 | 22c | 创建新数组，长度为 vB |
| `filled-new-array {vC, vD, ...}, type@BBBB` | 0x24 | 35c | 创建并填充数组（最多 5 元素） |
| `filled-new-array/range {vCCCC .. vNNNN}, type@BBBB` | 0x25 | 3rc | 范围创建并填充数组 |
| `fill-array-data vAA, +BBBBBBBB` | 0x26 | 31t | 使用数据表填充数组 |

### 7.2 数组读取（aget）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `aget vAA, vBB, vCC` | 0x44 | 23x | 读取 int/float 数组元素 |
| `aget-wide vAA, vBB, vCC` | 0x45 | 23x | 读取 long/double 数组元素 |
| `aget-object vAA, vBB, vCC` | 0x46 | 23x | 读取对象数组元素 |
| `aget-boolean vAA, vBB, vCC` | 0x47 | 23x | 读取 boolean 数组元素 |
| `aget-byte vAA, vBB, vCC` | 0x48 | 23x | 读取 byte 数组元素 |
| `aget-char vAA, vBB, vCC` | 0x49 | 23x | 读取 char 数组元素 |
| `aget-short vAA, vBB, vCC` | 0x4a | 23x | 读取 short 数组元素 |

### 7.3 数组写入（aput）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `aput vAA, vBB, vCC` | 0x4b | 23x | 写入 int/float 数组元素 |
| `aput-wide vAA, vBB, vCC` | 0x4c | 23x | 写入 long/double 数组元素 |
| `aput-object vAA, vBB, vCC` | 0x4d | 23x | 写入对象数组元素 |
| `aput-boolean vAA, vBB, vCC` | 0x4e | 23x | 写入 boolean 数组元素 |
| `aput-byte vAA, vBB, vCC` | 0x4f | 23x | 写入 byte 数组元素 |
| `aput-char vAA, vBB, vCC` | 0x50 | 23x | 写入 char 数组元素 |
| `aput-short vAA, vBB, vCC` | 0x51 | 23x | 写入 short 数组元素 |

### 十六进制解析示例

#### new-array vA, vB, type@CCCC (0x23)

**格式**: 22c
**字节布局**: `23 AB CCCC`

```asm
字节码: 23 01 12 00
        │  │  └─┴─ 类型索引 0x0012 ([I - int数组)
        │  └─────── 寄存器 (A=v1数组, B=v0长度)
        └────────── 操作码 0x23

解析:
- vA = v1 (数组引用)
- vB = v0 (数组长度)
- type@CCCC = [I (int[])
- 指令: new-array v1, v0, [I // type@0012

对应 Java: int[] arr = new int[v0];
总长度: 4 字节
```

#### aget vAA, vBB, vCC (0x44)

**格式**: 23x

```asm
字节码: 44 02 00 01
        │  │  │  └──── vCC = v1 (索引)
        │  │  └─────── vBB = v0 (数组)
        │  └────────── vAA = v2 (结果)
        └───────────── 操作码 0x44

解析:
- aget v2, v0, v1
- v2 = v0[v1] (读取 int 数组元素)
总长度: 4 字节
```

#### aput vAA, vBB, vCC (0x4b)

**格式**: 23x

```asm
字节码: 4b 02 00 01
        │  │  │  └──── vCC = v1 (索引)
        │  │  └─────── vBB = v0 (数组)
        │  └────────── vAA = v2 (值)
        └───────────── 操作码 0x4b

解析:
- aput v2, v0, v1
- v0[v1] = v2 (写入 int 数组元素)
总长度: 4 字节
```

### Java 示例

```java
// 创建数组
int[] arr = new int[10];        // new-array v0, v1, [I
int len = arr.length;           // array-length v2, v0

// 写入数组
arr[0] = 100;                   // const/16 v3, #100
                                // aput v3, v0, v4

// 读取数组
int val = arr[0];               // aget v5, v0, v4
```

---

## 8. 实例字段操作

### 8.1 读取实例字段（iget）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `iget vA, vB, field@CCCC` | 0x52 | 22c | 读取 int/float 实例字段 |
| `iget-wide vA, vB, field@CCCC` | 0x53 | 22c | 读取 long/double 实例字段 |
| `iget-object vA, vB, field@CCCC` | 0x54 | 22c | 读取对象引用实例字段 |
| `iget-boolean vA, vB, field@CCCC` | 0x55 | 22c | 读取 boolean 实例字段 |
| `iget-byte vA, vB, field@CCCC` | 0x56 | 22c | 读取 byte 实例字段 |
| `iget-char vA, vB, field@CCCC` | 0x57 | 22c | 读取 char 实例字段 |
| `iget-short vA, vB, field@CCCC` | 0x58 | 22c | 读取 short 实例字段 |

### 8.2 写入实例字段（iput）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `iput vA, vB, field@CCCC` | 0x59 | 22c | 写入 int/float 实例字段 |
| `iput-wide vA, vB, field@CCCC` | 0x5a | 22c | 写入 long/double 实例字段 |
| `iput-object vA, vB, field@CCCC` | 0x5b | 22c | 写入对象引用实例字段 |
| `iput-boolean vA, vB, field@CCCC` | 0x5c | 22c | 写入 boolean 实例字段 |
| `iput-byte vA, vB, field@CCCC` | 0x5d | 22c | 写入 byte 实例字段 |
| `iput-char vA, vB, field@CCCC` | 0x5e | 22c | 写入 char 实例字段 |
| `iput-short vA, vB, field@CCCC` | 0x5f | 22c | 写入 short 实例字段 |

### 十六进制解析示例

#### iget vA, vB, field@CCCC (0x52)

**格式**: 22c
**字节布局**: `52 AB CCCC`

```asm
字节码: 52 01 06 00
        │  │  └─┴─ 字段索引 0x0006 (instanceInt)
        │  └─────── 寄存器 (A=v1值, B=v0对象)
        └────────── 操作码 0x52

位字段:
- AB = 0x01
  - B (高4位) = 0 → v0 (对象)
  - A (低4位) = 1 → v1 (结果)

解析:
- iget v1, v0, LHelloWorld;.instanceInt:I // field@0006
- v1 = v0.instanceInt
总长度: 4 字节
```

#### iput vA, vB, field@CCCC (0x59)

**格式**: 22c

```asm
字节码: 59 20 06 00
        │  │  └─┴─ 字段索引 0x0006 (instanceInt)
        │  └─────── AB (A=v0值, B=v2对象)
        └────────── 操作码 0x59

位字段:
- AB = 0x20 = 0010 0000
  - A (低4位) = 0 → v0 (值)
  - B (高4位) = 2 → v2 (对象)

解析:
- iput v0, v2, LHelloWorld;.instanceInt:I // field@0006
- v2.instanceInt = v0
总长度: 4 字节
```

### Java 示例

```java
class Foo {
    private int value;

    void setValue(int v) {
        this.value = v;         // iput v1, v0, LFoo;.value:I
    }

    int getValue() {
        return this.value;      // iget v0, v1, LFoo;.value:I
    }
}
```

---

## 9. 静态字段操作

### 9.1 读取静态字段（sget）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `sget vAA, field@BBBB` | 0x60 | 21c | 读取 int/float 静态字段 |
| `sget-wide vAA, field@BBBB` | 0x61 | 21c | 读取 long/double 静态字段 |
| `sget-object vAA, field@BBBB` | 0x62 | 21c | 读取对象引用静态字段 |
| `sget-boolean vAA, field@BBBB` | 0x63 | 21c | 读取 boolean 静态字段 |
| `sget-byte vAA, field@BBBB` | 0x64 | 21c | 读取 byte 静态字段 |
| `sget-char vAA, field@BBBB` | 0x65 | 21c | 读取 char 静态字段 |
| `sget-short vAA, field@BBBB` | 0x66 | 21c | 读取 short 静态字段 |

### 9.2 写入静态字段（sput）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `sput vAA, field@BBBB` | 0x67 | 21c | 写入 int/float 静态字段 |
| `sput-wide vAA, field@BBBB` | 0x68 | 21c | 写入 long/double 静态字段 |
| `sput-object vAA, field@BBBB` | 0x69 | 21c | 写入对象引用静态字段 |
| `sput-boolean vAA, field@BBBB` | 0x6a | 21c | 写入 boolean 静态字段 |
| `sput-byte vAA, field@BBBB` | 0x6b | 21c | 写入 byte 静态字段 |
| `sput-char vAA, field@BBBB` | 0x6c | 21c | 写入 char 静态字段 |
| `sput-short vAA, field@BBBB` | 0x6d | 21c | 写入 short 静态字段 |

### 十六进制解析示例

#### sget vAA, field@BBBB (0x60)

**格式**: 21c

```asm
字节码: 60 01 10 00
        │  │  └─┴─ 字段索引 0x0010 (staticInt)
        │  └─────── 目标寄存器 v1
        └────────── 操作码 0x60

解析:
- sget v1, LHelloWorld;.staticInt:I // field@0010
- v1 = HelloWorld.staticInt (读取静态字段)
总长度: 4 字节
```

#### sget-object vAA, field@BBBB (0x62)

```asm
字节码: 62 02 17 00
        │  │  └─┴─ 字段索引 0x0017 (System.out)
        │  └─────── 目标寄存器 v2
        └────────── 操作码 0x62

解析:
- sget-object v2, Ljava/lang/System;.out:Ljava/io/PrintStream; // field@0017
- v2 = System.out
```

### Java 示例

```java
class Foo {
    private static int count;

    static void increment() {
        count++;                // sget v0, LFoo;.count:I
                                // add-int/lit8 v0, v0, #1
                                // sput v0, LFoo;.count:I
    }
}
```

---

## 10. 方法调用指令

### 10.1 普通方法调用（最多 5 个参数）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `invoke-virtual {vC, vD, ...}, method@BBBB` | 0x6e | 35c | 调用虚方法（动态绑定） |
| `invoke-super {vC, vD, ...}, method@BBBB` | 0x6f | 35c | 调用父类方法 |
| `invoke-direct {vC, vD, ...}, method@BBBB` | 0x70 | 35c | 调用直接方法（构造函数、私有方法） |
| `invoke-static {vC, vD, ...}, method@BBBB` | 0x71 | 35c | 调用静态方法 |
| `invoke-interface {vC, vD, ...}, method@BBBB` | 0x72 | 35c | 调用接口方法 |

### 10.2 范围方法调用（超过 5 个参数）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `invoke-virtual/range {vCCCC .. vNNNN}, method@BBBB` | 0x74 | 3rc | 范围调用虚方法 |
| `invoke-super/range {vCCCC .. vNNNN}, method@BBBB` | 0x75 | 3rc | 范围调用父类方法 |
| `invoke-direct/range {vCCCC .. vNNNN}, method@BBBB` | 0x76 | 3rc | 范围调用直接方法 |
| `invoke-static/range {vCCCC .. vNNNN}, method@BBBB` | 0x77 | 3rc | 范围调用静态方法 |
| `invoke-interface/range {vCCCC .. vNNNN}, method@BBBB` | 0x78 | 3rc | 范围调用接口方法 |

### 10.3 动态方法调用（Java 8+）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `invoke-custom {vC, vD, ...}, call_site@BBBB` | 0xfc | 35c | 调用动态方法（invokedynamic） |
| `invoke-custom/range {vCCCC .. vNNNN}, call_site@BBBB` | 0xfd | 3rc | 范围调用动态方法 ❌ |

### 10.4 多态方法调用（Android 9+）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `invoke-polymorphic {vC, vD, ...}, method@BBBB, proto@HHHH` | 0xfa | 45cc | 调用多态方法句柄 ❌ |
| `invoke-polymorphic/range {vCCCC .. vNNNN}, method@BBBB, proto@HHHH` | 0xfb | 4rcc | 范围调用多态方法句柄 ❌ |

### 十六进制解析示例

#### invoke-direct {vC..vG}, method@BBBB (0x70)

**格式**: 35c
**字节布局**: `70 AG BBBB FEDC`

```
示例: invoke-direct {v2}, BaseClass.<init>:()V
字节码: 70 10 00 00 02 00
        │  │  └─┴─ 方法索引 0x0000 (BaseClass.<init>)
        │  └─────── AG (A=1个参数, G=v0)
        └────────── 操作码 0x70
                   └─┴─ 参数寄存器 FEDC

位字段详解:
- AG = 0x10
  - A (高4位) = 1 → 1个参数
  - G (低4位) = 0 → 第一个参数 v0
- 但实际 FEDC = 0x0002
  - C = 0x2 → v2 (实际参数)

解析:
- invoke-direct {v2}, LBaseClass;.<init>:()V // method@0000
- 调用父类构造函数，v2 是 this 指针

总长度: 6 字节
```

#### invoke-virtual {vC..vG}, method@BBBB (0x6e)

**格式**: 35c

```
示例: invoke-virtual {v1}, getMessage:()Ljava/lang/String;
字节码: 6e 10 33 00 01 00
        │  │  └─┴─ 方法索引 0x0033 (getMessage)
        │  └─────── AG (A=1, G=v0)
        └────────── 操作码 0x6e
                   └─┴─ FEDC = 0x0001 (v1)

解析:
- invoke-virtual {v1}, Ljava/lang/ArithmeticException;.getMessage:()Ljava/lang/String; // method@0033
- 调用 v1.getMessage()，返回值通过 move-result-object 获取

总长度: 6 字节
```

#### invoke-custom {vC..vG}, call_site@BBBB (0xfc)

**格式**: 35c

```
字节码: fc 10 00 00 03 00
        │  │  └─┴─ call site 索引 0x0000
        │  └─────── AG (A=1, G=v0)
        └────────── 操作码 0xfc
                   └─┴─ FEDC = 0x0003 (v3)

解析:
- invoke-custom {v3}, call_site@0000
- 调用动态方法（Lambda 表达式或字符串拼接）
- call site 0 通常是 Java 编译器生成的字符串拼接优化

总长度: 6 字节
```

### Java 示例

```java
obj.method();                   // invoke-virtual {v0}, Lclass;.method:()V
super.method();                 // invoke-super {v0}, Lparent;.method:()V
new Foo();                      // new-instance v0, LFoo;
                                // invoke-direct {v0}, LFoo;.<init>:()V
Foo.staticMethod();             // invoke-static {}, LFoo;.staticMethod:()V
list.add("item");               // invoke-interface {v0, v1}, Ljava/util/List;.add:(Ljava/lang/Object;)Z
```

---

## 11. 一元运算指令

### 指令列表

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `neg-int vA, vB` | 0x7b | 12x | 整数取负 |
| `not-int vA, vB` | 0x7c | 12x | 整数按位取反 |
| `neg-long vA, vB` | 0x7d | 12x | 长整数取负 |
| `not-long vA, vB` | 0x7e | 12x | 长整数按位取反 |
| `neg-float vA, vB` | 0x7f | 12x | 浮点数取负 |
| `neg-double vA, vB` | 0x80 | 12x | 双精度浮点数取负 |

### 十六进制解析

```asm
neg-int v0, v1 (0x7b)
字节码: 7b 10
        │  └┴─ BA (B=v1源, A=v0目标)
        └──── 操作码 0x7b

解析:
- neg-int v0, v1
- v0 = -v1 (整数取负)
总长度: 2 字节
```

### Java 示例

```java
int x = 10;
int y = -x;                     // neg-int v1, v0
int z = ~x;                     // not-int v2, v0
```

---

## 12. 二元运算指令

### 12.1 算术运算（普通格式 23x）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `add-int vAA, vBB, vCC` | 0x90 | 23x | 整数加法 |
| `sub-int vAA, vBB, vCC` | 0x91 | 23x | 整数减法 |
| `mul-int vAA, vBB, vCC` | 0x92 | 23x | 整数乘法 |
| `div-int vAA, vBB, vCC` | 0x93 | 23x | 整数除法 |
| `rem-int vAA, vBB, vCC` | 0x94 | 23x | 整数取余 |
| `and-int vAA, vBB, vCC` | 0x95 | 23x | 整数按位与 |
| `or-int vAA, vBB, vCC` | 0x96 | 23x | 整数按位或 |
| `xor-int vAA, vBB, vCC` | 0x97 | 23x | 整数按位异或 |
| `shl-int vAA, vBB, vCC` | 0x98 | 23x | 整数左移 ❌ |
| `shr-int vAA, vBB, vCC` | 0x99 | 23x | 整数算术右移 ❌ |
| `ushr-int vAA, vBB, vCC` | 0x9a | 23x | 整数逻辑右移 ❌ |

### 12.2 长整数运算

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `add-long vAA, vBB, vCC` | 0x9b | 23x | 长整数加法 |
| `sub-long vAA, vBB, vCC` | 0x9c | 23x | 长整数减法 |
| `mul-long vAA, vBB, vCC` | 0x9d | 23x | 长整数乘法 |
| `div-long vAA, vBB, vCC` | 0x9e | 23x | 长整数除法 |
| `rem-long vAA, vBB, vCC` | 0x9f | 23x | 长整数取余 |
| `and-long vAA, vBB, vCC` | 0xa0 | 23x | 长整数按位与 |
| `or-long vAA, vBB, vCC` | 0xa1 | 23x | 长整数按位或 |
| `xor-long vAA, vBB, vCC` | 0xa2 | 23x | 长整数按位异或 |
| `shl-long vAA, vBB, vCC` | 0xa3 | 23x | 长整数左移 |
| `shr-long vAA, vBB, vCC` | 0xa4 | 23x | 长整数算术右移 |
| `ushr-long vAA, vBB, vCC` | 0xa5 | 23x | 长整数逻辑右移 |

### 12.3 浮点运算

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `add-float vAA, vBB, vCC` | 0xa6 | 23x | 浮点数加法 |
| `sub-float vAA, vBB, vCC` | 0xa7 | 23x | 浮点数减法 |
| `mul-float vAA, vBB, vCC` | 0xa8 | 23x | 浮点数乘法 |
| `div-float vAA, vBB, vCC` | 0xa9 | 23x | 浮点数除法 |
| `rem-float vAA, vBB, vCC` | 0xaa | 23x | 浮点数取余 |
| `add-double vAA, vBB, vCC` | 0xab | 23x | 双精度浮点数加法 |
| `sub-double vAA, vBB, vCC` | 0xac | 23x | 双精度浮点数减法 |
| `mul-double vAA, vBB, vCC` | 0xad | 23x | 双精度浮点数乘法 |
| `div-double vAA, vBB, vCC` | 0xae | 23x | 双精度浮点数除法 |
| `rem-double vAA, vBB, vCC` | 0xaf | 23x | 双精度浮点数取余 |

### 12.4 二元运算（2addr 格式）

这些指令使用两地址格式，目标和源寄存器之一相同，可以节省字节码空间。

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `add-int/2addr vA, vB` | 0xb0 | 12x | vA = vA + vB（整数加法） |
| `sub-int/2addr vA, vB` | 0xb1 | 12x | vA = vA - vB（整数减法） |
| `mul-int/2addr vA, vB` | 0xb2 | 12x | vA = vA * vB（整数乘法） |
| `div-int/2addr vA, vB` | 0xb3 | 12x | vA = vA / vB（整数除法） |
| `rem-int/2addr vA, vB` | 0xb4 | 12x | vA = vA % vB（整数取余） |
| `and-int/2addr vA, vB` | 0xb5 | 12x | vA = vA & vB（整数按位与） |
| `or-int/2addr vA, vB` | 0xb6 | 12x | vA = vA \| vB（整数按位或） |
| `xor-int/2addr vA, vB` | 0xb7 | 12x | vA = vA ^ vB（整数按位异或） |
| `shl-int/2addr vA, vB` | 0xb8 | 12x | vA = vA << vB（整数左移） ❌ |
| `shr-int/2addr vA, vB` | 0xb9 | 12x | vA = vA >> vB（整数算术右移） ❌ |
| `ushr-int/2addr vA, vB` | 0xba | 12x | vA = vA >>> vB（整数逻辑右移） ❌ |
| `add-long/2addr vA, vB` | 0xbb | 12x | vA = vA + vB（长整数加法） |
| `sub-long/2addr vA, vB` | 0xbc | 12x | vA = vA - vB（长整数减法） |
| `mul-long/2addr vA, vB` | 0xbd | 12x | vA = vA * vB（长整数乘法） |
| `div-long/2addr vA, vB` | 0xbe | 12x | vA = vA / vB（长整数除法） |
| `rem-long/2addr vA, vB` | 0xbf | 12x | vA = vA % vB（长整数取余） |
| `and-long/2addr vA, vB` | 0xc0 | 12x | vA = vA & vB（长整数按位与） |
| `or-long/2addr vA, vB` | 0xc1 | 12x | vA = vA \| vB（长整数按位或） |
| `xor-long/2addr vA, vB` | 0xc2 | 12x | vA = vA ^ vB（长整数按位异或） |
| `shl-long/2addr vA, vB` | 0xc3 | 12x | vA = vA << vB（长整数左移） |
| `shr-long/2addr vA, vB` | 0xc4 | 12x | vA = vA >> vB（长整数算术右移） |
| `ushr-long/2addr vA, vB` | 0xc5 | 12x | vA = vA >>> vB（长整数逻辑右移） |
| `add-float/2addr vA, vB` | 0xc6 | 12x | vA = vA + vB（浮点数加法） |
| `sub-float/2addr vA, vB` | 0xc7 | 12x | vA = vA - vB（浮点数减法） |
| `mul-float/2addr vA, vB` | 0xc8 | 12x | vA = vA * vB（浮点数乘法） |
| `div-float/2addr vA, vB` | 0xc9 | 12x | vA = vA / vB（浮点数除法） |
| `rem-float/2addr vA, vB` | 0xca | 12x | vA = vA % vB（浮点数取余） |
| `add-double/2addr vA, vB` | 0xcb | 12x | vA = vA + vB（双精度浮点数加法） |
| `sub-double/2addr vA, vB` | 0xcc | 12x | vA = vA - vB（双精度浮点数减法） |
| `mul-double/2addr vA, vB` | 0xcd | 12x | vA = vA * vB（双精度浮点数乘法） |
| `div-double/2addr vA, vB` | 0xce | 12x | vA = vA / vB（双精度浮点数除法） |
| `rem-double/2addr vA, vB` | 0xcf | 12x | vA = vA % vB（双精度浮点数取余） |

### 12.5 二元运算（立即数格式 lit16）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `add-int/lit16 vA, vB, #+CCCC` | 0xd0 | 22s | vA = vB + lit16（加法） |
| `rsub-int vA, vB, #+CCCC` | 0xd1 | 22s | vA = lit16 - vB（反向减法） ❌ |
| `mul-int/lit16 vA, vB, #+CCCC` | 0xd2 | 22s | vA = vB * lit16（乘法） |
| `div-int/lit16 vA, vB, #+CCCC` | 0xd3 | 22s | vA = vB / lit16（除法） |
| `rem-int/lit16 vA, vB, #+CCCC` | 0xd4 | 22s | vA = vB % lit16（取余） |
| `and-int/lit16 vA, vB, #+CCCC` | 0xd5 | 22s | vA = vB & lit16（按位与） |
| `or-int/lit16 vA, vB, #+CCCC` | 0xd6 | 22s | vA = vB \| lit16（按位或） |
| `xor-int/lit16 vA, vB, #+CCCC` | 0xd7 | 22s | vA = vB ^ lit16（按位异或） |

### 12.6 二元运算（立即数格式 lit8）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `add-int/lit8 vAA, vBB, #+CC` | 0xd8 | 22b | vAA = vBB + lit8（加法） |
| `rsub-int/lit8 vAA, vBB, #+CC` | 0xd9 | 22b | vAA = lit8 - vBB（反向减法） |
| `mul-int/lit8 vAA, vBB, #+CC` | 0xda | 22b | vAA = vBB * lit8（乘法） |
| `div-int/lit8 vAA, vBB, #+CC` | 0xdb | 22b | vAA = vBB / lit8（除法） |
| `rem-int/lit8 vAA, vBB, #+CC` | 0xdc | 22b | vAA = vBB % lit8（取余） |
| `and-int/lit8 vAA, vBB, #+CC` | 0xdd | 22b | vAA = vBB & lit8（按位与） |
| `or-int/lit8 vAA, vBB, #+CC` | 0xde | 22b | vAA = vBB \| lit8（按位或） |
| `xor-int/lit8 vAA, vBB, #+CC` | 0xdf | 22b | vAA = vBB ^ lit8（按位异或） |
| `shl-int/lit8 vAA, vBB, #+CC` | 0xe0 | 22b | vAA = vBB << lit8（左移） |
| `shr-int/lit8 vAA, vBB, #+CC` | 0xe1 | 22b | vAA = vBB >> lit8（算术右移） |
| `ushr-int/lit8 vAA, vBB, #+CC` | 0xe2 | 22b | vAA = vBB >>> lit8（逻辑右移） |

### 十六进制解析示例

#### add-int vAA, vBB, vCC (0x90)

**格式**: 23x
**字节布局**: `90 AA BB CC`

```
字节码: 90 02 00 01
        │  │  │  └──── vCC = v1 (第二个操作数)
        │  │  └─────── vBB = v0 (第一个操作数)
        │  └────────── vAA = v2 (结果)
        └───────────── 操作码 0x90

解析:
- add-int v2, v0, v1
- v2 = v0 + v1 (整数加法)

对应 Java:
  int a = 100;  // v0
  int b = 50;   // v1
  int sum = a + b;  // v2 = v0 + v1

总长度: 4 字节
```

#### add-int/lit8 vAA, vBB, #+CC (0xd8)

**格式**: 22b
**字节布局**: `d8 AA BB CC`

```asm
字节码: d8 00 00 0a
        │  │  │  └──── CC = 0x0a = 10 (8位立即数)
        │  │  └─────── BB = 0x00 = v0 (源寄存器)
        │  └────────── AA = 0x00 = v0 (目标寄存器)
        └───────────── 操作码 0xd8

解析:
- add-int/lit8 v0, v0, #int 10
- v0 = v0 + 10

对应 Java: a = a + 10;
总长度: 4 字节
立即数范围: -128 到 127 (8位有符号)
```

#### div-int/lit8 vAA, vBB, #+CC (0xdb)

```asm
字节码: db 01 01 00
        │  │  │  └──── CC = 0x00 = 0 (除数)
        │  │  └─────── BB = 0x01 = v1 (被除数)
        │  └────────── AA = 0x01 = v1 (结果)
        └───────────── 操作码 0xdb

解析:
- div-int/lit8 v1, v1, #int 0
- v1 = v1 / 0 → **抛出 ArithmeticException**

注意: 这是异常处理示例中的除零操作
```

### Java 示例

```java
// 普通格式
int a = 10, b = 20;
int sum = a + b;                // add-int v2, v0, v1

// 2addr 格式
a += 5;                         // add-int/2addr v0, v1

// lit8 格式
a = a + 10;                     // add-int/lit8 v0, v0, #10

// lit16 格式
int c = a + 5000;               // add-int/lit16 v1, v0, #5000

// 长整数
long x = 1000L, y = 2000L;
long z = x + y;                 // add-long v4, v0, v2
```

---

## 13. 类型转换指令

### 指令列表

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `int-to-long vA, vB` | 0x81 | 12x | int → long |
| `int-to-float vA, vB` | 0x82 | 12x | int → float |
| `int-to-double vA, vB` | 0x83 | 12x | int → double |
| `long-to-int vA, vB` | 0x84 | 12x | long → int |
| `long-to-float vA, vB` | 0x85 | 12x | long → float |
| `long-to-double vA, vB` | 0x86 | 12x | long → double |
| `float-to-int vA, vB` | 0x87 | 12x | float → int |
| `float-to-long vA, vB` | 0x88 | 12x | float → long |
| `float-to-double vA, vB` | 0x89 | 12x | float → double |
| `double-to-int vA, vB` | 0x8a | 12x | double → int |
| `double-to-long vA, vB` | 0x8b | 12x | double → long |
| `double-to-float vA, vB` | 0x8c | 12x | double → float |
| `int-to-byte vA, vB` | 0x8d | 12x | int → byte |
| `int-to-char vA, vB` | 0x8e | 12x | int → char |
| `int-to-short vA, vB` | 0x8f | 12x | int → short |

### 十六进制解析

```asm
int-to-long v0, v2 (0x81)
字节码: 81 20
        │  └┴─ BA (B=v2源, A=v0目标)
        └──── 操作码 0x81

解析:
- int-to-long v0, v2
- v0-v1 = (long)v2 (32位→64位)
总长度: 2 字节
```

### Java 示例

```java
int i = 100;
long l = (long) i;              // int-to-long v1, v0
float f = (float) i;            // int-to-float v2, v0
byte b = (byte) i;              // int-to-byte v3, v0
```

---

## 14. 比较指令

### 指令列表

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `cmpl-float vAA, vBB, vCC` | 0x2d | 23x | 浮点比较（NaN 返回 -1） |
| `cmpg-float vAA, vBB, vCC` | 0x2e | 23x | 浮点比较（NaN 返回 1） |
| `cmpl-double vAA, vBB, vCC` | 0x2f | 23x | 双精度比较（NaN 返回 -1） |
| `cmpg-double vAA, vBB, vCC` | 0x30 | 23x | 双精度比较（NaN 返回 1） |
| `cmp-long vAA, vBB, vCC` | 0x31 | 23x | 长整数比较 |

**返回值**:
- `1` 如果 vBB > vCC
- `0` 如果 vBB == vCC
- `-1` 如果 vBB < vCC

### 十六进制解析

```asm
cmp-long v4, v0, v2 (0x31)
字节码: 31 04 00 02
        │  │  │  └──── vCC = v2 (第二个long)
        │  │  └─────── vBB = v0 (第一个long)
        │  └────────── vAA = v4 (结果)
        └───────────── 操作码 0x31

解析:
- cmp-long v4, v0, v2
- 比较 v0-v1 和 v2-v3
- v4 = { 1 if v0>v2, 0 if v0==v2, -1 if v0<v2 }

总长度: 4 字节
```

### Java 示例

```java
float f1 = 3.14f, f2 = 2.71f;
if (f1 > f2) {                  // cmpg-float v2, v0, v1
                                // if-gtz v2, +offset
    // ...
}

long l1 = 100L, l2 = 200L;
if (l1 < l2) {                  // cmp-long v2, v0, v1
                                // if-ltz v2, +offset
    // ...
}
```

---

## 15. 控制流指令

### 15.1 无条件跳转

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `goto +AA` | 0x28 | 10t | 跳转（8 位偏移） |
| `goto/16 +AAAA` | 0x29 | 20t | 跳转（16 位偏移） ❌ |
| `goto/32 +AAAAAAAA` | 0x2a | 30t | 跳转（32 位偏移） ❌ |

### 15.2 条件跳转（两值比较）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `if-eq vA, vB, +CCCC` | 0x32 | 22t | 如果 vA == vB 则跳转 |
| `if-ne vA, vB, +CCCC` | 0x33 | 22t | 如果 vA != vB 则跳转 |
| `if-lt vA, vB, +CCCC` | 0x34 | 22t | 如果 vA < vB 则跳转 |
| `if-ge vA, vB, +CCCC` | 0x35 | 22t | 如果 vA >= vB 则跳转 |
| `if-gt vA, vB, +CCCC` | 0x36 | 22t | 如果 vA > vB 则跳转 |
| `if-le vA, vB, +CCCC` | 0x37 | 22t | 如果 vA <= vB 则跳转 |

### 15.3 条件跳转（与零比较）

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `if-eqz vAA, +BBBB` | 0x38 | 21t | 如果 vAA == 0 则跳转 |
| `if-nez vAA, +BBBB` | 0x39 | 21t | 如果 vAA != 0 则跳转 |
| `if-ltz vAA, +BBBB` | 0x3a | 21t | 如果 vAA < 0 则跳转 |
| `if-gez vAA, +BBBB` | 0x3b | 21t | 如果 vAA >= 0 则跳转 |
| `if-gtz vAA, +BBBB` | 0x3c | 21t | 如果 vAA > 0 则跳转 |
| `if-lez vAA, +BBBB` | 0x3d | 21t | 如果 vAA <= 0 则跳转 |

### 15.4 Switch 指令

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `packed-switch vAA, +BBBBBBBB` | 0x2b | 31t | 打包 switch（连续 case 值） |
| `sparse-switch vAA, +BBBBBBBB` | 0x2c | 31t | 稀疏 switch（非连续 case 值） |

### 十六进制解析示例

#### goto +AA (0x28)

**格式**: 10t
**字节布局**: `28 AA` (AA 是有符号偏移)

```asm
字节码: 28 0f
        │  └──── 偏移量 +15 (0x0f)
        └─────── 操作码 0x28

解析:
- goto 0015 // +000f
- 跳转到当前位置 + 15 字节处
- 用于跳过异常处理代码

偏移量范围: -128 到 +127 字节 (8位有符号)
总长度: 2 字节
```

#### if-eq vA, vB, +CCCC (0x32)

**格式**: 22t
**字节布局**: `32 AB CCCC`

```asm
字节码: 32 01 05 00
        │  │  └─┴─ 偏移量 +0x0005
        │  └─────── AB (A=v1, B=v0)
        └────────── 操作码 0x32

解析:
- if-eq v1, v0, +0005
- 如果 v1 == v0，跳转到 +5 字节处

对应 Java: if (x == y) { ... }
总长度: 4 字节
偏移量范围: -32768 to +32767 字节
```

#### if-eqz vAA, +BBBB (0x38)

**格式**: 21t

```asm
字节码: 38 00 08 00
        │  │  └─┴─ 偏移量 +0x0008
        │  └─────── 寄存器 v0
        └────────── 操作码 0x38

解析:
- if-eqz v0, +0008
- 如果 v0 == 0，跳转到 +8 字节处

对应 Java: if (x == 0) { ... }
总长度: 4 字节
```

### Java 示例

```java
// if-eq, if-ne
if (x == 10) {                  // const/16 v1, #10
                                // if-eq v0, v1, +offset
    // ...
}

// if-gtz
if (x > 0) {                    // if-gtz v0, +offset
    // ...
}

// packed-switch
switch (x) {                    // packed-switch v0, +data_offset
    case 1: break;              // 连续的 case 值
    case 2: break;
    case 3: break;
}

// sparse-switch
switch (x) {                    // sparse-switch v0, +data_offset
    case 10: break;             // 非连续的 case 值
    case 50: break;
    case 100: break;
}
```

---

## 16. 异常处理指令

### 指令列表

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `throw vAA` | 0x27 | 11x | 抛出异常 |
| `move-exception vAA` | 0x0d | 11x | 保存捕获的异常对象到寄存器 |

### 十六进制解析

#### throw vAA (0x27)

```asm
字节码: 27 00
        │  └──── 异常对象寄存器 v0
        └─────── 操作码 0x27

解析:
- throw v0
- 抛出 v0 中的异常对象

对应 Java: throw new RuntimeException();
总长度: 2 字节
```

#### move-exception vAA (0x0d)

```asm
字节码: 0d 01
        │  └──── 目标寄存器 v1
        └─────── 操作码 0x0d

解析:
- move-exception v1
- 将捕获的异常对象保存到 v1
- 这是 catch 块的第一条指令

对应 Java: catch (Exception e) { // e 存入 v1
总长度: 2 字节
```

### 异常处理完整示例

**DEX 反汇编**:
```asm
0000: const-string v0, "Finally block" // string@000d
0002: const/16 v1, #int 10 // #a
0004: div-int/lit8 v1, v1, #int 0 // #00
0006: goto 0015 // +000f
0007: move-exception v1
0008: sget-object v2, Ljava/lang/System;.out:Ljava/io/PrintStream; // field@0017
000a: invoke-virtual {v1}, Ljava/lang/ArithmeticException;.getMessage:()Ljava/lang/String; // method@0033
000d: move-result-object v3
000e: invoke-custom {v3}, call_site@0000
0011: move-result-object v3
0012: invoke-virtual {v2, v3}, Ljava/io/PrintStream;.println:(Ljava/lang/String;)V // method@0032
```

**字节码详细解析**:

```asm
1. const-string (偏移 0000)
字节码: 1a 00 0d 00
        │  │  └─┴─ 字符串索引 0x000d ("Finally block")
        │  └─────── 目标寄存器 v0
        └────────── 操作码 0x1a

2. const/16 (偏移 0002)
字节码: 13 01 0a 00
        │  │  └─┴─ 立即数 10 (0x000a)
        │  └─────── 目标寄存器 v1
        └────────── 操作码 0x13

3. div-int/lit8 (偏移 0004) - 触发异常
字节码: db 01 01 00
        │  │  │  └──── 8位立即数 0 (除数)
        │  │  └─────── 源寄存器 v1
        │  └────────── 目标寄存器 v1
        └───────────── 操作码 0xdb

作用: v1 = v1 / 0 → **抛出 ArithmeticException**

4. goto (偏移 0006) - 正常流程跳过异常处理
字节码: 28 0f
        │  └──── 8位偏移量 +15
        └─────── 操作码 0x28

5. move-exception (偏移 0007) - **异常处理入口**
字节码: 0d 01
        │  └──── 目标寄存器 v1
        └─────── 操作码 0x0d

作用: 将捕获的异常对象保存到 v1 寄存器
说明: 这是 catch 块的第一条指令

6. sget-object (偏移 0008)
字节码: 62 02 17 00
        │  │  └─┴─ 字段索引 0x0017 (System.out)
        │  └─────── 目标寄存器 v2
        └────────── 操作码 0x62

作用: 读取静态字段 System.out 到 v2

7. invoke-virtual (偏移 000a)
字节码: 6e 10 33 00 01 00
        │  │  └─┴─ 方法索引 0x0033 (getMessage)
        │  └─────── AG (1个参数)
        └────────── 操作码 0x6e
                   └─┴─ 参数 v1 (异常对象)

作用: 调用 v1.getMessage() 获取异常消息

8. move-result-object (偏移 000d)
字节码: 0c 03
        │  └──── 目标寄存器 v3
        └─────── 操作码 0x0c

作用: 将 getMessage() 的返回值（String）移动到 v3

9. invoke-custom (偏移 000e)
字节码: fc 10 00 00 03 00
        │  │  └─┴─ call site 索引 0x0000
        │  └─────── AG (1个参数)
        └────────── 操作码 0xfc
                   └─┴─ 参数 v3

作用: 调用动态方法（可能是字符串拼接优化）

10. invoke-virtual (偏移 0012)
字节码: 6e 20 32 00 32 00
        │  │  └─┴─ 方法索引 0x0032 (println)
        │  └─────── AG (2个参数)
        └────────── 操作码 0x6e
                   └─┴─ 参数 v2, v3

作用: 调用 System.out.println(v3) 打印异常消息
```

### Java 示例

```java
try {
    int x = 10 / 0;  // Will throw ArithmeticException
} catch (ArithmeticException e) {  // move-exception v1
    System.out.println("Caught: " + e.getMessage());
}

// 显式抛出异常
throw new RuntimeException("error");  // new-instance v0, Ljava/lang/RuntimeException;
                                      // invoke-direct {v0}, <init>
                                      // throw v0
```

---

## 17. 对象操作指令

### 指令列表

| 指令 | 操作码 | 格式 | 说明 |
|------|--------|------|------|
| `new-instance vAA, type@BBBB` | 0x22 | 21c | 创建新对象实例（未初始化） |

### 十六进制解析

```asm
字节码: 22 00 11 00
        │  │  └─┴─ 类型索引 0x0011 (Ljava/lang/Object;)
        │  └─────── 目标寄存器 v0
        └────────── 操作码 0x22

解析:
- new-instance v0, Ljava/lang/Object; // type@0011
- 创建新的 Object 实例（未初始化）
- v0 存储对象引用
- 之后需要调用 invoke-direct {v0}, <init> 初始化

对应 Java: Object obj = new Object();
  → new-instance v0, Ljava/lang/Object;
  → invoke-direct {v0}, Ljava/lang/Object;.<init>:()V

总长度: 4 字节
```

### Java 示例

```java
Object obj = new Object();      // new-instance v0, Ljava/lang/Object;
                                // invoke-direct {v0}, Ljava/lang/Object;.<init>:()V
```

---

## 未覆盖指令分析

- **未覆盖指令数**: 26 条
- **主要原因**: 极端代码条件、高级特性、编译器优化

### 分类详解

#### 超大寄存器索引指令（3 条）

这些指令需要使用编号 ≥ 256 的寄存器，在常规 Java 代码中极难触发：

| 指令 | 操作码 | 原因 |
|------|--------|------|
| `move/16` | 0x03 | 需要 256+ 寄存器 |
| `move-wide/16` | 0x06 | 需要 256+ 寄存器（64位） |
| `move-object/16` | 0x09 | 需要 256+ 寄存器（对象） |

**原因**: Java 编译器极少生成需要 256 个以上局部变量的方法，通常方法会被拆分重构。

#### 超长跳转指令（2 条）

需要方法体包含极长的字节码序列：

| 指令 | 操作码 | 原因 |
|------|--------|------|
| `goto/16` | 0x29 | 16位偏移跳转，需要跳转距离 > 127 字节 |
| `goto/32` | 0x2a | 32位偏移跳转，需要跳转距离 > 32767 字节 |

**原因**: 普通 Java 方法很少包含数千行字节码，且编译器会优化控制流。

#### 超大字符串池指令（1 条）

| 指令 | 操作码 | 原因 |
|------|--------|------|
| `const-string/jumbo` | 0x1b | 字符串池索引 > 65535 |

**原因**: 需要程序包含超过 65535 个不同的字符串常量，仅在超大型应用中出现。

#### 整数移位优化指令（6 条）

这些指令被编译器优化为更高效的 lit8 格式：

| 指令 | 操作码 | 原因 |
|------|--------|------|
| `shl-int` | 0x98 | 被优化为 `shl-int/lit8` |
| `shr-int` | 0x99 | 被优化为 `shr-int/lit8` |
| `ushr-int` | 0x9a | 被优化为 `ushr-int/lit8` |
| `shl-int/2addr` | 0xb8 | 被优化为 `shl-int/lit8` |
| `shr-int/2addr` | 0xb9 | 被优化为 `shr-int/lit8` |
| `ushr-int/2addr` | 0xba | 被优化为 `ushr-int/lit8` |

**原因**: Java 移位操作的第二个操作数（移位量）通常是常量且范围在 0-31，编译器优先使用 lit8 格式。

#### 方法句柄指令（Android 8+）（3 条）

需要使用 `java.lang.invoke.MethodHandle` API：

| 指令 | 操作码 | 原因 |
|------|--------|------|
| `invoke-polymorphic` | 0xfa | MethodHandle.invoke() |
| `invoke-polymorphic/range` | 0xfb | MethodHandle.invoke() 范围调用 |
| `const-method-handle` | 0xfe | 加载方法句柄常量 |

**原因**: 方法句柄 API 较少在常规 Android 开发中使用，主要用于框架级别的反射优化。

#### 方法类型指令（Android 8+）（1 条）

| 指令 | 操作码 | 原因 |
|------|--------|------|
| `const-method-type` | 0xff | 加载 MethodType 常量 |

**原因**: 需要使用 `java.lang.invoke.MethodType` API，同样较少使用。

#### 动态调用范围指令（1 条）

| 指令 | 操作码 | 原因 |
|------|--------|------|
| `invoke-custom/range` | 0xfd | Lambda 表达式范围调用 |

**原因**: 虽然已覆盖 `invoke-custom`，但范围版本需要 Lambda 表达式的参数数量 > 5 个，较为罕见。

#### 反向减法指令（1 条）

| 指令 | 操作码 | 原因 |
|------|--------|------|
| `rsub-int` | 0xd1 | vA = lit16 - vB（反向减法，16位立即数） |

**原因**: 编译器通常优化为 `rsub-int/lit8` 或转换为 `sub-int`，仅在立即数超出 8 位范围时才使用。

### 可以简单覆盖的指令建议

#### 可以尝试覆盖（3 条）

1. **`rsub-int`** (0xd1)
   ```java
   // 可能触发 rsub-int
   int a = 32768 - x;  // 超出 lit8 范围
   ```

2. **`shl-int`** / **`shr-int`** / **`ushr-int`** (0x98-0x9a)
   ```java
   // 使用变量作为移位量
   int shift = getShiftAmount();
   int result = x << shift;  // 可能生成 shl-int
   ```

3. **`invoke-polymorphic`** (0xfa) - 需要 Android 8+ 和 MethodHandle
   ```java
   import java.lang.invoke.*;

   MethodHandle mh = MethodHandles.lookup()
       .findVirtual(String.class, "length",
                    MethodType.methodType(int.class));
   int len = (int) mh.invoke("hello");  // 可能生成 invoke-polymorphic
   ```

#### 其它指令（23 条）

其余 23 条指令需要特殊环境或极端代码结构，。

### 覆盖率分类统计

| 类别 | 指令数量 | 覆盖率 | 说明 |
|------|---------|-------|------|
| **核心指令** | 180 | 100% | 日常开发常用指令 |
| **优化指令** | 15 | 40% | 编译器优化相关（移位、反向减法） |
| **高级特性** | 7 | 14% | 方法句柄、方法类型 |
| **极端情况** | 6 | 0% | 超大索引、超长跳转 |
| **保留操作码** | 16 | 0% | 未定义功能 |
| **总计** | **224** | **88.4%** | 整体覆盖率 |

---

### 寄存器编码

#### 4位寄存器 (AB 格式)

```asm
AB = 0x32
- A (低4位) = 0x2 = v2
- B (高4位) = 0x3 = v3
```

#### 参数寄存器 (35c 格式 FEDC)

```asm
FEDC = 0x4321
- C = 0x1 = v1 (第5个参数)
- D = 0x2 = v2 (第4个参数)
- E = 0x3 = v3 (第3个参数)
- F = 0x4 = v4 (第2个参数)
- G (从 AG 字段) = 第1个参数
```

### 寄存器命名约定

- `v0`, `v1`, ... - 局部寄存器（普通寄存器）
- `p0`, `p1`, ... - 参数寄存器（方法参数）
- 对于实例方法，`p0` 是 `this` 引用

**示例**:

```java
void method(int a, String b) {
    // p0 = this
    // p1 = a
    // p2 = b
    int local = 10;  // v0 = 10
}
```

### 数据类型编码

| Java 类型 | 类型描述符 | 说明 |
|-----------|-----------|------|
| `void` | `V` | 仅用于返回类型 |
| `boolean` | `Z` | 布尔值 |
| `byte` | `B` | 有符号字节 |
| `short` | `S` | 有符号短整数 |
| `char` | `C` | Unicode 字符 |
| `int` | `I` | 有符号整数 |
| `long` | `J` | 有符号长整数 |
| `float` | `F` | 单精度浮点数 |
| `double` | `D` | 双精度浮点数 |
| `String` | `Ljava/lang/String;` | 类对象 |
| `int[]` | `[I` | 整数数组 |
| `String[][]` | `[[Ljava/lang/String;` | 二维字符串数组 |

### 方法描述符

方法描述符格式: `(参数类型...)返回类型`

**示例**:
- `()V` - void 无参方法
- `(I)V` - void 方法，接受一个 int 参数
- `(ILjava/lang/String;)Z` - boolean 方法，接受 int 和 String 参数
- `([I)I` - int 方法，接受 int 数组参数

### 验证命令

运行以下命令查看当前覆盖情况：

```bash
cd "/Volumes/macOS/feicong-book/DEX文件格式详解"
just build
just coverage
```

### 参考资源

- [Dalvik 字节码格式](https://source.android.com/docs/core/runtime/dalvik-bytecode)
- [Dalvik 指令格式](https://source.android.com/docs/core/runtime/instruction-formats)
- [DEX 文件格式](https://source.android.com/docs/core/runtime/dex-format)

---
