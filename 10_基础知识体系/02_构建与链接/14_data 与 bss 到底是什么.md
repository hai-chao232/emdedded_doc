---
tags: [linker, data, bss, c-runtime]
aliases: [.data, .bss]
---

# `.data` 与 `.bss` 到底是什么？

## 一句话先说清

它们首先是“程序静态数据的分类/section 约定”。`.data` 常保存需要非零初值的可写静态对象；`.bss` 常保存需要零初始化的静态对象。最终它们都通常在 SRAM 中有运行时空间，但初始化方式不同。

---

## 1. `.data`

典型：

```c
uint32_t counter = 10;
static int mode = 2;
```

需求：

```text
运行时可写
+
进入 main 前已有指定初值
```

所以：

```text
Flash 保存 initial image
Startup copy
SRAM 运行
```

---

## 2. `.bss`

典型：

```c
uint32_t flag;
static uint8_t buffer[128];
```

C 语言要求 static storage duration 对象在没有显式初始化时为 0。

所以：

```text
Linker 在 SRAM 预留
Startup clear to zero
```

无需 Flash 保存 128 个零。

---

## 3. `.bss` 名字历史上是什么？

名字来自早期汇编器历史：

```text
Block Started by Symbol
```

今天不需要靠这个名字理解其语义，知道它代表常见零初始化/NOBITS 数据区域即可。

---

## 4. `int x = 0;` 一定在 `.bss` 吗？

编译器常会把显式初始化为 0 的静态对象也优化到 `.bss`，因为效果相同且更省镜像空间。

但具体 section placement 是工具链实现选择。

不要把 C 源码写法和固定 section 做绝对一一映射。

---

## 5. `const` 一定是 `.rodata` 吗？

也不能绝对化。

常见是 `.rodata`，但编译器可基于使用方式、优化、合并常量等做不同安排。

真正结果看 ELF。

---

## 6. 为什么它们不是“RAM 类型”？

`.data` `.bss` 是 section 概念。

SRAM 是硬件 memory region。

Linker 决定：

```text
这些 output sections 最终映射到 SRAM
```

两层不要混淆。

---

## 7. 当前最小 ELF 的教育意义

之前：

```text
_sdata = _edata
```

说明当前 `.data` 长度为 0。

但：

```text
_sbss < _ebss
```

说明有 `.bss` 区域。

这比只背定义更重要：你已经看到真实固件的结果。

---

## 8. 关联

- [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]
- [[10_基础知识体系/03_Cortex-M与启动中断/08_data copy 与 bss clear 为什么由 Startup 做]]
