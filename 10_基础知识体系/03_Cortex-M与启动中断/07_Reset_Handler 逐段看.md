---
tags: [startup, reset-handler]
aliases: [Reset_Handler]
---

# `Reset_Handler` 逐段看

## 一句话先说清

Reset_Handler 是复位后的第一段软件启动流程。它把“CPU 已经能执行指令”的状态继续推进到“C/C++ 程序可以安全进入 main”的状态。

---

# 1. 典型骨架

不同版本 startup 写法不同，但逻辑常类似：

```text
Reset_Handler:
    set/reload SP if startup does so
    call SystemInit
    copy .data
    clear .bss
    call __libc_init_array
    call main
    if main returns → exit / loop
```

必须打开当前实际 startup 核对，不要只背模板。

> [!tip] 本页讲"为什么"，真实反汇编在另一页
> 本页是**骨架**：每一步为什么存在、谁给谁提供数据。
> 拿到构建产物后，用 [[10_基础知识体系/03_Cortex-M与启动中断/20_Reset_Handler 反汇编逐条对照]] 对着真实机器码逐条看——
> 那页会告诉你 `ldr.w sp, [pc, #52]` 读的是哪个常量、copy 循环的边界寄存器是哪两个、
> 以及**为什么你当前工程里 copy 循环一次都没执行**。

---

# 2. 为什么可能再次设置 SP？

Cortex-M reset 硬件已经从 vector[0] 装过 initial MSP。

某些 startup 仍显式：

```asm
ldr sp, =_estack
```

这叫：

> 软件重新写一次 SP。

不能把它解释成：

> 此前 CPU 完全没有 stack。

---

# 3. `SystemInit()`

它处理芯片系统层初始化。

重要：

> 它的具体行为只由当前实际 `system_stm32f4xx.c` 决定。

不要按名字推断“自动上 180 MHz”。

详见：

[[10_基础知识体系/03_Cortex-M与启动中断/09_SystemInit 到底处在什么位置]]

---

# 4. `.data` copy

Linker 给出：

```text
_sidata  Flash source
_sdata   RAM destination
_edata   destination end
```

Reset_Handler 复制。

这一步建立：

```text
非零初值静态对象
```

的运行状态。

---

# 5. `.bss` clear

Linker 给出：

```text
_sbss
_ebss
```

Reset_Handler 对区间写 0。

这建立：

```text
未显式初始化/零初始化静态对象
```

的语言保证。

---

# 6. `__libc_init_array`

通常运行：

- preinit/init array；
- C++ 全局构造等 runtime 初始化。

当前即使只写 C，也不建议随意删掉，因为它属于当前 runtime 契约的一部分。

---

# 7. `main`

到这里才进入：

```text
用户应用层 C 程序
```

所以：

> `main` 不是处理器真正的复位入口。

---

# 8. 如果 main 返回？

裸机没有普通操作系统可以“返回”。

具体会：

- 调用 exit；
- 进入 loop；
- 走库实现；
- 触发 semihosting 等。

所以常见：

```c
while (1) {}
```

---

# 9. 证据链

静态：

```text
startup source
→ objdump -d
→ Reset_Handler machine code
```

动态：

```text
break Reset_Handler
→ step
→ break main
```

强度不同。

---

## 10. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/20_Reset_Handler 反汇编逐条对照]]（真实机器码，本页的实证版）
- [[10_基础知识体系/03_Cortex-M与启动中断/18_读Startup所需的最小Thumb汇编]]
- [[10_基础知识体系/02_构建与链接/23_真实链接脚本逐段注解]]（常量池里那些符号值从哪来）
- [[10_基础知识体系/03_Cortex-M与启动中断/08_data copy 与 bss clear 为什么由 Startup 做]]
- [[10_基础知识体系/03_Cortex-M与启动中断/09_SystemInit 到底处在什么位置]]
- [[10_基础知识体系/03_Cortex-M与启动中断/16_从上电到 main 的完整动态图]]
