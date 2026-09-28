---
tags: [P0, linker, linker-script, memory-layout]
stage: P0
---

# 05｜Linker Script 与内存布局

> [!tip] 深挖入口
> - [[10_基础知识体系/02_构建与链接/07_Section 到底是什么]]
> - [[10_基础知识体系/02_构建与链接/06_Symbol 符号到底是什么]]
> - [[10_基础知识体系/02_构建与链接/13_Location Counter 点号是什么]]
> - [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]
> - [[10_基础知识体系/02_构建与链接/19_ELF Section 与 Segment 的区别]]
> - [[10_基础知识体系/02_构建与链接/20_NOBITS NOLOAD 与为什么RAM占用不等于Flash占用]]
> - [[10_基础知识体系/02_构建与链接/21_Alignment Padding 与地址为什么会出现空洞]]
> - [[10_基础知识体系/01_计算机与MCU基础/04_地址 地址空间 与内存映射]]
> - [[10_基础知识体系/01_计算机与MCU基础/06_Flash SRAM ROM RAM]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/02_SP MSP PSP 是什么]]

> [!abstract] 本页目标
> 本页只负责串起“链接器如何把若干 `.o` 组织成一个可运行镜像”。
> 遇到局部概念时，优先跳转到基础知识页，不在本页无限展开。
>
> **本页用的是教学简化脚本。** 你工程里实际在用的那份（STM32CubeIDE 生成、15+ 个段）见
> [[10_基础知识体系/02_构建与链接/23_真实链接脚本逐段注解]]。
> 两页对照着看，才知道哪些结构是本质、哪些是厂商生成的样板。

## 1. 这节课解决什么问题？

编译器把每个 `.c` 变成 `.o` 以后，仍然没有一个完整的 MCU 程序。此时还需要回答：

- 哪些代码放进 Flash？
- 哪些变量最终在 RAM？
- 向量表为什么必须在启动地址附近？
- `.data` 为什么同时与 Flash、RAM 都有关？
- `.bss` 为什么占 RAM 却几乎不需要在 Flash 中保存一串零？
- `_estack/_sidata/_sdata/_edata/_sbss/_ebss` 是谁产生的？

这些由**链接阶段**统一解决。

## 2. 从 `.o` 到最终 ELF

```text
main.c ──编译──> main.o ─┐
startup.s ─────> startup.o├─ Linker + Linker Script ─> ELF
system.c ──────> system.o ┤
其他库对象 ───────────────┘
```

每个 `.o` 内部已有自己的输入 section，例如：

```text
.text
.rodata
.data
.bss
.isr_vector
```

链接器把多个输入 section 归并、排序、分配地址，形成最终输出 section。

## 3. MEMORY 回答“物理区域在哪里”

典型裸机脚本会先描述可用地址区域：

```ld
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
    RAM   (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
}
```

这不是在“创建 Flash/RAM”。

它只是告诉链接器：

> 当前目标芯片允许你把哪些内容安排在哪些地址范围内。

真实地址范围来自芯片 Datasheet / Reference Manual / Cortex-M 内存映射。

深入：
- [[10_基础知识体系/01_计算机与MCU基础/04_地址 地址空间 与内存映射]]
- [[10_基础知识体系/01_计算机与MCU基础/06_Flash SRAM ROM RAM]]
- [[10_基础知识体系/02_构建与链接/11_MEMORY 命令到底在描述什么]]

## 4. SECTIONS 回答“各类内容放哪里”

示意：

```ld
SECTIONS
{
    .isr_vector :
    {
        KEEP(*(.isr_vector))
    } > FLASH

    .text :
    {
        *(.text*)
        *(.rodata*)
    } > FLASH

    .data :
    {
        _sdata = .;
        *(.data*)
        _edata = .;
    } > RAM AT> FLASH

    .bss (NOLOAD) :
    {
        _sbss = .;
        *(.bss*)
        *(COMMON)
        _ebss = .;
    } > RAM
}
```

核心不是语法，而是回答：

```text
.isr_vector → Flash
.text       → Flash
.rodata     → Flash
.data       → 运行时在 RAM；初始值镜像在 Flash
.bss        → 运行时在 RAM；启动时清零
```

深入：
- [[10_基础知识体系/02_构建与链接/12_SECTIONS 命令怎样拼装最终镜像]]

## 5. Location Counter `.` 是什么

链接脚本中的 `.` 可以粗略理解为：

> 当前正在安排的输出地址位置。

例如：

```ld
_sdata = .;
```

不是分配一个 C 变量。

而是：

> 定义一个名为 `_sdata` 的 linker symbol，其值等于当前位置。

因此：

```text
_sdata
```

本质上先是“一个数值/地址符号”，不是“RAM 中又额外占了 4 字节”。

深入：
- [[10_基础知识体系/02_构建与链接/13_Location Counter 点号是什么]]

## 6. `.data` 为什么最容易混乱

假设：

```c
uint32_t counter = 10;
```

程序烧录前，初始值 `10` 必须有地方永久保存，否则掉电就消失。

因此：

```text
Flash
┌─────────────────────────────┐
│ .data 初始化镜像：counter=10 │
└─────────────────────────────┘
             │ Reset_Handler copy
             ▼
RAM
┌─────────────────────────────┐
│ counter 的运行实体 = 10      │
└─────────────────────────────┘
```

这就是：

- **LMA**（Load Memory Address）：初始化镜像加载位置，通常在 Flash。
- **VMA**（Virtual Memory Address）：程序运行时访问它的位置，通常在 RAM。

