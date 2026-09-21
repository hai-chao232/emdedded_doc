---
tags: [fundamentals, MMIO, register]
aliases: [Memory Mapped IO]
---

# MMIO：为什么外设寄存器看起来像内存？

## 一句话先说清

STM32 把外设控制寄存器放进 CPU 的地址空间。CPU 用普通 load/store 指令访问这些地址，芯片内部总线把访问送到 RCC、GPIO、USART 等外设。

---

## 1. 先看一个普通 RAM 访问

```c
uint32_t *p = (uint32_t *)0x20000000;
uint32_t x = *p;
```

概念：

```text
CPU 发出地址 0x20000000
        ↓
SRAM 响应
        ↓
返回数据
```

---

## 2. 再看一个外设寄存器访问

假设某寄存器地址：

```text
0x40023830
```

CPU 访问：

```text
CPU 发出地址 0x40023830
        ↓
总线地址译码
        ↓
RCC 外设响应
        ↓
读取/修改硬件状态
```

从 CPU 指令角度看，仍然是：

```text
load/store
```

但被访问的不是普通 RAM 单元，而是硬件寄存器。

---

## 3. 为什么这样设计很方便？

如果外设完全使用另一套特殊指令，编译器和软件模型会复杂很多。

Memory-Mapped I/O 让软件可以用统一地址访问：

```text
RAM
Flash
GPIO
RCC
NVIC
...
```

当然不同区域的语义完全不同。

---

## 4. Device Header 做了什么？

我们一般不会手写：

```c
*(volatile uint32_t *)0x40023830
```

而会写：

```c
RCC->AHB1ENR
```

Device Header 大致通过：

```text
基地址宏
+
C struct
+
volatile
```

把硬件地址包装成可读的 C 表达式。

概念示例：

```c
typedef struct {
    volatile uint32_t CR;
    volatile uint32_t PLLCFGR;
    ...
    volatile uint32_t AHB1ENR;
} RCC_TypeDef;

#define RCC_BASE 0x40023800UL
#define RCC ((RCC_TypeDef *)RCC_BASE)
```

于是：

```c
RCC->AHB1ENR
```

等价于：

> 从 `RCC_BASE + AHB1ENR offset` 访问一个 32 bit volatile 寄存器。

---

## 5. 为什么是 `volatile`？

外设寄存器可能：

- 被硬件异步改变；
- 每次读有硬件意义；
- 每次写都有副作用；
- 不能让编译器把访问随意消掉或缓存成普通变量。

因此寄存器声明通常带：

```c
volatile
```

更详细见：

[[10_基础知识体系/01_计算机与MCU基础/09_const static extern volatile]]

---

## 6. 外设寄存器不是普通变量

普通 RAM：

```c
x = 1;
x = 2;
```

编译器可能发现第一次写永远没人看到，从而优化掉。

但某个外设：

```c
REG = 1;
REG = 2;
```

两次写可能都触发真实硬件动作。

所以不能按普通变量语义随便优化。

---

## 7. Read-Modify-Write 为什么可能有问题？

经典：

```c
REG |= MASK;
```

大致是：

```text
read REG
modify in CPU
write REG back
```

如果：

- 硬件在读写之间改变某些 bit；
- 某些 bit 写 1 清除；
- 某些 bit 读有副作用；

就可能出问题。

因此硬件经常提供专门寄存器，例如 GPIO `BSRR`：

```text
写某些 bit → set
写另一些 bit → reset
```

避免普通读-改-写。

---

## 8. “寄存器地址”有两种常见含义

### CPU Core Registers

```text
R0-R15
xPSR
```

它们在 CPU 内部，不是普通 MMIO 地址。

### Peripheral/System Registers

```text
GPIO_MODER
RCC_AHB1ENR
SCB->VTOR
NVIC...
```

很多通过 memory-mapped address 访问。

不要把两者混成一种。

---

## 9. 谁决定外设地址？

不是 C 头文件决定的。

真正顺序是：

```text
芯片硬件设计
    ↓
Datasheet / RM 记录
    ↓
Device Header 用宏/struct 表达
    ↓
你的 C 代码访问
```

所以如果头文件有 bug，硬件不会跟着改变。

---

## 10. 001 GPIO 会怎样用到？

后面点灯：

```text
RCC_AHB1ENR
GPIOB_MODER
GPIOB_BSRR
```

你真正做的是：

```text
CPU load/store
  ↓
对 MMIO 地址访问
  ↓
RCC / GPIO 硬件改变状态
  ↓
引脚电平改变
```

这条链比“调用一个 API 点灯”更重要。

---

## 11. 关联

- [[10_基础知识体系/01_计算机与MCU基础/07_指针到底是什么]]
- [[10_基础知识体系/01_计算机与MCU基础/09_const static extern volatile]]
- [[10_基础知识体系/01_计算机与MCU基础/04_地址 地址空间 与内存映射]]
