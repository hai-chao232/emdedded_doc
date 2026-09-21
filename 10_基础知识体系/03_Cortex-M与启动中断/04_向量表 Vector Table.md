---
tags: [cortex-m, vector-table, interrupt, reset]
aliases: [Vector Table, 向量表]
---

# 向量表 Vector Table

## 一句话先说清

向量表是一张按 exception number 排列的“入口地址表”。Cortex-M 硬件在复位和异常发生时直接读取它：第 0 项提供初始 MSP，第 1 项提供 Reset Handler，后续项提供各种 exception/IRQ handler 入口。

---

# 1. 它到底是什么？

先不要神化。

它本质上是一段连续的 32-bit word 数据。

概念：

```text
index  address offset   content

0      +0x00            Initial MSP value
1      +0x04            Reset_Handler entry
2      +0x08            NMI_Handler entry
3      +0x0C            HardFault_Handler entry
...
15     +0x3C            SysTick_Handler entry
16     +0x40            IRQ0 Handler entry
17     +0x44            IRQ1 Handler entry
...
```

所以它叫“表”，因为：

> exception number 可以索引到一个固定表项。

---

# 2. 谁使用它？

最关键答案：

> **Cortex-M CPU 硬件使用。**

不是：

- CMake；
- Linker；
- Startup C 代码；
- GDB。

它们可以创建、放置、观察向量表，但真正发生复位/异常跳转时，是处理器硬件根据架构规则读取。

---

# 3. 谁创建它？

通常由 Startup 汇编文件定义内容，例如概念：

```asm
.section .isr_vector
.word _estack
.word Reset_Handler
.word NMI_Handler
.word HardFault_Handler
...
.word EXTI0_IRQHandler
```

所以：

```text
Startup source
→ 定义“表里有哪些 entry”
```

---

# 4. 谁决定它最终放在哪？

Linker Script。

例如：

```ld
.isr_vector :
{
    KEEP(*(.isr_vector))
} >ROM
```

所以：

```text
Startup
→ 负责内容

Linker Script
→ 负责最终地址和保留

CPU
→ 运行时使用
```

这三个角色一定要分开。

---

# 5. 它在 STM32F446 Flash 中通常长什么样？

如果主 Flash 固件被链接到：

```text
0x08000000
```

那么最终 `.isr_vector` 通常在这里开始：

```text
0x08000000  initial MSP word
0x08000004  Reset vector word
0x08000008  NMI vector
0x0800000C  HardFault vector
...
```

但 Cortex-M reset 架构上从启动向量位置读取，STM32 还存在 boot memory alias，下一节会细讲。

---

# 6. 为什么第 0 项不是 handler？

因为复位后 CPU 首先需要一个可用栈。

所以 table entry 0 是：

```text
initial MSP
```

不是异常 handler 地址。

这也是 Cortex-M 向量表与某些架构直觉不同的地方。

---

# 7. 第 1 项有什么效果？

CPU 取得它以后，建立复位入口控制流。

概念：

```text
vector[1]
   ↓
Reset_Handler entry
   ↓
PC / execution starts there
```

所以 Reset_Handler 不是因为：

```ld
ENTRY(Reset_Handler)
```

才被真实 MCU reset 找到。

关键是：

```text
向量表第 1 项
```

---

# 8. 中断时怎么用？

假设某个外部 IRQ 的 exception number 对应表项 N。

当异常被接受：

```text
外设产生 IRQ
→ NVIC / exception logic 接受
→ CPU 自动保存现场
→ CPU 从 VectorTableBase + 4*N 读取 handler entry
→ 进入 handler
```

所以中断 handler 不是“操作系统查函数名”。

硬件只认：

```text
表项中的入口值
```

---

# 9. IRQn 与向量表下标

Cortex-M 系统 exceptions 占前面的 exception numbers。

外部中断从：

```text
exception number 16
```

开始。

ST 的 `IRQn_Type` 常对外部 IRQ 使用：

```text
0, 1, 2, ...
```

因此粗略关系：

```text
vector index = IRQn + 16
```

但负值 IRQn 常用于表示系统 exceptions（如 SysTick 等），所以实际编码应以 CMSIS 定义理解，不要手工魔法数字滥算。

---

# 10. Thumb bit 为什么会看到最低位为 1？

Cortex-M handler 必须进入 Thumb state。

向量表中的异常入口值最低位承载相关状态语义。

因此如果：

```text
Reset_Handler symbol = 0x08000278
```

原始 vector word 可能表现为：

```text
0x08000279
```

CPU 使用入口时会按架构规则处理最低位。

所以：

> 看 raw vector bytes 时，handler entry 不一定和 `nm` symbol address 数值完全相同。

---

# 11. `KEEP()` 为什么重要？

CPU 硬件会使用向量表，但普通代码里不一定有函数“引用”整个表。

启用 `--gc-sections` 后，linker 可能把“看起来没有普通软件引用”的 section 视为可丢弃。

所以 linker script 常写：

```ld
KEEP(*(.isr_vector))
```

详见：

[[10_基础知识体系/02_构建与链接/16_KEEP gc-sections 与为什么向量表不能被删]]

---

# 12. 怎样从 ELF 直接看？

Section：

```bash
arm-none-eabi-objdump -h firmware.elf
```

Raw bytes：

```bash
arm-none-eabi-objdump -s -j .isr_vector firmware.elf
```

Symbols：

```bash
arm-none-eabi-nm -n firmware.elf
```

把：

```text
word0
word1
...
```

和：

```text
_estack
Reset_Handler
```

一一对照。

---

# 13. 如果向量表错了会怎样？

例如：

### Initial MSP 错

可能一进入代码/异常就使用非法 stack，迅速 fault。

### Reset vector 错

CPU 可能跳到错误地址，导致 fault/lockup/不可预测行为。

### 某 IRQ entry 错

对应中断一旦发生，可能进入错误 handler 或 fault。

### VTOR 指向错误表

所有后续 exception vector lookup 可能错误。

---

# 14. 向量表不是 NVIC

NVIC 管：

```text
enable
pending
priority
active
```

向量表提供：

```text
handler entry
```

二者配合。

---

# 15. 向量表不是“中断函数数组”

从 C 视角这样类比有帮助，但不完全准确：

- entry 0 不是函数；
- entry 内容有架构状态语义；
- 是硬件 exception 机制的一部分；
- 位置/对齐受架构约束。

---

# 16. 一张总图

```text
Startup source
    │
    │ .word _estack
    │ .word Reset_Handler
    │ .word ...
    ▼
input section .isr_vector
    │
    │ Linker Script:
    │ KEEP(*(.isr_vector)) > ROM
    ▼
ELF / Flash
0x08000000
┌──────────────────────────┐
│ initial MSP              │
├──────────────────────────┤
│ Reset_Handler entry      │
├──────────────────────────┤
│ NMI_Handler entry        │
├──────────────────────────┤
│ HardFault_Handler entry  │
├──────────────────────────┤
│ ...                      │
└──────────────────────────┘
    │
    │ CPU hardware reads
    ▼
Reset / Exception control flow
```

---

## 17. 下一步

必须继续看：

[[10_基础知识体系/03_Cortex-M与启动中断/05_Cortex-M Reset 到底发生了什么]]

然后再看：

[[10_基础知识体系/03_Cortex-M与启动中断/13_VTOR 向量表重定位]]
