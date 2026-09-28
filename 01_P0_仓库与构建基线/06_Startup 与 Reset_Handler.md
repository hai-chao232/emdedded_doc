---
tags: [P0, startup, reset-handler, vector-table, cortex-m]
stage: P0
---

# 06｜Startup 与 Reset_Handler

> [!tip] 深挖入口
> - [[10_基础知识体系/01_计算机与MCU基础/03_CPU 寄存器 指令 与执行]]
> - [[10_基础知识体系/01_计算机与MCU基础/13_函数地址与代码地址到底是什么]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/05_Cortex-M Reset 到底发生了什么]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/06_Startup 文件到底是什么]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/07_Reset_Handler 逐段看]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/08_data copy 与 bss clear 为什么由 Startup 做]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/15_Thumb状态与函数地址最低位]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/18_读Startup所需的最小Thumb汇编]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/19_C与汇编如何互相调用_AAPCS最小理解]]
> - [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]
> - [[10_基础知识体系/02_构建与链接/20_NOBITS NOLOAD 与为什么RAM占用不等于Flash占用]]
> - [[10_基础知识体系/04_验证与调试/09_怎样验证 Reset_Handler 到 main]]

> [!abstract] 本节目标
> 本页回答“链接完成后，CPU 如何从复位状态一路运行到 C 代码”。
> 局部机制请跳到基础知识页。

## 1. 最重要的纠正：CPU 不认识 `main`

`main()` 是 C/C++ 运行环境里的约定入口，不是 Cortex-M 硬件规则。

Cortex-M 在复位时只做硬件规定的动作，其中最关键的是读取向量表。

深入：
- [[10_基础知识体系/01_计算机与MCU基础/03_CPU 寄存器 指令 与执行]]
- [[10_基础知识体系/01_计算机与MCU基础/13_函数地址与代码地址到底是什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/05_Cortex-M Reset 到底发生了什么]]

## 2. 向量表不是抽象概念，而是一串机器可读取的数据

逻辑上可以画成：

```text
vector[0]  初始 MSP
vector[1]  Reset_Handler
vector[2]  NMI_Handler
vector[3]  HardFault_Handler
...
```

实际 Flash 中只是连续的 32 位数值。

谁负责什么（与 [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]] 的五“谁”一致）：

```text
Startup 源码       → 定义表项内容与 Handler 名字
Assembler + Linker → 把名字解析成最终数值
Linker Script      → 安排最终地址并保留（KEEP）
烧录工具           → 把最终镜像写进 Flash
Cortex-M 硬件      → 复位/异常时读取向量表
```

深入：
- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]

## 3. Cortex-M 复位后的第一组关键动作

概念模型：

```text
Reset
  ↓
读取向量表第 0 项
  ↓
MSP ← vector[0]
  ↓
读取向量表第 1 项
  ↓
PC ← Reset_Handler 入口值
  ↓
开始执行 Reset_Handler
```

这里最关键的是区分：

```text
地址
```

和：

```text
这个地址中保存的值
```

例如：

```text
0x08000000 是某个地址
[0x08000000] 是该地址处保存的 32 位数
```

如果 `[0x08000000] = 0x20020000`，那么被加载进 MSP 的是 `0x20020000`。

## 4. 为什么函数会有“地址”

函数编译后就是一串机器指令，被链接器放到 `.text` 某一段地址范围。

例如：

```text
08000278 ... Reset_Handler
```

可以理解为：

> Reset_Handler 的代码入口位于 Flash 附近的这个地址。

因此向量表中才能保存它的入口值。

深入：
- [[10_基础知识体系/01_计算机与MCU基础/13_函数地址与代码地址到底是什么]]

## 5. 为什么向量表里的函数入口可能是奇数

Cortex-M 使用 Thumb 指令状态。函数指针/异常入口值的 bit0 带有 Thumb 状态含义。

所以你可能看到：

```text
符号地址：        0x08000278
向量表中的入口值：0x08000279
```

这通常不是错位 1 字节。

深入：
- [[10_基础知识体系/03_Cortex-M与启动中断/15_Thumb状态与函数地址最低位]]

## 6. Startup 文件通常包含什么

典型 Startup 至少包含：

```text
向量表
Reset_Handler
默认异常/中断 Handler
weak alias / weak symbol
```

它不是“神秘启动器”，它同样会被汇编/编译、进入 `.o`、参与链接，最终成为程序本身的一部分。

深入：
- [[10_基础知识体系/03_Cortex-M与启动中断/06_Startup 文件到底是什么]]

## 7. Reset_Handler 为什么必须初始化 C 运行环境

复位后，RAM 不会自动变成 ELF 中描述的 C 变量状态。

因此 Reset_Handler（或者它调用的 runtime）需要完成至少两类工作。

