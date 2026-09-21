---
tags: [cortex-m, reset, boot]
aliases: [Reset]
---

# Cortex-M Reset 到底发生了什么？

## 一句话先说清

复位不是“直接调用 Reset_Handler”。Cortex-M 硬件首先从启动向量表位置读取 initial MSP，再读取 Reset vector；STM32 的 boot mapping 决定启动地址 0x00000000 当时映射到哪种 boot memory。

---

# 1. 架构层面的最小模型

复位时概念上：

```text
① 读取 [0x00000000] → initial MSP
② 读取 [0x00000004] → Reset vector
③ 建立核心复位状态
④ 从 Reset vector 开始执行
```

这里为了理解先用架构视角。

---

# 2. 但我们的 Flash 明明链接在 0x08000000？

对。

STM32 主 Flash 正常地址是：

```text
0x08000000
```

这就引出 boot memory alias。

---

# 3. STM32 Boot Mapping

STM32 支持不同 boot source，例如：

- Main Flash；
- System Memory bootloader；
- SRAM（具体能力/配置以芯片资料为准）。

启动时，芯片把选中的 boot memory 映射到：

```text
0x00000000
```

因此普通从 Main Flash 启动时可以概念理解：

```text
CPU reset fetch at 0x00000000
          │
          │ boot alias
          ▼
Main Flash content linked at 0x08000000
```

所以首两个 vector word 能被 CPU 取得。

---

# 4. 这和 Linker Script 有什么关系？

Linker Script 通常仍把 `.isr_vector` 放在：

```text
0x08000000
```

因为这是 Main Flash 的正常地址。

启动 alias 是芯片运行时地址映射机制，不是 linker “把同一份内容复制两份”。

---

# 5. VTOR 又在哪里出现？

复位后的 exception vector lookup base 与架构/实现的向量表基址设置有关。

ST 的 `SystemInit()` 常会设置：

```text
SCB->VTOR
```

指向实际 Flash vector table 地址，例如：

```text
0x08000000 + offset
```

这样后续异常可以直接使用新的 vector base。

详见：

[[10_基础知识体系/03_Cortex-M与启动中断/13_VTOR 向量表重定位]]

---

# 6. Reset 后 RAM 是什么状态？

不要假设：

```text
全 0
```

C runtime 所要求的状态由 startup 建立：

```text
.data copy
.bss clear
```

这正是 Reset_Handler 后半段的重要职责。

---

# 7. Reset 和 Power-on 是一回事吗？

不是完全一回事。

可能有：

- power reset；
- pin reset；
- software reset；
- watchdog reset；
- debug reset；
- 其他 reset source。

不同 reset 对外设/RAM/调试状态的影响可能不同。

P0 先不要把所有 reset 类型展开，但一定不要用：

> “复位后所有东西都像断电一样”

这种过度简化。

---

# 8. Reset_Handler 是硬件 exception handler

Reset 在 Cortex-M exception model 中占有特殊位置。

它的入口来自 vector table。

但它和普通 IRQ 仍有特殊复位语义，不能把所有中断规则机械套过来。

---

# 9. 一张流程图

```text
Reset event
    ↓
STM32 boot selection / mapping
    ↓
0x00000000 resolves to selected boot memory
    ↓
CPU reads word0
    ↓
MSP = initial stack value
    ↓
CPU reads word1
    ↓
Reset handler entry
    ↓
开始执行 Startup 中的 Reset_Handler
```

---

## 10. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
- [[10_基础知识体系/03_Cortex-M与启动中断/06_Startup 文件到底是什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/13_VTOR 向量表重定位]]
