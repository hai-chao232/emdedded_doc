---
tags: [fundamentals, heap, malloc]
aliases: [Heap]
---

# 堆 Heap 是什么？

## 一句话先说清

Heap 是程序运行时用于动态内存分配的一块内存管理区域。它不是“所有剩余 RAM”的自动同义词，也不是裸机程序必须使用的东西。

---

# 1. 动态分配

例如：

```c
void *p = malloc(128);
```

程序运行到这里才请求：

```text
128 Bytes
```

这与：

```c
static uint8_t buffer[128];
```

这种编译/链接时已确定大小的对象不同。

---

# 2. Heap 通常也来自 SRAM

概念：

```text
SRAM
├── .data
├── .bss
├── heap ↑
│
│ free
│
└── stack ↓
```

但真正布局由 linker/runtime 策略决定。

---

# 3. 为什么嵌入式早期课程尽量不用 Heap？

动态分配会带来：

- 分配失败；
- 碎片；
- 生命周期管理；
- 非确定性；
- 额外 runtime 支持。

在实时/资源受限系统中，需要非常明确地决定是否使用。

因此早期外设实验更适合：

```text
静态 buffer
明确容量
无动态分配
```

---

# 4. Linker 预留 Heap ≠ 程序正在用 malloc

ST linker script 可能有：

```text
_Min_Heap_Size
```

甚至 `_user_heap_stack` section。

这只是：

> 为潜在 heap/stack 使用预留空间、参与链接检查。

不能由此推出程序已经调用了 `malloc()`。

---

# 5. `_sbrk` 为什么经常出现？

Newlib 的 `malloc` 等功能需要底层知道：

> heap 可以向哪里扩展？

裸机工程常通过 `_sbrk` 提供这种机制。

如果程序/库拉入需要它的功能，而你没有实现或提供 nosys stub，可能出现链接问题。

---

# 6. 动态内存和 RTOS

以后 FreeRTOS 会引入自己的 heap 实现方案：

```text
heap_1
heap_2
heap_4
...
```

那时“heap”可能是 RTOS 管理的一块数组/区域，不一定等同于 C library 的 `malloc` heap。

---

# 7. 关联

- [[10_基础知识体系/01_计算机与MCU基础/06_Flash SRAM ROM RAM]]
- [[10_基础知识体系/01_计算机与MCU基础/10_栈 Stack 是什么]]
- [[10_基础知识体系/02_构建与链接/18_C Library Newlib Nano 与裸机 syscall]]
