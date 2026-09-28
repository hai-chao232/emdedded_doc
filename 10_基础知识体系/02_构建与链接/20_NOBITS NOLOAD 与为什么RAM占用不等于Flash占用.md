---
tags: [bss, NOBITS, NOLOAD, ELF, RAM, Flash]
stage: P0
---

# NOBITS、NOLOAD 与为什么 RAM 占用不等于 Flash 占用

## 1. 一个反直觉现象

假设：

```c
uint8_t buffer[4096];
```

如果它属于 `.bss`，运行时显然需要 4096 B RAM。

但我们没有必要在 Flash 中额外保存：

```text
4096 个 0
```

因为 Reset_Handler 可以启动时清零。

所以：

```text
运行时需要 N 字节
```

不等于：

```text
ELF 文件中必须有 N 个对应字节
```

也不等于：

```text
Flash 必须额外烧 N 个零
```

## 2. `SHT_NOBITS`

ELF section 类型 `NOBITS` 的直觉是：

> 这个 section 有运行时地址和大小，但在文件里不需要为其内容保存同等大小的数据字节。

典型例子就是 `.bss`。

## 3. `NOLOAD`

GNU ld 脚本中可能写：

```ld
.bss (NOLOAD) :
{
    ...
} > RAM
```

它表达“这段输出 section 不作为普通装载内容处理”的意图。

不要把 `NOLOAD` 与 `NOBITS` 当作完全同一个层级的概念：

```text
NOLOAD → linker script 输出 section 属性/装载意图
NOBITS → ELF section 类型
```

实际生成结果应通过 `readelf -S -l` 验证。

## 4. 为什么 `size` 容易让初学者误判

`size` 的 `bss` 列描述的是一类运行时存储占用统计。

看到：

```text
bss = 4096
```

不能直接说：

> “固件 Flash 又增加了 4096 字节。”

应同时看：

```bash
objdump -h
readelf -S
readelf -l
```

## 5. 栈/堆预留也是同样思路

有些工程会在 linker script 里预留栈/堆空间。

这些空间可能表现为：

```text
RAM 地址范围被保留
```

但不是说：

```text
Flash 里必须存一份同尺寸初始字节
```

> [!example] 真实工程里的样子
> `embedded_lab` 的链接脚本里有一个 `._user_heap_stack` 输出段，它**不含任何 `*(...)`**，
> 只是用 location counter 往前推进了 `_Min_Heap_Size + _Min_Stack_Size`。
>
> 它在 ELF 里长这样：
> ```text
> ._user_heap_stack  Size 00000604  VMA 2000001c  ALLOC
> ```
> **还有 `Size`，但文件里一个字节都没有。** 这就是 NOBITS 类段的本质。
>
> 详见 [[10_基础知识体系/02_构建与链接/23_真实链接脚本逐段注解]]。

### 最干净的证明：`FileSiz` vs `MemSiz`

想证明"占 RAM 不占 Flash"，**不要用 `objdump -h`**——它不显示这两个字段，还会给你一个误导性的 LMA。

用 program header：

```bash
arm-none-eabi-readelf -l build/.../exp001_gpio_output.elf
```

```text
Type  Offset   VirtAddr   PhysAddr   FileSiz  MemSiz   Flg
LOAD  0x001000 0x08000000 0x08000000 0x00334  0x00334  R E
LOAD  0x001000 0x20000000 0x08000334 0x00000  0x0001c  RW
LOAD  0x00001c 0x2000001c 0x08000334 0x00000  0x00604  RW
                                       ↑        ↑
                                    文件里    运行时
                                    0 字节    占这些
```

```text
.bss              FileSiz 0  MemSiz 0x1c  (= 28)
._user_heap_stack FileSiz 0  MemSiz 0x604 (= 1540)
```

**`FileSiz = 0` 就是"不占 Flash"的直接证据。**
这是同一个事实在 ELF 里最不含糊的表达。

> [!warning] `objdump -h` 会在这里误导你
> 它给这两个段显示的 LMA 是 `0x08000334`——一个 **Flash 地址**。
> 这不表示它们要被加载到 Flash；是因为它们没有文件内容，链接器只是顺手填了所属 segment 的起始地址。
> 详见 [[10_基础知识体系/03_Cortex-M与启动中断/20_Reset_Handler 反汇编逐条对照]] 第 10.2 节。

## 6. 对启动链的意义

`.bss` 能省 Flash 的前提是：

```text
启动代码负责建立“全零”的运行时状态
```

所以 `.bss` clear 不是无关紧要的样板代码，而是 C 运行环境契约的一部分。
