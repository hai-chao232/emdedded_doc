---
tags: [reference, memory, linker]
aliases: [RAM Flash, Section 速查]
---

# RAM / Flash 与常见 Section

这是一张**速查卡**。完整原理见：

- [[01_P0_仓库与构建基线/05_Linker Script 与内存布局]]
- [[01_P0_仓库与构建基线/06_Startup 与 Reset_Handler]]
- [[01_P0_仓库与构建基线/09_动手追踪一次启动]]

## 1. 当前 STM32F446ZE 主存储区

```text
Flash
起始：0x0800_0000
大小：512 KiB
结束边界：0x0808_0000

SRAM
起始：0x2000_0000
大小：128 KiB
结束边界：0x2002_0000
```

最后一个 SRAM byte 是 `0x2001_FFFF`，而 `0x2002_0000` 是结束边界/可作为初始 SP 的顶部数值。

## 2. 概念内存图

```text
Flash
0x0808_0000 ─────────────────── end boundary
              ...
              .data initial image
              .rodata
              .text
              .isr_vector
0x0800_0000 ─────────────────── origin

SRAM
0x2002_0000 ─────────────────── _estack
              stack ↓

              runtime free area

              heap ↑
              other NOBITS areas
              .bss
              .data runtime copy
0x2000_0000 ─────────────────── origin
```

实际位置以 linker script + ELF section table 为准。

## 3. Section 速查

| Section | 典型内容 | Flash | RAM | Startup |
|---|---|---:|---:|---|
| `.isr_vector` | Cortex-M 向量表 | ✅ | 通常否 | CPU 复位直接读取 |
| `.text` | 机器指令 | ✅ | 通常否 | 无 |
| `.rodata` | 只读常量 | ✅ | 通常否 | 无 |
| `.data` | 非零初值的可写静态对象 | ✅ 初始镜像 | ✅ 运行副本 | Flash → RAM copy |
| `.bss` | 零初值/未显式初始化静态对象 | 不保存整段零 | ✅ | clear to zero |
| heap/stack reserve | 运行时预留 | ❌ | ✅ | 视实现 |

## 4. `.data`

```c
uint32_t counter = 10;
```

典型：

```text
LMA = Flash
VMA = RAM
```

固件中保存 `10`，Reset_Handler 再复制到 RAM 的运行位置。

## 5. `.bss`

```c
uint32_t flag;
static uint8_t buffer[32];
```

Linker 在 RAM 预留空间，Startup 清零；无需在 Flash 中保存整段 `00`。

## 6. VMA / LMA

- **VMA**：section 运行时使用的地址。
- **LMA**：section 初始镜像在固件中被加载/保存的地址。

`.data` 是最经典的 VMA ≠ LMA 案例。

## 7. Linker Symbol

| Symbol | 含义 |
|---|---|
| `_estack` | 初始栈顶边界 |
| `_sidata` | `.data` 初始镜像 Flash 地址 |
| `_sdata` | `.data` RAM 起点 |
| `_edata` | `.data` RAM 末端边界 |
| `_sbss` | `.bss` 起点 |
| `_ebss` | `.bss` 末端边界 |

常用：

```text
.data length = _edata - _sdata
.bss  length = _ebss - _sbss
```

但 `arm-none-eabi-size` 汇总的 `bss` 可能还包含其他 NOBITS section，所以不保证等于 `_ebss - _sbss`。

## 8. Stack

Cortex-M 常见栈向低地址增长：

```text
高地址
_estack
  ↓ push / call / exception
低地址
```

链接成功不能证明最大运行时栈深度一定安全。

## 9. Heap

只有程序使用动态分配（如 `malloc/new`）时才真正参与运行，但 linker script 可能提前预留 heap 空间，因此 `size` 中仍可能体现 RAM 占用。

早期裸机实验尽量避免依赖动态分配。

## 10. `size` 的粗略估算

```text
Flash ≈ text + data
RAM static ≈ data + bss
```

适合快速比较，不适合作为完整内存安全证明，因为还存在 stack、heap、自定义 section 等。

## 11. 三个常见误区

- `.data 在 RAM` **不代表** Flash 里没有初始镜像。
- `.bss 不占 Flash` 更准确地说是“不需要保存整段零内容”。
- `_estack = 0x20020000` 不代表该地址是 SRAM 中一个普通可写 word；它是顶部边界数值。
