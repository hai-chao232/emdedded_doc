---
tags: [startup, map, boot-flow]
aliases: [上电到 main, 启动总地图]
---

# 从上电到 `main()` 的完整动态图

这是一张“迷路时回来”的总地图。

每个方框都可以点进专题。

---

# 阶段 A：固件还在 PC 上

```text
C / ASM source
      │
      ▼
[[10_基础知识体系/02_构建与链接/01_从源码到 ELF 的完整流水线]]
      │
      ▼
Object Files
      │
      ▼
[[10_基础知识体系/02_构建与链接/09_Linker 到底做了什么]]
      │
      ├─ [[10_基础知识体系/02_构建与链接/10_Linker Script 为什么存在]]
      ├─ [[10_基础知识体系/02_构建与链接/07_Section 到底是什么]]
      ├─ [[10_基础知识体系/02_构建与链接/06_Symbol 符号到底是什么]]
      └─ [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]
      │
      ▼
ELF
```

此时：

> MCU 还什么都没执行。

---

# 阶段 B：固件被烧进 Flash

Flash 概念：

```text
0x08000000
┌─────────────────────────┐
│ .isr_vector             │
│  initial MSP            │
│  Reset vector           │
│  other handlers         │
├─────────────────────────┤
│ .text                   │
├─────────────────────────┤
│ .rodata                 │
├─────────────────────────┤
│ .data initial image     │
└─────────────────────────┘
```

SRAM 此时不要先假设已经符合 C 变量初始状态。

---

# 阶段 C：Reset

```text
Reset event
   │
   ▼
STM32 boot mapping
   │
   ▼
CPU reads boot vector word0
   │
   ▼
MSP = initial stack value
   │
   ▼
CPU reads boot vector word1
   │
   ▼
execution begins at Reset_Handler
```

专题：

- [[10_基础知识体系/03_Cortex-M与启动中断/05_Cortex-M Reset 到底发生了什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
- [[10_基础知识体系/03_Cortex-M与启动中断/02_SP MSP PSP 是什么]]

---

# 阶段 D：Reset_Handler

```text
Reset_Handler
   │
   ├─ optional explicit SP reload
   │
   ├─ [[10_基础知识体系/03_Cortex-M与启动中断/09_SystemInit 到底处在什么位置]]
   │
   ├─ copy .data
   │      Flash → SRAM
   │
   ├─ clear .bss
   │      SRAM → 0
   │
   ├─ __libc_init_array
   │
   └─ main()
```

---

# 阶段 E：进入 main 前的 SRAM

假设：

```c
uint32_t counter = 10;
uint32_t flag;
```

Startup 完成：

```text
SRAM

0x20000000...
┌─────────────────────┐
│ .data               │
│ counter = 10        │
├─────────────────────┤
│ .bss                │
│ flag = 0            │
├─────────────────────┤
│ heap / reserve      │
│                     │
│                     │
│ stack grows down ↓  │
└─────────────────────┘
0x20020000 ← initial stack top area
```

---

# 阶段 F：main 运行

到这里应用才开始真正做：

```text
RCC
GPIO
Timer
DMA
...
```

所以 001 GPIO 是建立在整个 P0 启动地基之上的。

---

# 三条一定要分开的线

```text
Linker:
决定地址和 symbol

CPU hardware:
Reset/exception 时读取 vector、保存现场

Startup software:
建立 C runtime 状态
```

如果你又混乱了，先回这三句话。
