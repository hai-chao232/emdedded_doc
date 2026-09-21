---
tags: [cortex-m, exception, interrupt]
aliases: [Exception, Interrupt]
---

# 异常 Exception 与中断 Interrupt 的关系

## 一句话先说清

在 Cortex-M 语境中，Exception 是总称。它包括 Reset、NMI、HardFault、SysTick 等系统异常，也包括来自外设的 IRQ。日常口语里的“中断”通常主要指外部 IRQ，但硬件处理模型统一在 exception 机制中。

---

# 1. 为什么术语容易混乱？

中文里经常把：

```text
exception
interrupt
IRQ
handler
ISR
```

都宽泛叫“中断”。

但 Cortex-M 文档有更严格层次。

---

# 2. Exception Number

向量表就是按 exception number 排列。

概念：

```text
0  initial MSP（特殊，不是 exception handler）
1  Reset
2  NMI
3  HardFault
...
15 SysTick
16 IRQ0
17 IRQ1
...
```

因此外部 IRQ 在统一 exception number 空间里排在系统 exceptions 后面。

---

# 3. IRQn_Type

CMSIS/ST 常定义：

```text
SysTick_IRQn = 负值
...
WWDG_IRQn = 0
...
```

负值用于系统 exception 的枚举表达。

外部 IRQ 从 0 开始编号。

不要把：

```text
IRQn = 0
```

误认为：

> 向量表第 0 项。

实际外部 IRQ0 对应 exception number 16。

---

# 4. 外设中断链

以后 EXTI 实验会看到：

```text
GPIO edge
→ EXTI detects
→ EXTI pending
→ IRQ line to NVIC
→ NVIC pending/priority
→ CPU accepts exception
→ vector table handler lookup
→ handler
```

这里同时有：

```text
STM32 peripheral mechanism
+
Cortex-M exception mechanism
```

---

# 5. Fault 也是 Exception

例如：

```text
HardFault
MemManage
BusFault
UsageFault
```

它们不是外设 IRQ，但进入 handler 的底层机制仍属于 exception 系统。

---

# 6. ISR / Handler

ISR：

```text
Interrupt Service Routine
```

Handler：

```text
Exception Handler
```

在 Cortex-M 代码中常以：

```c
void EXTI0_IRQHandler(void)
```

这样的函数实现。

但异常进入/退出由 CPU 硬件执行特殊流程，不只是普通 C 函数 call。

---

# 7. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/11_异常进入时 CPU 自动做了什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/12_NVIC 到底负责什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
