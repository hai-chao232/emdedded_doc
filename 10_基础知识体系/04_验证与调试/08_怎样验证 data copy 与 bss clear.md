---
tags: [debug, data, bss, startup]
---

# 怎样验证 `.data` copy 与 `.bss` clear？

## 目标命题

> Reset_Handler 真实执行后，有初值静态变量从 Flash initial image 复制到 SRAM，零初始化静态变量被清为 0。

---

# 1. 先制造可观察变量

```c
volatile uint32_t counter = 10;
volatile uint32_t flag;
```

预测：

```text
counter → .data
flag    → .bss
```

---

# 2. 静态 ELF 证据

```bash
arm-none-eabi-nm -n "$ELF" \
 | grep -E 'counter|flag|_sidata|_sdata|_edata|_sbss|_ebss'

arm-none-eabi-objdump -h "$ELF"
```

确认：

```text
counter runtime address 在 SRAM
.data LMA 在 Flash
flag 在 .bss 范围
```

---

# 3. Flash initial image

```bash
arm-none-eabi-objdump -s -j .data "$ELF"
```

尝试找到：

```text
0A 00 00 00
```

即 little-endian 的 10。

---

# 4. 动态验证方案 A：停在 main

```text
break main
reset
continue
```

看：

```text
counter == 10
flag == 0
```

这已经证明 startup 后状态正确，但还不能单独说明具体是哪段指令完成。

---

# 5. 动态验证方案 B：停在 Reset_Handler

这是更强学习法。

```text
break Reset_Handler
```

然后单步到 `.data` copy 前：

```text
读取 RAM[counter]
```

再单步 copy 完成：

```text
RAM[counter] 应变成 10
```

同理观察 `.bss` clear 前后。

---

# 6. 为什么“copy 前 RAM 恰好已经是 10”不推翻机制？

因为 RAM 可能保留：

- 上一次调试会话值；
- debugger 下载行为；
- 某种 reset 未完全掉电。

因此不能靠一次“前值不是 10”作为必要条件。

关键是：

> 程序不能依赖那个残留；startup 必须建立保证。

---

# 7. 故障注入

临时跳过 copy：

```text
counter 的语言保证被破坏
```

跳过 clear：

```text
flag 的语言保证被破坏
```

记录多种 reset 情况，观察“偶然正确”的陷阱。

---

## 8. 关联

- [[10_基础知识体系/04_验证与调试/10_复位类型 与 RAM 残留为什么会误导你]]
- [[10_基础知识体系/03_Cortex-M与启动中断/08_data copy 与 bss clear 为什么由 Startup 做]]
