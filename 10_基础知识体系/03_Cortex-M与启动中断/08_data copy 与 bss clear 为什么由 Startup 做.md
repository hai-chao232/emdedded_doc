---
tags: [startup, data, bss, c-runtime]
---

# `.data` copy 与 `.bss` clear 为什么由 Startup 做？

## 一句话先说清

Linker 只决定“应该在哪里”；CPU reset 后 RAM 里实际是什么，必须由运行时软件建立。因此 Startup 用 linker 提供的边界 symbol 把 `.data` 初值复制到 SRAM，并把 `.bss` 清零。

---

# 1. Linker 不能在运行时搬数据

Linker 在 PC 上运行。

它生成 ELF 后工作就结束了。

所以即使 ELF 中已经写明：

```text
.data VMA = SRAM
.data LMA = Flash
```

STM32 上电时：

> 不会因为“ELF 知道”就自动把 bytes 搬过去。

需要真实机器指令执行 copy。

---

# 2. `.data` 需要谁提供什么？

### Compiler

把：

```c
uint32_t counter = 10;
```

归入适合的 input section。

### Linker

决定：

```text
Flash load image
RAM runtime address
_sidata
_sdata
_edata
```

### Startup

运行时做：

```text
Flash → RAM
```

三者共同完成。

---

# 3. `.bss` 同理

### Linker

只预留：

```text
[_sbss, _ebss)
```

### Startup

运行时：

```text
逐 word/byte 写 0
```

于是 C 语言语义才成立。

---

# 4. 为什么不让硬件自动做？

Cortex-M reset 机制只负责核心启动的基础行为，例如取 initial MSP 和 Reset vector。

它不知道你的：

```text
C compiler section policy
linker script
.data / .bss 边界
```

这些属于软件运行时约定。

---

# 5. 一个完整例子

```c
volatile uint32_t counter = 10;
volatile uint32_t flag;
```

Reset 前：

```text
Flash
counter init image = 10

SRAM
counter location = unknown
flag location    = unknown
```

Startup 后：

```text
SRAM
counter = 10
flag    = 0
```

---

# 6. 为什么故障注入很有价值？

如果临时跳过 `.data` copy：

```text
counter 不再保证是 10
```

跳过 `.bss` clear：

```text
flag 不再保证是 0
```

注意“某一次碰巧正确”不等于保证还成立。

---

# 7. 关联

- [[10_基础知识体系/02_构建与链接/14_data 与 bss 到底是什么]]
- [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]
- [[01_P0_仓库与构建基线/09_动手追踪一次启动]]
