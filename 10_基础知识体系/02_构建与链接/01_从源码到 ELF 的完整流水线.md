---
tags: [build, pipeline]
---

# 从源码到 ELF 的完整流水线

## 一句话先说清

一个 `.c` 文件不会直接“变成 ELF”。典型过程是：预处理 → 编译 → 汇编 → 目标文件；多个目标文件和库再由 linker 合并成最终 ELF。

---

## 1. 总图

```text
main.c
  │
  │ preprocessor
  ▼
main.i
  │
  │ compiler
  ▼
main.s
  │
  │ assembler
  ▼
main.o
  │
  │
startup.s ──assembler──> startup.o
system.c  ─────────────> system.o
  │
  ├──────────────┐
  │              │
  ▼              ▼
object files + libraries
        │
        │ linker + linker script
        ▼
       ELF
```

现实中 GCC driver 可能把中间步骤隐藏掉，但逻辑仍然存在。

---

## 2. 为什么要分这么多阶段？

因为每一阶段解决的问题不同：

```text
Preprocess
→ 宏、include、条件编译

Compile
→ C 语义变成目标架构汇编/中间代码

Assemble
→ 汇编变成机器码 + relocatable object

Link
→ 多个 object 合并、解析 symbol、安排最终地址
```

---

## 3. CMake/Ninja 在哪里？

它们在流水线外层组织“谁调用谁”。

```text
CMake
→ 生成构建规则

Ninja
→ 根据依赖图调用 arm-none-eabi-gcc 等工具

ARM GNU
→ 真正完成预处理/编译/汇编/链接
```

所以：

> CMake 不是编译器，Ninja 也不是编译器。

---

## 4. 为什么 `.o` 还不是最终地址？

单独编译 `main.c` 时，编译器不知道：

- startup 最终多大；
- `.text` 前面还有谁；
- `main` 最终是 0x08000250 还是别的；
- 外部函数定义来自哪个 object；
- `.data` 最终在 SRAM 哪里。

所以 `.o` 里会保留 symbol、section、relocation 等信息，等 linker 决定最终布局。

---

## 5. 为什么 linker script 是链接阶段的输入？

因为 linker 本身只知道：

> 我有一堆 section 和 symbol，要合并。

但它不知道这颗 STM32 的：

```text
Flash 起点
Flash 大小
SRAM 起点
SRAM 大小
向量表应放哪
.data 要 RAM 运行、Flash 加载
```

这些策略由 linker script 提供。

---

## 6. 最终 ELF 比机器码更丰富

ELF 里可能有：

```text
最终 machine code
section table
program headers
symbol table
debug info
地址
各种 metadata
```

因此 ELF 不只是“可执行内容”，还是调试和分析的核心载体。

---

## 7. 关联

- [[10_基础知识体系/02_构建与链接/02_预处理到底做了什么]]
- [[10_基础知识体系/02_构建与链接/03_编译器到底做了什么]]
- [[10_基础知识体系/02_构建与链接/04_汇编器到底做了什么]]
- [[10_基础知识体系/02_构建与链接/05_Object File 目标文件是什么]]
- [[10_基础知识体系/02_构建与链接/09_Linker 到底做了什么]]
