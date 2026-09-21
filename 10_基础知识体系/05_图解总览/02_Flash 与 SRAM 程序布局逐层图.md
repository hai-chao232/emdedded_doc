---
tags: [diagram, flash, sram, sections]
---

# Flash 与 SRAM 程序布局逐层图

## 图 A：只有物理区域

```text
FLASH                               SRAM
0x08000000                          0x20000000
┌───────────────┐                   ┌───────────────┐
│               │                   │               │
│               │                   │               │
│               │                   │               │
└───────────────┘                   └───────────────┘
0x08080000                          0x20020000
```

此时还没有 `.text/.data` 的概念。

---

# 图 B：Linker 开始放 Output Sections

```text
FLASH
0x08000000
┌────────────────────────┐
│ .isr_vector            │
├────────────────────────┤
│ .text                  │
├────────────────────────┤
│ .rodata                │
├────────────────────────┤
│ .data load image       │
├────────────────────────┤
│ free Flash ...         │
└────────────────────────┘


SRAM
0x20000000
┌────────────────────────┐
│ .data runtime          │
├────────────────────────┤
│ .bss                   │
├────────────────────────┤
│ heap/stack reserve ... │
│                        │
│ free/runtime           │
│                        │
│ stack grows down       │
└────────────────────────┘
0x20020000
```

---

# 图 C：为什么 `.data` 出现在两边？

不是复制了两个独立变量。

```text
Flash:
initial image

        startup copy
             │
             ▼

SRAM:
runtime object
```

同一个语言对象在不同阶段需要不同存储角色。

---

# 图 D：`.bss` 为什么 Flash 没大块 payload？

```text
ELF metadata:
“SRAM 这里要占 N bytes”

Reset_Handler:
“把这 N bytes 清 0”
```

而不是：

```text
Flash 真的保存 N 个 00
```

---

# 图 E：Stack 的“区域”和 SP

```text
SRAM high
0x20020000  ← _estack value / initial MSP
    │
    │ push
    ▼
┌──────────────┐
│ stack frame  │
├──────────────┤
│ stack frame  │
└──────────────┘
    │
    ▼ growth toward lower address
```

Linker 可能预留最小 stack 空间，但真实运行使用量由程序行为决定。

---

# 图 F：Heap 与 Stack 不要死记“必然相向增长”

很多 linker script/裸机 runtime 常画：

```text
heap ↑
free
stack ↓
```

这很有帮助，但不是 C 语言标准要求的唯一布局。

当前工程应以实际 linker script/runtime 为准。