> [!warning] 关键不是背缩写，而是分清“存在哪”和“运行在哪”
> 同一个 `counter` 同时出现在两个地址区间：Flash 里是**初始值的一份拷贝**，RAM 里才是**程序运行时真正读写的那份**。
> 两个地址都对，问“counter 到底在哪”时必须先问“你问的是哪一个”。

深入：
- [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]

常见 linker symbols：

```text
_sidata → Flash 中 .data 初始化镜像起始地址
_sdata  → RAM 中 .data 运行区起始地址
_edata  → RAM 中 .data 运行区结束地址
```

它们共同给 Reset_Handler 一个搬运范围：`[_sidata, ...)` → `[_sdata, _edata)`。

注意 `_sidata` 只标出**起点**：源区长度由 `_edata - _sdata` 决定，而不是由另一个 `_eidata` 标出。

## 7. `.bss` 为什么不同

例如：

```c
uint32_t flag;
```

C 语言要求具有静态存储期、未显式初始化的对象初值为 0。

最浪费的做法是：

```text
Flash 里真的存几千个 0
```

更合理的做法：

```text
ELF 只描述：
“这段 RAM 运行时需要 N 字节，并且启动时清零”
```

因此 `.bss` 常表现为 `SHT_NOBITS`，或者在 linker script 中以 `NOLOAD` 形式描述。

深入：
- [[10_基础知识体系/02_构建与链接/20_NOBITS NOLOAD 与为什么RAM占用不等于Flash占用]]

常见符号：

```text
_sbss → RAM .bss 起始
_ebss → RAM .bss 结束
```

Reset_Handler 遍历 `[_sbss, _ebss)` 清零。

## 8. `_estack` 为什么常等于 RAM 顶端

示意：

```ld
_estack = ORIGIN(RAM) + LENGTH(RAM);
```

当前工程 RAM 区域是：

```text
0x20000000 ~ 0x2001FFFF
```

那么“第一个越过 RAM 的地址”是：

```text
0x20000000 + 0x20000 = 0x20020000
```

栈向低地址增长时，可以把初始 MSP 设置到这里。

> [!note] 边界地址不等于属于这块 RAM 的字节
> `0x20020000` 是“刚好越过末端”的地址。把它作为初始 MSP 是合法的，因为栈是**先减后写**：第一次压栈才访问 `0x2001FFFC`，仍在 RAM 内。

深入：
- [[10_基础知识体系/03_Cortex-M与启动中断/02_SP MSP PSP 是什么]]

## 9. `ALIGN()` 与 padding

链接器不能总把下一个对象紧贴上一个对象。

如果一个 section 或对象要求 4/8/16 字节对齐，中间可能出现 padding：

```text
对象 A 结束
    ↓
[padding]
    ↓
对象 B 的合法对齐地址
```

因此看到地址“跳了几个字节”不一定是丢失空间。

```ld
. = ALIGN(4);
```

可以读成：把当前位置向上推进到下一个 4 字节对齐的地址。

深入：
- [[10_基础知识体系/02_构建与链接/21_Alignment Padding 与地址为什么会出现空洞]]

## 10. `KEEP()` 为什么常用于向量表

启用：

```text
-ffunction-sections
-fdata-sections
-Wl,--gc-sections
```

后，链接器可能回收看似“没有普通代码引用”的 section。

向量表恰恰经常是由硬件读取，而不是通过普通 C 调用关系引用。

因此：

```ld
KEEP(*(.isr_vector))
```

是在告诉 linker：

> 即使普通引用分析看不到它，也不能丢掉。

深入：
- [[10_基础知识体系/02_构建与链接/16_KEEP gc-sections 与为什么向量表不能被删]]

## 11. Linker Script 与 Startup 的接口

Linker Script 负责“给出地址与边界”：

```text
_estack
_sidata
_sdata
_edata
_sbss
_ebss
```

Startup / Reset_Handler 负责“在运行时使用这些地址”：

```text
_sidata → _sdata ... _edata
_sbss   → 清零 ... _ebss
```

它们不是两个互不相干的文件，而是一对**生产者 / 消费者**。

这也是为什么真正理解启动链时，05 与 06 必须来回对照。

## 12. 静态验证

推荐至少交叉看：

```bash
arm-none-eabi-size xxx.elf
arm-none-eabi-objdump -h xxx.elf
arm-none-eabi-nm -n xxx.elf
arm-none-eabi-readelf -h -S -l -s xxx.elf
```

分别回答：

```text
size     → 大致 ROM/RAM 规模
objdump  → section 地址、大小
nm       → symbol 值与类型
readelf  → ELF / section / program header / symbol 的结构化事实
```

## 13. 离开本页前

闭卷回答：

1. `.data` 为什么同时关联 Flash 和 RAM？
2. `.bss` 为什么占 RAM，却不需要在 Flash 中保存同等大小的零？
3. `_sidata/_sdata/_edata` 是“变量”还是“地址符号”？
4. `_estack` 为什么常等于 RAM 顶端边界？为什么这样设置是安全的？
5. `MEMORY` 与 `SECTIONS` 分别解决什么问题？
6. `KEEP(.isr_vector)` 为什么有意义？
7. VMA 和 LMA 分别回答什么问题？
8. 看到两个 symbol 地址之间有空洞，先怀疑什么？

下一步：[[01_P0_仓库与构建基线/06_Startup 与 Reset_Handler]]
