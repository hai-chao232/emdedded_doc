---
tags: [debug, breakpoint, watchpoint]
---

# Breakpoint、Watchpoint、Step 各自是什么？

## 一句话先说清

Breakpoint 按“执行到某地址”停；Watchpoint 按“某数据地址被访问/修改”停；Step 控制执行粒度。它们对应不同调试问题。

---

# 1. Breakpoint

例如：

```gdb
break main
```

意思：

> 当 PC 到达 main 对应地址时停下。

---

# 2. 软件断点 vs 硬件断点

在 Flash 上不能像 RAM 那样随便修改指令插入 trap，所以 MCU 调试常依赖 Cortex-M 硬件 breakpoint 资源（FPB）。

数量有限。

VS Code/GDB 可能替你自动选择。

---

# 3. Watchpoint

例如：

```gdb
watch counter
```

目标：

> counter 被写时停。

Cortex-M 可用 DWT 等硬件资源实现数据 watchpoint。

资源也有限。

---

# 4. 为什么 watchpoint 特别适合查“谁改了变量”？

如果：

```text
某变量偶尔变成奇怪值
```

你不一定知道是哪段代码写的。

Watchpoint 直接在发生写入时停住，比全局搜代码更接近真实执行证据。

---

# 5. `step` 与 `next`

### step

进入函数内部。

### next

通常把函数调用当一行执行过去。

但它们都依赖 debug info/source mapping。

---

# 6. Instruction Step

如果要看 startup 汇编：

```text
source-line step
```

有时太粗。

应使用 instruction-level single step，逐条看：

```text
PC
register
memory
```

---

# 7. 关联

- [[10_基础知识体系/04_验证与调试/06_怎样观察寄存器 内存 PC 与 SP]]
- [[10_基础知识体系/04_验证与调试/08_怎样验证 data copy 与 bss clear]]
