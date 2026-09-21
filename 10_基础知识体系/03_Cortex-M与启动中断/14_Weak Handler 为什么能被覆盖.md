---
tags: [startup, weak, linker, interrupt]
aliases: [Weak Handler]
---

# Weak Handler 为什么能被覆盖？

## 一句话先说清

Startup 可以给每个 IRQ 提供 weak 默认定义；当你的 C 文件提供同名 strong handler 时，linker 的 symbol resolution 会优先采用 strong definition，于是向量表最终指向你的实现。

---

# 1. 默认情况

Startup 可能概念上：

```asm
.weak EXTI0_IRQHandler
.thumb_set EXTI0_IRQHandler, Default_Handler
```

表示：

```text
如果没人提供更强定义
→ EXTI0_IRQHandler 使用 Default_Handler
```

---

# 2. 用户代码

```c
void EXTI0_IRQHandler(void)
{
    ...
}
```

这是 strong definition。

Linker 解析后：

```text
EXTI0_IRQHandler
→ 用户的函数
```

向量表中的 symbol relocation 最终也指向用户函数地址。

---

# 3. 这不是运行时“替换”

不是 MCU 启动后检查：

```text
有没有用户函数？
```

替换发生在链接时。

最终 ELF 里已经决定了地址。

---

# 4. 名字为什么必须完全匹配？

如果写错：

```c
void EXTI_0_IRQHandler(void)
```

而 startup 要的是：

```text
EXTI0_IRQHandler
```

Linker 看成两个不同 symbol。

结果：

> 中断仍可能进入 Default_Handler。

---

# 5. `W` 是什么？

`nm` 输出：

```text
W Some_Handler
```

表示 weak symbol。

不表示 warning。

---

# 6. 关联

- [[10_基础知识体系/02_构建与链接/06_Symbol 符号到底是什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/06_Startup 文件到底是什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/17_一次外部中断发生时的完整流程]]
