---
tags: [diagram, timeline, startup]
---

# 从上电到 `main()` 时间线

把“地址空间”和“时间顺序”分开看。

```text
TIME
 │
 ▼
────────────────────────────────────────────────────

[PC 构建阶段]
source
→ preprocess
→ compile
→ assemble
→ link
→ ELF

────────────────────────────────────────────────────

[烧录]
ELF / extracted image
→ Main Flash

────────────────────────────────────────────────────

[Reset event]
CPU core reset state

────────────────────────────────────────────────────

[硬件步骤 1]
read initial MSP from reset vector table
MSP = ...

────────────────────────────────────────────────────

[硬件步骤 2]
read Reset vector
start executing Reset_Handler

────────────────────────────────────────────────────

[软件步骤 1]
optional SP reload in startup

────────────────────────────────────────────────────

[软件步骤 2]
SystemInit()

────────────────────────────────────────────────────

[软件步骤 3]
.data:
Flash initial image
→ SRAM runtime area

────────────────────────────────────────────────────

[软件步骤 4]
.bss:
SRAM range
→ zero

────────────────────────────────────────────────────

[软件步骤 5]
__libc_init_array()

────────────────────────────────────────────────────

[应用入口]
main()

────────────────────────────────────────────────────

[实验代码]
RCC / GPIO / ...
```

---

## 每一步是谁做？

| 步骤 | 主体 |
|---|---|
| 编译链接 | PC 工具链 |
| 烧录 | 下载工具/调试器 |
| 取 MSP / Reset vector | Cortex-M 硬件 |
| SystemInit/data/bss | Startup 软件 |
| main | 用户程序 |

如果你不知道“谁做”，就很容易把 Linker、Startup 和 CPU 混成一团。
