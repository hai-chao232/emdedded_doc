---
tags: [cortex-m, PC, LR, xPSR]
aliases: [PC, LR, xPSR]
---

# PC、LR、xPSR 分别是什么？

## 一句话先说清

PC 决定控制流执行到哪里；LR 保存普通函数返回或异常返回相关信息；xPSR 保存条件码、Thumb 状态、异常号等程序状态。

---

# 1. PC：Program Counter

```text
R15 = PC
```

CPU 通过 PC 决定从哪里取指令。

所以：

```text
Reset_Handler 地址大于 main
```

不意味着 main 先执行。

真正流程：

```text
Reset vector
→ PC = Reset_Handler
→ Reset_Handler 执行
→ bl main
→ PC 跳 main
```

---

# 2. LR：Link Register

```text
R14 = LR
```

普通函数调用时，它常与返回地址相关。

异常处理中，它会被赋予特殊 `EXC_RETURN` 值。

因此在 HardFault 调试时：

```text
LR 是普通函数返回地址？
还是 EXC_RETURN？
```

很重要。

---

# 3. xPSR

xPSR 可以看成几个状态寄存器视图的组合：

```text
APSR
IPSR
EPSR
```

包含：

- 算术条件标志；
- 当前 exception number；
- Thumb 状态等。

---

# 4. 为什么异常栈帧会保存 PC/LR/xPSR？

因为中断执行结束后，CPU 需要恢复：

```text
原来执行到哪里
原来的返回链状态
原来的程序状态
```

所以基本异常栈帧包含它们。

---

# 5. Thumb 状态

Cortex-M 只执行 Thumb/Thumb-2 指令集状态。

向量表中的 handler address 最低位与 Thumb 状态入口语义有关。

因此你在向量表 raw word 中常看到：

```text
函数 symbol 地址 + 1
```

或最低位为 1 的入口值。

不要把这个最低位误认为实际指令地址是奇数 byte 在随便执行。

---

# 6. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/05_Cortex-M Reset 到底发生了什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
- [[10_基础知识体系/03_Cortex-M与启动中断/11_异常进入时 CPU 自动做了什么]]