### 7.1 `.data` copy

```text
Flash 初始化镜像
_sidata
   │
   │ copy
   ▼
_sdata ........ _edata
RAM
```

### 7.2 `.bss` clear

```text
_sbss ........ _ebss
        ↓
全部写 0
```

只有完成这些操作后：

```c
uint32_t counter = 10;
uint32_t flag;
```

才满足 C 语言层面的预期：

```text
counter == 10
flag == 0
```

深入：
- [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]
- [[10_基础知识体系/02_构建与链接/20_NOBITS NOLOAD 与为什么RAM占用不等于Flash占用]]
- [[10_基础知识体系/03_Cortex-M与启动中断/08_data copy 与 bss clear 为什么由 Startup 做]]

## 8. C Runtime 初始化：一个常见误读

反汇编里常能看到：

```asm
bl __libc_init_array
```

一个自然但**不准确**的读法是：

> “Startup 汇编里直接 `bl .init_array`。”

实际不是这样。`.init_array` 是一个 **section**（函数指针数组），不是一段可以 `bl` 过去的代码。

真实链条是：

```text
bl __libc_init_array        ← 汇编调用一个 C 函数
        ↓
__libc_init_array (C 函数)
        ↓
遍历 .init_array 中的函数指针
        ↓
逐个调用它们
```

所以：

```text
.init_array → 数据（函数指针表）
__libc_init_array → 代码（遍历并调用这张表的 C 函数）
```

C++ 的全局对象构造函数、标了 `__attribute__((constructor))` 的函数，都靠这条链被调用。

> [!note] 为什么这个区分重要
> 读到 `bl __libc_init_array` 时，如果你以为它在 `bl` 一个 section，后面看到 `.init_array` 出现在 `objdump -h` 里就会彻底混乱。
> 记住：**能 `bl` 的只有代码；`.init_array` 是表。**

## 9. `SystemInit()` 在哪里

CMSIS/芯片厂商启动代码常在进入 `main()` 前调用 `SystemInit()`。

它的具体职责依芯片和工程而定，常与时钟/核心系统配置有关。

这里不要背“SystemInit 就等于时钟初始化”这种过度简化。

正确方法是：

1. 找到当前工程真正链接到的 `SystemInit` 定义。
2. 阅读源码。
3. 必要时通过反汇编/断点确认它确实被调用。

深入：
- [[10_基础知识体系/03_Cortex-M与启动中断/09_SystemInit 到底处在什么位置]]

## 10. weak handler 为什么好用

Startup 常给中断 Handler 提供弱默认实现。

概念上：

```text
Startup:
GPIO_IRQHandler  → weak/default

你的应用:
GPIO_IRQHandler  → strong definition
```

链接时强定义可以替代弱定义。

因此你不必去修改厂商 Startup 文件，也可以提供自己的中断处理函数。

深入：
- [[10_基础知识体系/03_Cortex-M与启动中断/14_Weak Handler 为什么能被覆盖]]

## 11. 读 Reset_Handler 只需要最小汇编能力

当前阶段不需要系统学完 ARM 汇编。

至少能识别：

```text
ldr  → 从内存读取/装地址
str  → 写内存
mov  → 传值
cmp  → 比较
b    → 跳转
bl   → 调用函数并设置 LR
```

特别提醒 `ldr` 的两种读法差别：

```asm
ldr r0, [r1]      ← 加载“地址指向的数据”
ldr r0, =_sdata   ← 加载“地址本身”
```

读反汇编时要一直问自己：**这是在加载地址，还是在加载地址里的内容？**

深入：
- [[10_基础知识体系/03_Cortex-M与启动中断/18_读Startup所需的最小Thumb汇编]]
- [[10_基础知识体系/03_Cortex-M与启动中断/19_C与汇编如何互相调用_AAPCS最小理解]]

## 12. 本页与 Linker Script 的接口

Linker Script：

```text
决定布局 + 产生边界符号
```

Startup：

```text
读取这些符号 + 执行初始化
```

最典型接口：

```text
_estack
_sidata
_sdata
_edata
_sbss
_ebss
```

因此真正理解启动链时，05 与 06 必须来回对照。

## 13. 离开本页前

闭卷回答：

1. CPU 为什么不会直接找 `main()`？
2. 向量表第 0、1 项分别干什么？
3. 谁定义向量表、谁决定其地址、谁运行时读取它？
4. 为什么函数能被向量表“指向”？
5. 为什么函数入口值可能是奇数？
6. Reset_Handler 为什么需要 copy `.data` 和 clear `.bss`？
7. `.init_array` 和 `__libc_init_array` 分别是什么？哪个能被 `bl`？
8. weak handler 解决了什么工程问题？

下一步：[[01_P0_仓库与构建基线/07_最小 Executable 与 ELF 体检]]
