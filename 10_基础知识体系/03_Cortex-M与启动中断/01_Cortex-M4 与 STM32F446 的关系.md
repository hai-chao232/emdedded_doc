---
tags: [cortex-m, stm32, cpu]
---

# Cortex-M4 与 STM32F446 的关系

## 一句话先说清

Cortex-M4 是 ARM 设计的处理器核心架构；STM32F446 是 ST 把 Cortex-M4 核心、Flash、SRAM、GPIO、RCC、TIM、USART 等外围模块集成到一颗 MCU 里的具体芯片。

---

## 1. 不要把 Cortex-M4 和 STM32F446 当同一个东西

可以画成：

```text
STM32F446 MCU
┌─────────────────────────────────┐
│ Cortex-M4 CPU Core              │
│  ├─ R0-R15 / xPSR               │
│  ├─ NVIC                        │
│  ├─ SysTick                     │
│  └─ SCB                         │
│                                 │
│ ST 外设                         │
│  ├─ RCC                         │
│  ├─ GPIO                        │
│  ├─ TIM                         │
│  ├─ DMA                         │
│  ├─ ADC                         │
│  ├─ USART                       │
│  └─ ...                         │
│                                 │
│ Flash / SRAM / Bus Matrix       │
└─────────────────────────────────┘
```

---

## 2. 谁定义 NVIC？

NVIC 是 Cortex-M 架构的一部分。

所以：

> 不只是 STM32 有 NVIC。

不同厂商的 Cortex-M MCU 都会以架构规定的方式提供 NVIC。

---

## 3. 谁定义 GPIOB？

GPIOB 是 STM32 这颗具体 MCU 的外设。

所以：

```text
GPIOB_MODER
RCC_AHB1ENR
```

主要查 STM32 Reference Manual。

---

## 4. 谁定义 Reset 的基本行为？

Cortex-M 架构规定：

```text
复位时从 vector table 取得 initial MSP
再取得 Reset vector
```

ST 再决定：

```text
启动时 0x00000000 映射到哪种 boot memory
Flash 实际地址
boot pin / option bytes 等
```

两层共同构成 STM32 的实际启动。

---

## 5. 为什么资料要分层？

问题：

> EXTI0 中断为什么最后能进入 `EXTI0_IRQHandler`？

需要同时看：

```text
STM32 EXTI
→ 产生中断请求

NVIC
→ 使能 / 优先级 / pending

Cortex-M exception mechanism
→ 自动入栈、查 vector、跳 handler

Startup
→ vector table 里提供 EXTI0 handler entry
```

这正说明“MCU 外设”和“CPU 内核机制”不能只学一边。

---

## 6. CMSIS 为什么位于两者之间？

CMSIS-Core 给出 Cortex-M 核心统一访问接口。

ST Device Header 再描述具体 STM32 外设和 IRQn。

所以源码里会自然连接：

```text
ARM core definitions
+
ST device definitions
```

---

## 7. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/10_异常 Exception 与中断 Interrupt 的关系]]
- [[10_基础知识体系/03_Cortex-M与启动中断/12_NVIC 到底负责什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
