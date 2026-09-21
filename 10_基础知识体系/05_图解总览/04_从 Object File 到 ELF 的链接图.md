---
tags: [diagram, linker, object]
---

# 从 Object File 到 ELF 的链接图

```text
main.c
  ↓
main.o
┌─────────────────┐
│ .text.main      │
│ .data           │
│ .bss            │
│ symbol table    │
│ relocations     │
└─────────────────┘

system.c
  ↓
system.o
┌─────────────────┐
│ .text.SystemInit│
│ ...             │
└─────────────────┘

startup.s
  ↓
startup.o
┌─────────────────┐
│ .isr_vector     │
│ .text.Reset...  │
│ weak symbols    │
│ relocations     │
└─────────────────┘

         │
         │     linker script
         │   ┌───────────────┐
         └──►│ MEMORY        │
             │ SECTIONS      │
             │ symbols       │
             └──────┬────────┘
                    ▼

Linker:
┌────────────────────────────────┐
│ collect input sections         │
│ resolve symbols                │
│ choose final addresses         │
│ apply relocations              │
│ garbage collect unused         │
│ create output sections         │
└───────────────┬────────────────┘
                ▼

ELF
┌────────────────────────────────┐
│ .isr_vector @ Flash            │
│ .text       @ Flash            │
│ .rodata     @ Flash            │
│ .data VMA  @ SRAM              │
│ .data LMA  @ Flash             │
│ .bss       @ SRAM              │
│ symbol table                    │
│ debug info ...                  │
└────────────────────────────────┘
```

---

## 关键认识

```text
.o 中的 .text
≠
最终 ELF 中某个唯一固定地址的 .text
```

Linker 才把多个 input sections 拼成 output sections。

相关：

- [[10_基础知识体系/02_构建与链接/05_Object File 目标文件是什么]]
- [[10_基础知识体系/02_构建与链接/07_Section 到底是什么]]
- [[10_基础知识体系/02_构建与链接/08_Relocation 重定位到底是什么]]
