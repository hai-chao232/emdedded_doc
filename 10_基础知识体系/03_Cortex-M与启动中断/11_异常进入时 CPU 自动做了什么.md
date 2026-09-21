---
tags: [cortex-m, exception-entry, stack]
---

# 异常进入时 CPU 自动做了什么？

## 一句话先说清

当 Cortex-M 接受一个异常时，硬件会自动保存一组基本寄存器现场到当前栈，更新处理器状态，从向量表读取 handler 入口并开始执行。Handler 不需要自己先手动保存 PC 才能知道怎么返回。

---

# 1. 为什么需要保存现场？

假设主程序正在执行：

```text
instruction A
instruction B
instruction C
```

B 后面突然有 IRQ。

中断处理完以后，CPU 必须恢复到原来的执行上下文，否则主程序无法继续。

---

# 2. 基本异常栈帧

概念上硬件自动保存：

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

通常共 8 words：

```text
8 × 4 = 32 Bytes
```

如果 FPU 扩展上下文参与，栈帧可能更大。

---

# 3. 图

异常前：

```text
SP
 ↓
[ existing stack ... ]
```

异常接受后：

```text
低地址方向
┌────────────┐
│ R0         │
├────────────┤
│ R1         │
├────────────┤
│ R2         │
├────────────┤
│ R3         │
├────────────┤
│ R12        │
├────────────┤
│ LR         │
├────────────┤
│ PC         │
├────────────┤
│ xPSR       │
└────────────┘
       ↑
   saved frame
```

实际内存顺序/对齐规则以架构定义为准，图用于建立“硬件自动保存基本现场”的概念。

---

# 4. 然后 CPU 去哪里找 handler？

通过当前 vector table base：

```text
base + 4 * exception_number
```

取得 entry。

于是：

```text
PC ← handler entry
```

开始执行 handler。

---

# 5. LR 会变成什么？

异常进入后，LR 通常装入特殊 `EXC_RETURN` 值。

它不是普通函数返回地址。

它编码了异常返回所需的信息，例如：

- 返回 Thread/Handler；
- 使用 MSP/PSP；
- 栈帧类型。

---

# 6. 异常返回

Handler 结束时，通过符合异常返回语义的 LR/控制流，CPU 硬件会：

```text
从 stack 恢复自动保存的现场
恢复 PC/xPSR 等
继续被打断代码
```

所以中断前后能“无缝继续”。

---

# 7. 为什么 ISR 里也可能继续压栈？

硬件只保存最基本架构定义的一组。

编译器生成的 handler 函数如果要使用更多 callee-saved registers、局部变量等，仍可能额外调整 stack。

所以中断栈消耗：

```text
hardware frame
+
compiler-generated frame
+
called functions
```

---

# 8. 为什么嵌套中断增加栈压力？

如果高优先级中断打断低优先级 handler：

```text
又要保存一层现场
```

所以：

> 中断嵌套深度与 stack 预算直接相关。

---

# 9. 关联

- [[10_基础知识体系/01_计算机与MCU基础/10_栈 Stack 是什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/03_PC LR xPSR 分别是什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/12_NVIC 到底负责什么]]
