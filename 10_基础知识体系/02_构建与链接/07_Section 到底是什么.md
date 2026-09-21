---
tags: [build, section, linker]
aliases: [Section]
---

# Section 到底是什么？

## 一句话先说清

Section 是目标文件/ELF 中按用途和属性组织代码或数据的逻辑容器。它不是一块独立物理芯片，但 linker 会把 output section 映射到具体地址范围。

---

## 1. 为什么需要 section？

程序内容属性不同：

```text
代码
只读常量
有初值可写数据
零初始化数据
向量表
调试信息
```

如果全部混成一团，linker 无法分别决定：

```text
谁放 Flash
谁放 SRAM
谁需要 copy
谁不进入运行镜像
```

Section 就是分类机制。

---

## 2. Input Section

每个 `.o` 可以有：

```text
.text
.rodata
.data
.bss
.isr_vector
```

这些是 input sections。

例如：

```text
main.o:.text
system.o:.text
startup.o:.text
```

---

## 3. Output Section

Linker script 可以：

```ld
.text :
{
    *(.text)
    *(.text*)
} >ROM
```

把多个 input `.text*` 收集成最终 ELF 的一个 output `.text`。

---

## 4. Section 名字本身不是绝对规则

`.text`、`.data` 是约定俗成。

你也可以自定义：

```text
.fast_code
.noinit
.my_table
```

但需要 linker script 和代码属性配合。

---

## 5. `.bss` 为什么常是 NOBITS？

它需要：

```text
运行时占地址
```

但不需要：

```text
ELF 文件里保存同样大小的一片 0 byte
```

所以 ELF section type 可以表示：

> 占内存，但没有对应文件 payload。

---

## 6. Section 与 Segment

Section 更偏：

```text
链接/组织视角
```

Program segment 更偏：

```text
加载/运行视角
```

在 MCU 初学阶段，先重点掌握 section；后面用 `readelf -l` 再理解 segment。

---

## 7. `objdump -h`

最终 ELF：

```bash
arm-none-eabi-objdump -h firmware.elf
```

可看到：

```text
name
size
VMA
LMA
file offset
flags
```

这是把 linker script 与最终结果对照起来的关键工具。

---

## 8. 关联

- [[10_基础知识体系/02_构建与链接/12_SECTIONS 命令怎样拼装最终镜像]]
- [[10_基础知识体系/02_构建与链接/14_data 与 bss 到底是什么]]
- [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]
