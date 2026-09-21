---
tags: [cortex-m, NVIC, interrupt]
aliases: [NVIC]
---

# NVIC 到底负责什么？

## 一句话先说清

NVIC 是 Cortex-M 的 Nested Vectored Interrupt Controller。它管理外部中断等异常的 enable、pending、active、priority 和嵌套仲裁，但它不是产生所有中断事件的外设本身，也不存 handler 代码。

---

# 1. 谁产生事件？

例如 GPIO 外部中断：

```text
GPIO pin edge
→ EXTI peripheral detects
→ EXTI request
```

这是 STM32 外设侧。

NVIC 不负责检测 PB0 电平。

---

# 2. NVIC 接到请求以后做什么？

它参与管理：

```text
是否 enable
是否 pending
当前 active 状态
priority
能否抢占当前执行
```

最后由 Cortex-M exception logic 决定是否接受。

---

# 3. NVIC 不等于向量表

NVIC：

```text
谁可以进？
什么时候进？
谁优先？
```

向量表：

```text
进了以后跳到哪？
```

这两个经常被混在一起。

---

# 4. Enable 和 Peripheral Enable 也是两层

以后 EXTI 可能需要：

```text
EXTI 自己的 mask/config
+
NVIC_EnableIRQ(...)
```

只开 NVIC 不代表外设一定会产生请求。

只配置外设但 NVIC 未 enable，也可能 handler 不进入。

---

# 5. Pending

Pending 表示：

> 某异常请求正在等待处理。

可能由：

- 外设事件；
- 软件设置；
- 某些系统机制。

如果优先级条件不允许马上执行，它可以保持 pending。

---

# 6. Active

当 CPU 正在执行对应 handler 时，exception 处于 active。

某些情况下一个 exception 可同时表现 pending + active（例如处理期间又来一次请求），具体看架构和外设行为。

---

# 7. Priority

数值和“高低优先级”的关系在 Cortex-M 里容易混。

实现中通常：

```text
较小的优先级数值
→ 更高逻辑优先级
```

但有效 priority bits 数量由 MCU 实现决定，CMSIS 提供接口帮助处理。

004 NVIC 实验会系统学习。

---

# 8. Nested

“N” 是 Nested。

高优先级 exception 可以在规则允许时抢占低优先级 handler。

于是形成：

```text
main
→ IRQ A
  → IRQ B
  ←
←
```

这也是 stack 与优先级设计的重要来源。

---

# 9. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/10_异常 Exception 与中断 Interrupt 的关系]]
- [[10_基础知识体系/03_Cortex-M与启动中断/11_异常进入时 CPU 自动做了什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
