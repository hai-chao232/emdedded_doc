---
tags: [diagram, vector-table, reset]
---

# 向量表与 Reset 取址图

## 1. 最终 Flash 中

假设当前固件：

```text
Main Flash
0x08000000
┌───────────────────────────────┐
│ word0: 0x20020000             │ ← initial MSP
├───────────────────────────────┤
│ word1: Reset_Handler entry    │
├───────────────────────────────┤
│ word2: NMI_Handler entry      │
├───────────────────────────────┤
│ word3: HardFault_Handler      │
├───────────────────────────────┤
│ ...                           │
└───────────────────────────────┘
```

---

# 2. Reset 时 STM32 boot mapping

```text
CPU wants
0x00000000
      │
      │ STM32 normal Flash boot alias
      ▼
Main Flash content
0x08000000
```

---

# 3. 第一次读取

```text
CPU
 │ read vector word0
 ▼
0x00000000 alias
 │
 ▼
0x08000000 content
 │
 ▼
0x20020000
 │
 ▼
MSP = 0x20020000
```

---

# 4. 第二次读取

```text
CPU
 │ read vector word1
 ▼
0x00000004 alias
 │
 ▼
0x08000004 content
 │
 ▼
Reset_Handler entry
 │
 ▼
开始执行 Reset_Handler
```

---

# 5. 后续 VTOR

SystemInit / software 可能把：

```text
SCB->VTOR = 0x08000000
```

于是运行期普通 exception：

```text
exception number N
       │
       ▼
address = VTOR + 4*N
       │
       ▼
read handler entry
       │
       ▼
handler
```

---

# 6. 三个地址不要混

```text
0x00000000
→ reset boot alias / initial vector lookup address

0x08000000
→ Main Flash 实际映射、固件 link address

SCB->VTOR
→ 运行期 exception vector base
```

在正常简单工程里它们最终指向同一套 vector 内容，但概念并不相同。
