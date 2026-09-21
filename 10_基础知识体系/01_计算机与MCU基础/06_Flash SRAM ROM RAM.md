---
tags: [fundamentals, flash, sram, memory]
aliases: [Flash 与 SRAM, ROM RAM]
---

# Flash、SRAM、ROM、RAM

## 一句话先说清

在当前 STM32F446 裸机学习中，最重要的区分是：Flash 非易失，主要保存固件；SRAM 易失，主要作为程序运行时工作区。ROM/RAM 是更广的类别词，不应机械等同于某一种具体硬件。

---

## 1. 非易失与易失

### Flash

断电后通常保留内容。

适合保存：

```text
代码
常量
.data 初始镜像
```

### SRAM

断电后内容不保证保留。

适合运行时：

```text
可写变量
栈
heap
DMA buffer
```

---

## 2. 为什么代码通常放 Flash？

MCU 上电后需要有地方保存程序。

如果程序只在 SRAM：

```text
断电
→ SRAM 内容丢失
→ 下次上电无程序可执行
```

所以固件烧进 Flash。

Cortex-M 能从 Flash 区域取机器指令。

---

## 3. 为什么可写变量主要放 SRAM？

Flash 虽然可编程，但写入机制与 SRAM 完全不同：

- 擦写有粒度限制；
- 擦写速度慢；
- 有寿命限制；
- 不能像普通 RAM 一样每条 C 赋值都直接写。

因此普通可写变量运行在 SRAM。

---

## 4. `.data` 为什么同时和 Flash、SRAM 有关？

```c
uint32_t counter = 10;
```

需求有两个：

```text
断电后初始值 10 不能丢
→ 初始镜像要在 Flash

运行时 counter 要频繁修改
→ 运行副本要在 SRAM
```

所以：

```text
Flash: 10
  ↓ startup copy
SRAM: counter = 10
```

这就是 VMA/LMA 的经典来源。

---

## 5. `.bss` 为什么只需要 SRAM 空间？

```c
uint32_t flag;
```

语言要求初值为 0。

没有必要把 1000 个 0 写进 Flash 镜像：

```text
00 00 00 00 00...
```

只需：

```text
Linker 预留 SRAM
Startup 清零
```

---

## 6. ROM 是什么？

ROM 原义：

```text
Read-Only Memory
```

现代 MCU 语境中，人们有时会把“非易失程序存储区”泛称 ROM，但 STM32 主程序存储通常是真正的 Flash。

所以课程里尽量用：

```text
Flash
```

而不是模糊说 ROM。

Linker script 里区域名字叫：

```ld
ROM
```

也只是脚本给 memory region 起的名字，不表示物理器件真的一定是掩膜 ROM。

---

## 7. RAM 是什么？

RAM：

```text
Random Access Memory
```

是更广泛类别。

SRAM 是 RAM 的一种。

当前 STM32 内部工作内存主要讨论 SRAM。

---

## 8. 当前 STM32F446ZE 的课程地图

```text
Flash
0x08000000
┌─────────────────────────┐
│ Vector Table            │
├─────────────────────────┤
│ .text                   │
├─────────────────────────┤
│ .rodata                 │
├─────────────────────────┤
│ .data load image        │
│ ...                     │
└─────────────────────────┘
0x08080000 boundary


SRAM
0x20000000
┌─────────────────────────┐
│ .data runtime           │
├─────────────────────────┤
│ .bss                    │
├─────────────────────────┤
│ heap / reserve          │
│                         │
│ free / runtime space    │
│                         │
│ stack ↓                 │
└─────────────────────────┘
0x20020000 boundary
```

实际 section 顺序和大小以 ELF / linker script 为准。

---

## 9. “Flash 里的地址”和“文件偏移”不同

ELF 文件在你的 PC 磁盘上有自己的 file offset。

某段内容最终在 MCU 中可能被标记为：

```text
VMA/LMA 0x0800....
```

不要把：

```text
ELF 文件第 1234 Byte
```

和：

```text
MCU address 0x08001234
```

直接等同。

---

## 10. Reset 会清空 RAM 吗？

不能简单说“会”。

不同 reset 类型、芯片行为、debugger 操作都可能影响实际 RAM 内容。

这正是为什么 startup 必须显式建立语言要求的 `.data/.bss` 状态，而不能依赖“复位后 RAM 恰好是什么”。

后面见：

[[10_基础知识体系/04_验证与调试/10_复位类型 与 RAM 残留为什么会误导你]]

---

## 11. 关联

- [[10_基础知识体系/01_计算机与MCU基础/04_地址 地址空间 与内存映射]]
- [[10_基础知识体系/01_计算机与MCU基础/10_栈 Stack 是什么]]
- [[10_基础知识体系/01_计算机与MCU基础/12_堆 Heap 是什么]]
- [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]
