---
tags: [debug, registers, memory]
---

# 怎样观察寄存器、内存、PC 与 SP？

## 一句话先说清

调试的本质不是“让程序停住”，而是停住后用当前 PC、SP、核心寄存器、SRAM 和外设寄存器回答具体问题。

---

# 1. 看 PC

问题：

> CPU 现在执行到哪里？

GDB：

```gdb
p/x $pc
```

再结合：

```gdb
info line *$pc
```

或 disassembly。

---

# 2. 看 SP/MSP/PSP

问题：

> 栈现在在哪里？

```gdb
p/x $sp
```

并对照：

```text
0x20000000 ~ 0x20020000
```

确认在有效 SRAM。

---

# 3. 看普通寄存器

```gdb
info registers
```

Startup 汇编单步时，特别关注：

```text
r0
r1
r2
r3
sp
lr
pc
```

---

# 4. 看内存

GDB `x` 命令概念：

```gdb
x/4wx 0x20000000
```

可按 word 查看。

按 byte：

```gdb
x/16bx ADDRESS
```

学习端序时非常有帮助。

---

# 5. 看 symbol 地址

```gdb
p/x &counter
p/x &_sdata
```

把 symbol 和实际 memory dump 对起来。

---

# 6. 看外设寄存器

可以：

```gdb
p/x RCC->AHB1ENR
```

如果 debug info/headers context 可用。

也可以直接按地址读。

但牢记：

> 外设寄存器的 read 可能有副作用或动态变化。

---

# 7. 不要只截图

真正有用的实验记录应保存：

```text
当时停在哪？
为什么停？
预期是什么？
实际 PC/SP/内存是什么？
结论是什么？
```

而不是只有一张 IDE 截图。

---

## 8. 关联

- [[10_基础知识体系/04_验证与调试/03_GDB 的心智模型]]
- [[10_基础知识体系/04_验证与调试/12_实验记录应该保存什么证据]]
