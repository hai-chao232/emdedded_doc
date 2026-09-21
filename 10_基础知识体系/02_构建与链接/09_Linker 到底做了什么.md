---
tags: [build, linker]
aliases: [Linker]
---

# Linker 到底做了什么？

## 一句话先说清

Linker 把多个 object/library 组合成一个整体程序，解析跨文件 symbol，决定 section 最终布局，应用 relocation，并生成最终 ELF。

---

## 1. Linker 的输入

典型：

```text
main.o
startup.o
system.o
libc.a
libgcc.a
...
linker script
```

---

## 2. Linker 的四个核心任务

### ① 收集

把 input sections 收进 output sections。

### ② 解析 symbol

例如：

```text
main.o 需要 SystemInit
system.o 提供 SystemInit
```

将引用与定义连接起来。

### ③ 布局

决定：

```text
.isr_vector 在哪
.text 在哪
.data VMA 在哪
.bss 在哪
```

### ④ Relocation

根据最终地址修正机器码/数据中的地址引用。

---

## 3. Linker 不负责什么？

它不负责：

- CPU reset 后真的执行代码；
- 把 `.data` 从 Flash 复制到 SRAM；
- 把 `.bss` 清零；
- 打开 GPIO 时钟；
- 烧录 MCU。

它只生成一个“已经安排好”的程序映像和元数据。

---

## 4. 为什么需要 Linker Script？

通用桌面程序可由系统默认 linker script 处理大量事情。

裸机 MCU 没有操作系统 loader 帮你安排地址。

你必须明确告诉 linker：

```text
Flash 从哪开始
SRAM 从哪开始
向量表放哪
哪些 section 放哪
.data 的运行地址和加载地址
```

这就是：

[[10_基础知识体系/02_构建与链接/10_Linker Script 为什么存在]]

---

## 5. Linker 和 CMake 的关系

CMake 负责把：

```text
-Tpath/to/script.ld
```

等参数送进最终链接命令。

真正解释 `.ld` 的是 GNU linker（通常由 `arm-none-eabi-gcc` driver 调用）。

---

## 6. 关联

- [[10_基础知识体系/02_构建与链接/10_Linker Script 为什么存在]]
- [[10_基础知识体系/02_构建与链接/08_Relocation 重定位到底是什么]]
- [[10_基础知识体系/02_构建与链接/17_ELF BIN HEX MAP 分别是什么]]
