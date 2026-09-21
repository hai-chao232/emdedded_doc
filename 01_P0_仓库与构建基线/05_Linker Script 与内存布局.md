---
tags: [P0, linker, memory, sections]
stage: P0
---

# 05｜Linker Script 与内存布局

> [!important] 这一课不要硬啃
> 如果 Linker Script 看起来像“每一行都认识、合起来完全不懂”，按这个顺序补：
>
> 1. [[10_基础知识体系/01_计算机与MCU基础/04_地址 地址空间 与内存映射]]
> 2. [[10_基础知识体系/02_构建与链接/05_Object File 目标文件是什么]]
> 3. [[10_基础知识体系/02_构建与链接/07_Section 到底是什么]]
> 4. [[10_基础知识体系/02_构建与链接/06_Symbol 符号到底是什么]]
> 5. [[10_基础知识体系/02_构建与链接/09_Linker 到底做了什么]]
> 6. [[10_基础知识体系/02_构建与链接/10_Linker Script 为什么存在]]
> 7. [[10_基础知识体系/02_构建与链接/13_Location Counter 点号是什么]]
> 8. [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]
>
> 图解：[[10_基础知识体系/05_图解总览/02_Flash 与 SRAM 程序布局逐层图]]

> [!abstract] 本节目标
> 理解编译器得到 `.o` 后，链接器怎样回答：**代码、常量、变量、向量表最终放到 MCU 的什么地址？**

## 1. 为什么编译完还不够？

多个目标文件分别带有自己的 `.text/.data/.bss`，但此时还有问题没解决：

```text
main 最终在哪？
Reset_Handler 最终在哪？
向量表放哪？
多个 .text 怎样合并？
变量运行时放 RAM 哪里？
.data 的初始值保存在 Flash 哪里？
```

这些主要由 Linker + Linker Script 解决。

## 2. `.o` 是可重定位目标文件

`.o` 已经包含机器码、数据、section、symbol、relocation 信息，但很多引用和地址仍待最终链接。

```text
main.o
system.o
startup.o
   ↓ linker
firmware.elf
```

## 3. 当前主内存事实

```text
Flash：0x0800_0000，512 KiB
SRAM ：0x2000_0000，128 KiB
```

128 KiB = `0x20000`，所以 SRAM 顶部边界：

```text
0x20000000 + 0x20000 = 0x20020000
```

这也是当前 `_estack` 的合理值。

## 4. `MEMORY`：本固件允许使用哪些区域？

典型：

```ld
MEMORY
{
  RAM (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
  ROM (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
}
```

它不是“创建物理内存”，而是把当前固件允许使用的地址范围告诉 linker。

## 5. `SECTIONS`：各种内容分别放哪里？

概念图：

```text
Flash
├── .isr_vector
├── .text
├── .rodata
└── .data initial image

RAM
├── .data runtime copy
├── .bss
├── heap / other reserves
└── stack
```

速查：[[90_知识卡片/RAM Flash 与常见 Section]]。

## 6. Input Section 和 Output Section

每个 `.o` 都可能有：

```text
main.o .text
startup.o .text
system.o .text
```

Linker Script 可收集为最终 ELF 的 output `.text`：

```ld
.text :
{
    *(.text)
    *(.text*)
} >ROM
```

所以“`.text`”这个名字既可能指 input section，也可能指最终 output section，语境要分清。

## 7. `.`：Location Counter

例如：

```ld
.data :
{
    _sdata = .;
    *(.data*)
    _edata = .;
} >RAM
```

`.` 表示 linker 当前放置位置。

因此 `_sdata`、`_edata` 是由 linker 在不同位置建立的 symbol 值。

## 8. Linker Symbol 不是普通 C 变量

例如：

```ld
_estack = ORIGIN(RAM) + LENGTH(RAM);
```

得到：

```text
_estack = 0x20020000
```

它表示 ELF symbol 数值，不是“0x20020000 这个地址里存了一个变量 `_estack`”。

## 9. 为什么 `_estack` 取 RAM 顶部？

Cortex-M 栈通常向低地址增长，初始 SP 设为 SRAM 顶部边界很自然。

注意：

```text
0x20020000 = 顶部边界
0x2001FFFF = 最后一个有效 byte 地址
```

两者不矛盾。

## 10. `.data` 为什么有两个地址？

例如：

```c
uint32_t counter = 10;
```

运行时它必须可写，所以在 RAM；但 `10` 要随固件掉电保存，所以初始镜像在 Flash。

典型：

```text
LMA = Flash
VMA = RAM
```

Linker Script：

```ld
.data :
{
    _sdata = .;
    *(.data*)
    _edata = .;
} >RAM AT>ROM

_sidata = LOADADDR(.data);
```

形成：

```text
_sidata → Flash source
_sdata  → RAM destination start
_edata  → RAM destination end
```

Startup 负责真正复制。

## 11. `.bss` 为什么不保存整片 0？

例如：

```c
uint32_t flag;
```

C 语言要求进入程序时它为 0，但没有必要在 Flash 保存大量零字节。Linker 只在 RAM 预留空间，Startup 对 `[_sbss, _ebss)` 清零。

## 12. 当前真实输出揭示了一个重要细节

之前：

```text
_sbss = 0x20000000
_ebss = 0x2000001c
```

所以这个 `.bss` 区间只有：

```text
0x1c = 28 B
```

但 `arm-none-eabi-size` 却显示：

```text
bss = 1568
```

这不矛盾。`size` 的 bss 汇总可以包含其他 NOBITS / RAM 占用 section，例如 linker script 预留的 heap/stack 区域。

所以不能简单认为：

```text
size.bss == _ebss - _sbss
```

应继续看：

```bash
arm-none-eabi-objdump -h firmware.elf
```

这个真实例子说明：**工具输出必须结合 Linker Script 和 section table 解释。**

## 13. `KEEP()` 为什么常用于向量表？

启用 `--gc-sections` 后，linker 会丢弃看似未引用的 section。但向量表由硬件直接读取，不一定有普通软件引用，所以常见：

```ld
KEEP(*(.isr_vector))
```

防止它被垃圾回收。

## 14. `ENTRY(Reset_Handler)` 不能替代向量表

ELF 的入口信息和 Cortex-M 真实复位机制不是一回事。

真实复位：

```text
vector[0] → MSP
vector[1] → Reset_Handler
```

所以正确的向量表位置和内容仍然是关键。见 [[06_Startup 与 Reset_Handler]]。

## 15. 链接成功能证明什么？

能证明：

```text
symbol 可解析
section 可按脚本放置
没有明显 region overflow
生成了 ELF
```

不能证明：

```text
真实 MCU 型号一定匹配
向量表一定能被 CPU 正确读取
startup 已经成功执行
运行时 stack 一定安全
板子已经跑起来
```

## 16. 常见链接错误

- `region ... overflowed`：某内存区域放不下。
- `undefined reference`：symbol 没实现或没链接进来。
- `multiple definition`：同名强定义冲突。
- 更隐蔽的错误：脚本语法正确，但描述的是错误 MCU。

## 17. 离开本页前

1. `.o` 为什么还不是最终固件？
2. `MEMORY` 与 `SECTIONS` 分别解决什么？
3. input / output section 有什么区别？
4. linker symbol 为什么不是 C 变量？
5. `.data` 为什么有 VMA/LMA？
6. `.bss` 为什么不需要在 Flash 保存一份全 0？
7. 为什么 `size.bss` 不一定等于 `_ebss - _sbss`？
8. `KEEP(.isr_vector)` 为什么重要？

下一步：[[06_Startup 与 Reset_Handler]]。
