---
tags: [build, object-file]
aliases: [.o, relocatable object]
---

# Object File：`.o` 目标文件是什么？

## 一句话先说清

`.o` 是已经包含目标机器码/数据、但还没有完成整个程序最终地址布局和外部 symbol 解析的可重定位目标文件。

---

## 1. 它已经不是 C 源码

例如：

```text
main.c
→ main.c.o
```

`.o` 里已经可能有 Cortex-M4 机器码。

所以：

> 编译阶段已经做了很多工作。

---

## 2. 它为什么还不能直接烧录？

因为它只知道自己内部的局部结构。

它可能仍然不知道：

```text
main 最终在 Flash 哪
SystemInit 在哪个 object
Reset_Handler 最终地址
.data 最终在 SRAM 哪
```

多个 `.o` 还没拼在一起。

---

## 3. `.o` 内部有什么？

典型：

```text
ELF header
input sections
symbol table
relocation entries
machine code
data
debug info（可能）
```

所以 `.o` 本身通常也是 ELF 格式，只是类型是 relocatable。

---

## 4. 每个 `.o` 都可以有自己的 `.text`

例如：

```text
main.o
  .text
  .data
  .bss

system.o
  .text
  .data
  .bss

startup.o
  .isr_vector
  .text
```

Linker 再把匹配的 input sections 收集到最终 output sections。

---

## 5. `.o` 里的地址常常不是最终 MCU 地址

`objdump -h main.o` 可能看到 section address 从 0 开始或相对布局。

不要期待：

```text
0x0800....
0x2000....
```

这些绝对地址通常链接后才确定。

---

## 6. 实验：比较 `.o` 和 `.elf`

分别：

```bash
arm-none-eabi-nm main.c.o
arm-none-eabi-nm firmware.elf
```

观察：

```text
main 在 .o 中是什么状态？
最终 ELF 中地址是多少？
外部 symbol 是否仍然 U？
```

这是理解 linker 最直观的练习。

---

## 7. 关联

- [[10_基础知识体系/02_构建与链接/07_Section 到底是什么]]
- [[10_基础知识体系/02_构建与链接/06_Symbol 符号到底是什么]]
- [[10_基础知识体系/02_构建与链接/08_Relocation 重定位到底是什么]]
