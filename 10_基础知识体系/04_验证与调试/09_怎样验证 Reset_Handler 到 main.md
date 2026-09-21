---
tags: [debug, reset-handler, main]
---

# 怎样验证 `Reset_Handler → main()`？

## 目标命题

> 真正烧录到板上的 CPU reset 后，确实先经过当前 ELF 的 Reset_Handler，并最终到达 main。

---

# 1. 静态证据

先看：

```bash
arm-none-eabi-nm -n "$ELF" \
 | grep -E 'Reset_Handler|main'
```

再：

```bash
arm-none-eabi-objdump -d "$ELF"
```

确认 Reset_Handler 反汇编中存在到：

```text
SystemInit
__libc_init_array
main
```

等预期控制流。

---

# 2. 动态断点

设置：

```text
break Reset_Handler
break main
```

然后真正执行：

```text
reset halt
continue
```

首先应停 Reset_Handler。

继续后停 main。

---

# 3. 同时观察 SP

在 Reset_Handler：

```text
SP 应位于有效 SRAM
```

在 main：

```text
SP 仍应合理
```

这把：

```text
向量表 initial MSP
```

也纳入验证。

---

# 4. 观察 VTOR

main 前后读：

```text
SCB->VTOR
```

记录当前 vector base。

---

# 5. 不要只使用“break main”

只在 main 断住只能证明：

> 最终到了 main。

如果要学习启动流程，应主动在：

```text
Reset_Handler
SystemInit
data copy
bss clear
main
```

几个节点观察。

---

# 6. 如果 reset 后直接停 main？

可能是 IDE/debugger 配置自动：

```text
runToEntryPoint = main
```

或类似设置。

这不代表 CPU 硬件跳过了 Reset_Handler。

要区分：

```text
真实启动行为
vs
调试器为了方便自动运行到 main
```

---

## 7. 关联

- [[10_基础知识体系/04_验证与调试/03_GDB 的心智模型]]
- [[10_基础知识体系/03_Cortex-M与启动中断/16_从上电到 main 的完整动态图]]
