---
tags: [cortex-m, VTOR, vector-table]
aliases: [VTOR]
---

# VTOR：向量表重定位

## 一句话先说清

VTOR 是 Cortex-M System Control Block 中的 Vector Table Offset Register。它让软件为后续异常指定向量表基址，因此向量表不必永远使用复位时的 boot alias 地址。

---

# 1. 为什么需要重定位？

启动时，STM32 可能通过 boot alias 让：

```text
0x00000000
```

映射到 Main Flash。

但程序正常运行后，我们更希望明确告诉 CPU：

```text
实际向量表就在 0x08000000
```

于是设置：

```text
SCB->VTOR = 0x08000000
```

概念上如此，具体对齐和偏移要求按 Cortex-M 实现。

---

# 2. Boot alias 和 VTOR 不是一回事

### Boot alias

STM32 芯片的启动映射机制。

解决：

> Reset 最初从 0x00000000 读取时，那里对应哪种 boot memory？

### VTOR

Cortex-M 核心寄存器。

解决：

> 后续 exception lookup 使用哪一个 vector table base？

---

# 3. Bootloader 为什么特别需要 VTOR？

假设：

```text
Bootloader vectors: 0x08000000
Application vectors: 0x08010000
```

Bootloader 跳到 application 前，应用通常需要让：

```text
VTOR → 0x08010000
```

否则应用运行时中断仍可能跳进 bootloader 的 vector table。

---

# 4. 向量表为什么要对齐？

硬件需要通过：

```text
base + 4 * exception_number
```

高效索引，同时 VTOR 低位存在架构规定用途/保留，因此 vector base 有对齐要求。

具体对齐与实现支持的 interrupt 数量等有关，查对应 Cortex-M4 文档/CMSIS，而不要随便选地址。

---

# 5. `SystemInit()` 里的 VTOR

ST system 文件常有类似逻辑：

```text
SCB->VTOR = FLASH_BASE | VECT_TAB_OFFSET
```

所以：

> 当前 VTOR 到底是多少，要看实际 `SystemInit()` 源码和编译宏。

---

# 6. 怎样验证？

Debugger：

```text
读取 SCB->VTOR
```

然后：

```text
把它和 .isr_vector VMA 对照
```

若不一致，需要明确解释是 boot alias、运行阶段还是配置错误。

---

# 7. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
- [[10_基础知识体系/03_Cortex-M与启动中断/05_Cortex-M Reset 到底发生了什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/09_SystemInit 到底处在什么位置]]
