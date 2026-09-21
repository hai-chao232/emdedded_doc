---
tags: [cortex-m, SP, MSP, PSP]
aliases: [MSP, PSP, SP]
---

# SP、MSP、PSP 是什么？

## 一句话先说清

SP 是当前栈指针的概念/寄存器别名；Cortex-M 实际提供 MSP 和 PSP 两个栈指针。复位后默认使用 MSP，异常处理始终使用 MSP。

---

## 1. R13 是 SP

通用描述中：

```text
R13 = SP
```

但 Cortex-M 内部有两个实际 stack pointer：

```text
MSP = Main Stack Pointer
PSP = Process Stack Pointer
```

当前使用哪个，取决于处理器模式和 CONTROL 等状态。

---

## 2. 复位以后用谁？

复位后：

```text
Thread mode
使用 MSP
```

而 initial MSP 值来自向量表第 0 项。

所以：

```text
vector[0]
   ↓
MSP
```

是整个启动链最早的一步。

---

## 3. Handler mode 使用谁？

Cortex-M 进入 exception/interrupt handler 时：

```text
Handler mode 始终使用 MSP
```

即使被打断的 Thread mode 原来使用 PSP。

异常入口的栈帧保存在哪个 stack，则与异常发生前使用的 SP 有关；异常处理本身进入 Handler mode 后使用 MSP。

这是后面 RTOS 里非常关键的区别。

---

## 4. PSP 为什么存在？

它常用于：

```text
应用线程/任务栈
RTOS task
隔离 Thread mode 和 Handler mode 的栈
```

FreeRTOS 等系统会非常明显地使用 PSP/MSP 分工。

P0 先知道它存在，不需要现在实现 context switch。

---

## 5. 初始 MSP 为什么通常是 RAM 顶部？

因为 Cortex-M 常用下降栈：

```text
高地址
0x20020000 ← initial MSP
     ↓ push
0x2001FFFC
     ↓
更低地址
```

详见：

[[10_基础知识体系/01_计算机与MCU基础/10_栈 Stack 是什么]]

---

## 6. 怎样在 GDB 看？

常见：

```gdb
info registers
p/x $sp
p/x $msp
p/x $psp
```

具体 GDB target 支持命名可能有所不同。

---

## 7. 常见误区

### MSP 就是一块内存

不是。MSP 是 CPU 核心寄存器，里面保存栈地址。

### `_estack` 就是 MSP

不是。

```text
_estack = linker symbol
MSP = CPU register
```

复位时 vector[0] 通常把 `_estack` 的数值送入 MSP。

---

## 8. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
- [[10_基础知识体系/03_Cortex-M与启动中断/05_Cortex-M Reset 到底发生了什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/11_异常进入时 CPU 自动做了什么]]
