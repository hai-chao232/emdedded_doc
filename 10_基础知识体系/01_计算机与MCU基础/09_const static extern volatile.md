---
tags: [fundamentals, c, const, static, extern, volatile]
aliases: [volatile, static, extern, const]
---

# `const`、`static`、`extern`、`volatile`

## 一句话先说清

这四个关键字解决的是不同维度的问题。尤其 `volatile` 只约束编译器对访问的优化，不等于原子、不等于线程安全、不等于内存屏障。

---

# 1. `const`

```c
const uint32_t x = 10;
```

核心语义：

> 不能通过这个名字按普通方式修改对象。

在嵌入式中它常让只读数据最终放到 Flash 的 `.rodata`，但：

> `const` 是语言语义，具体 section 仍由编译器/linker 决定。

---

# 2. `static`

`static` 有两个常见用途。

## 文件作用域

```c
static int state;
```

通常表示 internal linkage：

> 名字只在当前 `.c` 文件内部链接可见。

## 函数内部

```c
void f(void)
{
    static int count;
}
```

表示对象具有 static storage duration：

> 多次调用之间保留状态。

---

# 3. `extern`

```c
extern int state;
```

表示：

> 这里是声明，不在这里提供该对象的定义。

例如：

```c
// a.c
int state;

// b.c
extern int state;
```

Linker 最终把 b.c 对 state 的引用解析到 a.c 的定义。

---

# 4. `volatile`

这是 MCU 中非常重要的关键字。

```c
volatile uint32_t *reg;
```

它告诉编译器：

> 对这个对象的读写是可观察行为，不能像普通变量那样随意消掉、合并或长期缓存。

---

# 5. 为什么外设寄存器需要 `volatile`？

硬件可能在你的 C 代码之外改变寄存器。

例如：

```text
UART 收到数据
→ 硬件设置 status bit
```

如果编译器认为：

```c
while ((STATUS & READY) == 0) {
}
```

中的 STATUS 是普通不会变化的内存，它可能做出错误优化。

`volatile` 告诉它：

> 每次需要时都应真正重新访问对象。

---

# 6. `volatile` 不保证原子性

假设：

```c
volatile uint32_t counter;
counter++;
```

大致仍可能是：

```text
read
add
write
```

如果中断同时修改 counter，仍可能出现竞争。

所以：

```text
volatile != atomic
```

---

# 7. `volatile` 不保证线程安全

它不提供：

- 锁；
- 临界区；
- 互斥；
- 原子 read-modify-write；
- 多核内存排序保证。

在 Cortex-M 单核裸机里，它常用于：

```text
MMIO
ISR 和主循环共享简单状态
```

但同步正确性还需要单独分析。

---

# 8. `volatile const`

有些寄存器：

```c
volatile const uint32_t STATUS;
```

意思：

```text
volatile
→ 硬件可能随时改变，软件每次要真实读

const
→ 软件不应该写
```

非常适合只读状态寄存器。

---

# 9. Device Header 中的修饰符

CMSIS/ST 头文件中可能看到宏：

```text
__I
__O
__IO
```

常用于表示：

```text
read-only
write-only
read/write
```

它们通常展开到 `volatile` / `const volatile` 等组合。

看头文件时不要只把它们当装饰。

---

# 10. `static volatile`

例如：

```c
static volatile bool event_pending;
```

可以同时表达：

```text
static
→ 名字/存储期的某种限制

volatile
→ 每次访问不能按普通变量优化
```

不同关键字可以同时作用于不同维度。

---

# 11. 常见误区

### `const` 一定放 Flash

不一定。语言不强制。

### `static` 就是“放静态区”

它在不同上下文还影响 linkage。

### `extern` 会创建变量

通常只是声明，不提供存储定义。

### `volatile` 能解决中断竞争

不能。

---

## 12. 关联

- [[10_基础知识体系/01_计算机与MCU基础/08_C 变量的存储期 作用域 与链接属性]]
- [[10_基础知识体系/01_计算机与MCU基础/05_MMIO 为什么寄存器像内存]]
- [[10_基础知识体系/03_Cortex-M与启动中断/15_中断与主循环共享变量为什么危险]]
