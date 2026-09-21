---
tags: [build, linker-script]
aliases: [Linker Script]
---

# Linker Script 为什么存在？

## 一句话先说清

Linker Script 是“最终程序怎样占用目标地址空间”的规则书。它把 object file 中的逻辑 section 映射到真实 MCU 的 Flash/SRAM 地址，并定义 startup 需要的关键 symbol。

---

## 1. 裸机为什么特别依赖它？

桌面 Linux 程序启动时有：

```text
OS loader
virtual memory
dynamic linker
```

MCU 裸机没有这一层替你安排。

复位后硬件直接从内存映射中取地址。

因此固件必须在链接时就把很多地址定死。

---

## 2. Linker Script 主要回答哪些问题？

```text
有哪些可用 memory region？
每个 region 起始地址和长度？
.isr_vector 放哪？
.text/.rodata 放哪？
.data 运行时放哪、初值从哪加载？
.bss 放哪？
stack top 是多少？
需要定义哪些 linker symbols？
```

---

## 3. 它不是“程序运行时脚本”

`.ld` 不会被 STM32 CPU 执行。

它在 PC 上由 linker 读取。

最后它的效果体现在：

```text
ELF section address
symbol value
program image
```

---

## 4. 当前常见结构

```ld
ENTRY(Reset_Handler)

_estack = ORIGIN(RAM) + LENGTH(RAM);

MEMORY
{
  RAM (xrw) : ORIGIN = ..., LENGTH = ...
  ROM (rx)  : ORIGIN = ..., LENGTH = ...
}

SECTIONS
{
  .isr_vector : { ... } >ROM
  .text       : { ... } >ROM
  .data       : { ... } >RAM AT>ROM
  .bss        : { ... } >RAM
}
```

每一块都可以单独深入：

- [[10_基础知识体系/02_构建与链接/11_MEMORY 命令到底在描述什么]]
- [[10_基础知识体系/02_构建与链接/12_SECTIONS 命令怎样拼装最终镜像]]
- [[10_基础知识体系/02_构建与链接/13_Location Counter 点号是什么]]
- [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]

---

## 5. Linker Script 和芯片 Datasheet 的关系

Datasheet / Reference Manual 告诉你物理/逻辑内存事实：

```text
Flash 地址和容量
SRAM 地址和容量
```

Linker Script 把这些事实写成 linker 能理解的规则。

所以 `.ld` 如果写错，Linker 不会自动从芯片“读出真相”。

---

## 6. 一个危险点：写错也可能链接成功

如果你误写：

```text
RAM = 256K
```

但真实 MCU 只有 128K，而程序只用了前 20K：

```text
链接仍可能成功
```

这说明：

> Linker Script 只是你告诉 linker 的模型，不是硬件真值自动验证器。

---

## 7. 关联

- [[10_基础知识体系/02_构建与链接/11_MEMORY 命令到底在描述什么]]
- [[10_基础知识体系/02_构建与链接/12_SECTIONS 命令怎样拼装最终镜像]]
- [[10_基础知识体系/01_计算机与MCU基础/04_地址 地址空间 与内存映射]]
