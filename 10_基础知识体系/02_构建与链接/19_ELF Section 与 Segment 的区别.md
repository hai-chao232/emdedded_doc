---
tags: [ELF, section, segment, linker, loader]
stage: P0
---

# ELF Section 与 Segment 的区别

## 1. 为什么会同时出现两个概念

用：

```bash
readelf -S
```

你看到 Section Header。

用：

```bash
readelf -l
```

你看到 Program Header / Segment。

它们描述同一个 ELF 的不同视角。

## 2. Section：更偏链接、组织与调试

常见：

```text
.isr_vector
.text
.rodata
.data
.bss
.symtab
.debug_*
```

Section 关注：

> 不同性质的数据在 ELF 中怎样分类和组织。

链接器非常关心 section。

## 3. Segment：更偏“运行/加载需要哪些区域”

Program Header 中的 segment，尤其 `LOAD`，关注：

> 运行这个文件时，哪些文件内容应该映射/加载到哪些目标地址范围。

一个 LOAD segment 可能覆盖多个 section。

所以关系不是：

```text
一个 section = 一个 segment
```

## 4. 裸机为什么仍然值得理解 Segment

MCU 没有 Linux 那样的用户态 ELF loader，但：

- 调试器；
- 烧录工具；
- ELF 解析工具；

仍可根据 ELF 的装载信息决定哪些内容需要写入目标地址。

因此不能把：

```text
整个 ELF 文件字节流
```

等同于：

```text
需要烧入 Flash 的原始镜像
```

ELF 内部还可能有符号表、调试信息等，根本不应该烧进 MCU 运行内存。

## 5. 与 `.bss` 的关系

`.bss` 可以在运行时需要内存，却没有等量文件字节。

因此：

```text
文件大小
运行时内存大小
Flash 编程字节数
```

三者不是同一个数字。

## 6. P0 使用方法

至少运行一次：

```bash
arm-none-eabi-readelf -S firmware.elf
arm-none-eabi-readelf -l firmware.elf
```

并尝试找到：

```text
.isr_vector/.text/.data/.bss
```

与：

```text
LOAD segments
```

之间的对应关系。

不要死记 ELF 标准字段，先建立两个视角的区别。
