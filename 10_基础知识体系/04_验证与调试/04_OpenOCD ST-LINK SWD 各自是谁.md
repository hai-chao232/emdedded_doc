---
tags: [debug, OpenOCD, ST-LINK, SWD]
---

# OpenOCD、ST-LINK、SWD 各自是谁？

## 一句话先说清

ST-LINK 是调试探针/硬件与其协议生态；SWD 是 ARM 的串行调试接口；OpenOCD 是运行在 PC 上、把 GDB 等工具和调试探针连接起来的调试服务器软件。

---

# 1. 物理链

```text
PC
 │ USB
 ▼
ST-LINK probe
 │ SWDIO / SWCLK / GND / optional RESET
 ▼
STM32F446
```

---

# 2. SWD

Serial Wire Debug。

核心物理信号：

```text
SWDIO
SWCLK
```

比传统 JTAG 使用更少引脚。

通过调试架构可以访问：

```text
CPU core
memory
registers
breakpoint resources
```

---

# 3. ST-LINK

ST 的调试/烧录探针。

NUCLEO 板上通常自带 ST-LINK，可通过板上连线调试目标 MCU。

它是：

> 硬件调试接口桥梁。

---

# 4. OpenOCD

Open On-Chip Debugger。

运行在你的 Linux PC。

它可以：

- 驱动调试 adapter；
- 初始化 target；
- 提供 GDB server；
- 执行 reset/halt；
- flash programming。

---

# 5. Cortex-Debug

VS Code 插件不是调试硬件本身。

它主要帮你：

```text
启动 GDB/OpenOCD
发送配置
显示 registers/call stack/memory
管理 breakpoint
```

---

# 6. 哪一层坏了怎么看？

### OpenOCD 找不到 ST-LINK

优先查：

```text
USB
权限
adapter config
```

### OpenOCD 能连接但 target 识别失败

查：

```text
SWD wiring
target power
reset
target config
```

### GDB 连上但 break main 不工作

查：

```text
ELF symbol
烧录是否一致
reset/continue 流程
优化
```

---

## 7. 关联

- [[10_基础知识体系/04_验证与调试/03_GDB 的心智模型]]
- [[10_基础知识体系/04_验证与调试/09_怎样验证 Reset_Handler 到 main]]
