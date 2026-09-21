---
tags: [fundamentals, stack]
aliases: [Stack]
---

# 栈 Stack 是什么？

## 一句话先说清

栈是一段按后进先出方式使用的运行时内存。CPU 用 SP 指向当前栈位置；函数调用、局部临时数据、保存寄存器和异常现场都可能消耗栈。

---

# 1. 先不要把栈想成一个特殊芯片

栈通常只是 SRAM 中的一段区域。

例如：

```text
SRAM
0x20000000
┌─────────────────────┐
│ .data               │
├─────────────────────┤
│ .bss                │
├─────────────────────┤
│ free / heap         │
│                     │
│                     │
│ stack               │
└─────────────────────┘
0x20020000
```

真正让这段 RAM “成为栈”的，是：

```text
SP
+
push/pop / load/store 的使用方式
```

---

# 2. 为什么初始 SP 放在 SRAM 顶部？

Cortex-M 常见 full-descending stack。

概念：

```text
高地址
0x20020000  ← initial SP
     │
     │ push
     ▼
0x2001FFFC
     │
     ▼
更低地址
```

所以从顶部开始向低地址增长，可以利用下方 RAM。

---

# 3. `_estack` 是什么？

Linker script 常定义：

```ld
_estack = ORIGIN(RAM) + LENGTH(RAM);
```

它给出一个 linker symbol：

```text
_estack = 0x20020000
```

Startup / vector table 使用它作为初始 MSP。

它不是：

> SRAM 顶部存了一个 C 变量叫 `_estack`。

---

# 4. 函数调用为什么可能用栈？

假设：

```c
foo(a, b);
```

调用过程中可能需要保存：

- 返回信息；
- callee-saved registers；
- 局部变量；
- 临时值；
- 超出参数寄存器数量的参数。

具体多少由 ABI、编译器和优化决定。

---

# 5. 局部变量一定在栈吗？

不是。

```c
int x = 1;
```

编译器可能：

```text
放寄存器
放栈
直接常量折叠
完全删除
```

所以：

> “automatic variable 通常与 stack 有关”可以；
> “所有局部变量都在 stack”不准确。

---

# 6. 中断为什么也用栈？

Cortex-M 进入异常时，硬件会自动保存一组现场，例如概念上：

```text
R0
R1
R2
R3
R12
LR
PC
xPSR
```

压入当前使用的栈。

这样中断结束后，CPU 才能恢复原程序继续执行。

所以：

> 中断嵌套会增加栈压力。

---

# 7. 栈溢出是什么？

如果 stack 一直向下增长，最终碰到：

```text
.bss
heap
其他数据
```

就可能破坏内存。

裸机里这种错误可能表现得非常诡异：

```text
偶发 HardFault
变量莫名改变
返回地址损坏
程序跑飞
```

Linker “预留了栈空间”也不代表运行时绝不会超。

---

# 8. Linker Script 里的 stack reserve

有些 ST linker script 会有：

```text
_Min_Stack_Size
_user_heap_stack
```

它可以帮助链接时保留一块 RAM、检查最小空间。

但它不是完整的运行时栈分析。

---

# 9. 怎样观察栈？

Debugger 里可以看：

```text
MSP
PSP
SP
```

也可以：

- 在函数中单步观察 SP；
- 触发中断前后观察 SP；
- 用 fill pattern 检查 high-water mark；
- 使用 RTOS 自带 stack watermark。

当前 P0 先做到：

> 能解释 `_estack`、MSP 和“向低地址增长”。

---

# 10. 常见误区

### 栈是 linker 创建的一个特殊内存

不是。它本质还是 SRAM。

### `_estack` 指向最后一个有效字节

更精确地说，它常是 RAM 顶部边界，用作初始 SP。

### 栈只和函数有关

异常/中断也大量使用栈。

---

## 11. 关联

- [[10_基础知识体系/01_计算机与MCU基础/11_函数调用 栈帧 LR 与返回]]
- [[10_基础知识体系/03_Cortex-M与启动中断/02_SP MSP PSP 是什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/11_异常进入时 CPU 自动做了什么]]
