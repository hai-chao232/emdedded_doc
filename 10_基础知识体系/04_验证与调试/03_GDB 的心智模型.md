---
tags: [debug, GDB]
aliases: [GDB]
---

# GDB 的心智模型

## 一句话先说清

GDB 是调试客户端/调试器前端。它理解 ELF 的 symbol/debug info，并通过调试服务器控制目标 MCU：停下、继续、单步、读写寄存器和内存。

---

# 1. GDB 不直接等于 ST-LINK

常见链：

```text
VS Code / terminal
      ↓
arm-none-eabi-gdb
      ↓
OpenOCD / ST-LINK server
      ↓
debug probe
      ↓
SWD
      ↓
STM32
```

每层出问题，症状不同。

---

# 2. ELF 为什么对 GDB 很重要？

ELF 提供：

```text
symbol
地址
debug info
源码行号
类型
```

GDB 才能让你：

```gdb
break main
print counter
list
```

否则它只会看到裸地址和机器状态。

---

# 3. GDB 能做什么？

核心：

```text
halt
continue
step
next
breakpoint
watchpoint
read/write memory
read/write registers
evaluate symbols
```

---

# 4. `reset` 是谁实现的？

GDB 本身通常通过 remote protocol 向调试服务器发送 monitor/target 命令。

例如 OpenOCD 提供：

```text
reset halt
```

具体命令依服务器。

所以有时：

```text
GDB 命令能解析
但 OpenOCD 不支持某 monitor command
```

是不同层的问题。

---

# 5. 源码单步不是 CPU 的天然概念

CPU 只执行指令。

GDB 根据 debug info 把：

```text
instruction address
```

映射回：

```text
source line
```

所以优化构建中单步可能：

- 跳行；
- 变量不可见；
- 函数被 inline。

---

# 6. GDB 看见的内存是真实的吗？

当目标 halt 时，通过 debug access 读取的 SRAM/外设寄存器通常是真实目标状态。

但某些寄存器：

- read 有副作用；
- 会被硬件异步改变；
- 某些 peripheral 在 halt 时继续运行/暂停取决于 debug freeze 配置。

所以“Debugger 读一次”也要结合硬件语义。

---

## 7. 关联

- [[10_基础知识体系/04_验证与调试/04_OpenOCD ST-LINK SWD 各自是谁]]
- [[10_基础知识体系/04_验证与调试/05_Breakpoint Watchpoint Step 各自是什么]]
- [[10_基础知识体系/04_验证与调试/06_怎样观察寄存器 内存 PC 与 SP]]
