---
tags: [startup, SystemInit, clock]
aliases: [SystemInit]
---

# `SystemInit()` 到底处在什么位置？

## 一句话先说清

`SystemInit()` 是 ST system 文件提供的系统级初始化入口，通常由 Reset_Handler 在进入完整 C runtime 之前调用。它不是“自动把一切配置好”的万能函数，具体行为必须看当前版本源码。

---

# 1. 它在时间线上哪里？

典型：

```text
Reset
→ initial MSP
→ Reset_Handler
→ SystemInit()
→ .data copy
→ .bss clear
→ runtime init
→ main
```

这意味着：

> `SystemInit()` 运行时，普通 `.data/.bss` 初始化可能尚未完成。

---

# 2. 为什么这件事重要？

如果你修改 SystemInit 并在里面依赖：

```c
int global = 10;
```

你不能自动假设：

```text
global 已经从 Flash copy 到 SRAM
```

因为时序可能还没到。

---

# 3. 它可能做什么？

依当前 ST 文件版本，可能涉及：

- FPU access 配置；
- RCC reset/default state；
- vector table relocation；
- 系统级时钟相关初始设置；
- 其他内核/系统配置。

真正答案：

> 打开当前构建使用的 `system_stm32f4xx.c`。

---

# 4. `HSE_VALUE` 与 `SystemInit`

如果源码定义：

```text
HSE_VALUE = 8000000U
```

只表示：

> 软件在计算/配置时认为 HSE 输入是 8 MHz。

它不表示：

```text
HSE 已启用
PLL 已配好
SYSCLK 已切换
CPU 已到 180 MHz
```

---

# 5. `SystemCoreClock`

这是一个软件变量/状态表示。

它用于让库和应用知道当前核心时钟的预期值。

但：

> 软件变量本身不会驱动真实硬件时钟。

必须让 RCC 配置和变量值保持一致。

---

# 6. 为什么当前课程不在 P0 深挖 PLL？

因为早期 001～004 可以先在默认 HSI 16 MHz 下学习：

```text
GPIO
EXTI
NVIC
```

完整时钟树会在既定 Clock checkpoint 系统学习。

这是控制变量，不是忽略时钟。

---

# 7. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/07_Reset_Handler 逐段看]]
- [[10_基础知识体系/03_Cortex-M与启动中断/13_VTOR 向量表重定位]]
- [[01_P0_仓库与构建基线/04_platform_stm32f446ze 平台层]]
