---
tags: [fundamentals, cpu, register, instruction]
---

# CPU、寄存器、指令与执行

## 一句话先说清

CPU 不是“直接执行 C 语言”。编译器把 C 变成机器指令，CPU 按 PC 指向的位置取指令，并在寄存器、内存和运算单元之间不断搬运与计算。

---

## 1. 从 C 到 CPU

你写：

```c
x = x + 1;
```

CPU 看不到这行 C。

大致会变成类似：

```text
从内存把 x 读进寄存器
寄存器 +1
把结果写回内存
```

最终是真正的机器指令编码。

---

## 2. 为什么 CPU 需要寄存器？

寄存器在 CPU 核心内部，访问非常直接。

如果每次简单加法都直接在 SRAM 上完成，会受到更多访问限制和延迟。

因此典型过程：

```text
Memory
  ↓ load
Register
  ↓ ALU operation
Register
  ↓ store
Memory
```

---

## 3. Cortex-M4 有哪些你现在最需要认识的寄存器？

先不背全部，只记：

```text
R0-R12   通用寄存器
R13      SP
R14      LR
R15      PC
xPSR     程序状态
```

另外 SP 这个名字背后还涉及：

```text
MSP
PSP
```

后面见：

[[10_基础知识体系/03_Cortex-M与启动中断/02_SP MSP PSP 是什么]]

---

## 4. PC：程序计数器

PC 可以粗略理解为：

> CPU 当前 / 下一步取指令的位置。

如果程序正常执行：

```text
instruction A
instruction B
instruction C
```

PC 会随控制流变化。

遇到：

```text
branch
function call
exception
```

PC 会跳到别处。

因此：

> “函数地址较小”不代表“函数先执行”。

CPU 执行谁，由 PC 的控制流决定。

这正是为什么：

```text
main 地址 < Reset_Handler 地址
```

也不代表复位先进入 main。

---

## 5. LR：Link Register

函数调用时，CPU 需要记住：

> 函数执行完以后回哪里？

ARM 中常使用 LR 保存返回相关信息。

普通函数调用概念：

```text
caller
  ↓ call
callee
  ↓ return
caller 的下一条
```

后面异常处理中，LR 还可能保存特殊的 `EXC_RETURN` 值，而不是普通代码地址。

---

## 6. SP：Stack Pointer

SP 指向当前栈位置。

函数调用、局部变量、保存寄存器、异常入栈都可能使用栈。

Cortex-M 常见栈向低地址增长：

```text
高地址
  │
  │ 初始 SP
  ▼
[push]
  ▼
更低地址
```

详见：

[[10_基础知识体系/01_计算机与MCU基础/10_栈 Stack 是什么]]

---

## 7. Load / Store 思维

Cortex-M 属于 load/store 风格架构。

大量运算围绕：

```text
Load:
Memory → Register

Compute:
Register → ALU → Register

Store:
Register → Memory
```

这会帮助你理解 startup 里的汇编：

```asm
ldr r0, =_sdata
ldr r3, [r2, r4]
str r3, [r0, r4]
```

不用一开始背指令，只要先看懂“地址”和“数据搬运”。

---

## 8. CPU 与外设寄存器

STM32 的 GPIO/RCC 也是地址空间中的地址。

所以 CPU 访问：

```c
GPIOB->MODER
```

本质上也会变成某种：

```text
load/store 到外设地址
```

这叫 Memory-Mapped I/O：

[[10_基础知识体系/01_计算机与MCU基础/05_MMIO 为什么寄存器像内存]]

---

## 9. CPU 并不知道“变量名”

例如：

```c
uint32_t counter;
```

`counter` 是源代码层名字。

链接后它可能对应：

```text
0x20000020
```

CPU 运行时主要关心：

```text
地址
寄存器
指令
值
```

Symbol 主要帮助 linker/debugger 和我们理解程序。

---

## 10. 一条指令从哪里来？

粗略：

```text
C source
  ↓ compiler
assembly / machine code
  ↓ assembler
object code
  ↓ linker
final address
  ↓ Flash
CPU fetch
```

这就是为什么构建系统与 CPU 启动过程能连成一条链。

---

## 11. 常见误区

### CPU “执行一个函数”

更底层地说：

> PC 跳到某段指令地址，然后逐条执行。

### 寄存器就是 STM32 外设寄存器

“寄存器”有两个常见层次：

```text
CPU core register
R0/SP/LR/PC

Peripheral register
GPIO_MODER/RCC_AHB1ENR
```

都叫 register，但位置和作用完全不同。

---

## 12. 关联

- [[10_基础知识体系/01_计算机与MCU基础/11_函数调用 栈帧 LR 与返回]]
- [[10_基础知识体系/01_计算机与MCU基础/04_地址 地址空间 与内存映射]]
- [[10_基础知识体系/01_计算机与MCU基础/05_MMIO 为什么寄存器像内存]]
- [[10_基础知识体系/03_Cortex-M与启动中断/01_Cortex-M4 与 STM32F446 的关系]]
