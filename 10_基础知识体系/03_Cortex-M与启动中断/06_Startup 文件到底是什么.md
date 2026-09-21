---
tags: [startup, cortex-m]
aliases: [Startup]
---

# Startup 文件到底是什么？

## 一句话先说清

Startup 是“把处理器复位入口和 C 程序运行环境接起来”的低级启动代码。它通常同时定义向量表、Reset_Handler、默认异常处理器和 weak handler。

---

# 1. 它为什么经常是汇编？

因为它发生在完整 C runtime 建立之前。

它需要直接控制：

- section；
- vector words；
- SP；
- symbol；
- 数据复制；
- 分支调用。

汇编能明确表达这些低级行为。

---

# 2. Startup 不是 Linker Script

Startup：

```text
是代码/数据源文件
会被 assembler 编译成 .o
最终有机器码和 vector data
```

Linker Script：

```text
是 linker 的规则文件
不会在 MCU 上执行
```

两者通过 symbol 和 section 契约连接。

---

# 3. Startup 通常包含四块

```text
① .isr_vector
② Reset_Handler
③ Default_Handler
④ weak aliases / handler declarations
```

不同厂商/版本写法会不同，但职责类似。

---

# 4. `.isr_vector`

定义：

```text
initial MSP
Reset
NMI
HardFault
...
外部 IRQ handlers
```

详见：

[[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]

---

# 5. `Reset_Handler`

通常做：

```text
SystemInit
.data copy
.bss clear
runtime init
main
```

详见：

[[10_基础知识体系/03_Cortex-M与启动中断/07_Reset_Handler 逐段看]]

---

# 6. Default Handler

如果某个中断没有用户实现，weak alias 可能让它落到：

```text
Default_Handler
```

常见行为是：

```text
无限循环
```

方便调试时发现：

> 有一个没处理的异常进来了。

---

# 7. Startup 为什么和具体 MCU 有关？

因为向量表中的 IRQ 列表与芯片有关。

STM32F446 的外部 IRQ 布局不一定和别的 STM32F4 或其他 Cortex-M MCU 完全相同。

所以不能随便拿另一颗芯片 startup。

---

# 8. 怎样确认当前工程到底用哪份 Startup？

不要只看目录。

看详细构建：

```bash
cmake --build build --target ... -v
```

确认实际汇编输入文件。

再：

```bash
nm
objdump
```

确认结果。

---

## 9. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/07_Reset_Handler 逐段看]]
- [[10_基础知识体系/03_Cortex-M与启动中断/14_Weak Handler 为什么能被覆盖]]
- [[10_基础知识体系/02_构建与链接/04_汇编器到底做了什么]]
