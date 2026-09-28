---
tags: [cortex-m, thumb, function-pointer, vector-table]
stage: P0
---

# Thumb 状态与函数地址最低位

## 1. 最常见的困惑

你可能看到：

```text
nm:
08000278 T Reset_Handler
```

而向量表 dump：

```text
08000279
```

为什么多了 1？

## 2. Cortex-M 只执行 Thumb 指令集状态

Cortex-M 采用 Thumb/Thumb-2 指令编码。

在 ARM 的函数指针/异常入口约定中，入口值的最低位具有“Thumb 状态”含义。

所以：

```text
bit0 = 1
```

并不是在说：

> 第一条指令真的从一个奇数字节地址开始。

代码仍按指令要求对齐。

## 3. 地址位与状态信息混在一个值里

可以先用下面的心智模型：

```text
入口值：
xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxx1
                                   ↑
                              Thumb 状态信息
```

当处理器建立控制流时，会按架构规则解释这个值。

因此比较时不要机械做：

```text
vector[1] == nm symbol value
```

而应该比较“按 Thumb 语义解释后的入口是否对应”。

### 真实工程里的第二个例子：ELF 的 e_entry

向量表**不是唯一**带这个位的地方。

```bash
arm-none-eabi-nm build/.../exp001_gpio_output.elf | grep Reset_Handler
# 08000278 W Reset_Handler        ← 偶数

arm-none-eabi-readelf -h build/.../exp001_gpio_output.elf | grep Entry
# Entry point address:  0x8000279   ← 奇数
```

在 ARM ELF 约定下，这个 `e_entry` 用最低位表达 Thumb 入口状态，所以这里看到的是：

```text
Reset_Handler symbol value = 0x08000278
ELF e_entry                = 0x08000279
```

两者差 1，但并不矛盾。

> [!important] `ELF e_entry` 不是 Cortex-M 的硬件复位入口来源
> `e_entry` 是 **ELF 文件格式中的入口字段**。Cortex-M 芯片发生硬件 Reset 时，不会去读取 ELF Header。
>
> 真正的硬件复位入口来自向量表：
>
> ```text
> vector[0] → initial MSP
> vector[1] → Reset_Handler entry
> ```
>
> 所以 `e_entry = 0x08000279` 与向量表中的 Thumb Handler entry 都可能体现 bit0 的 Thumb 语义，但它们属于**不同层次、不同用途**，不能理解成“CPU 复位时读取 e_entry”。

> [!warning] 判据不是“是否发生控制流跳转”
> 一个很容易说过头的概括是：
>
> ```text
> ✗ 控制流要跳过去 → 一定是奇数
> ```
>
> **这是错的。** `bl` 同样改变控制流，但反汇编通常显示正常对齐的偶地址目标。
>
> 从你自己的反汇编里就能看到：
>
> ```asm
> 8000258 <SystemInit>:           ← 代码地址是偶数
> 8000278 <Reset_Handler>:        ← 代码地址是偶数
> 80002dc: bl   8000314 <_init>   ← 直接分支目标显示为偶数
> 80002f4: blx  r3                ← 间接调用会按架构规则解释寄存器中的入口值
> ```
>
> P0 阶段更稳妥的区分方式是：
>
> ```text
> 代码实际所在地址 / 符号地址 / 直接分支显示的目标
> → 通常看到按 Thumb 指令要求对齐的地址
>
> 需要携带 Thumb 状态信息的函数入口表示
> → bit0 可以为 1
> ```
>
> 典型的后一类包括：
>
> - Cortex-M 向量表中的 Handler entry；
> - ARM ABI 语境中的函数指针值；
> - 当前 ELF 中的 `e_entry` 表示。
>
> bit0 = 1 **不表示机器指令实际存放在奇数字节地址**。

真实反汇编见 [[10_基础知识体系/03_Cortex-M与启动中断/20_Reset_Handler 反汇编逐条对照]] 第 10.1 节。

## 4. 为什么函数指针也会受影响

在 ARM EABI 语境中，函数指针不是永远等价于“纯数据地址”。

它还必须携带足够的信息，使间接调用知道目标指令集状态。

Cortex-M 虽然只有 Thumb 状态，但这个表示规则仍然存在。

## 5. 错误设置会怎样

如果异常向量入口不满足 Cortex-M 对 Thumb 状态的要求，可能触发 fault，而不是正常进入 Handler。

不过要分清：**这个约束只在"值被当作入口解释"时才成立**。

```text
bit0 = 0 的入口值 → 被解释成"ARM 状态"
                   而 Cortex-M 没有 ARM 状态
                   → INVSTATE UsageFault

bit0 用在别处（比如当普通整数比较）→ 没有这个约束
```

因此 bit0 不是可随意忽略的装饰位——**前提是它落在入口值的位置上**。

## 6. P0 需要掌握到哪里

需要：

- 看到 `symbol` 与 `vector entry` 差 1 不慌；
- 知道 bit0 是状态含义；
- 知道实际代码并不是放在奇数字节起点。

暂时不需要：

- ARM/Thumb 历史兼容细节；
- BX/BLX 的全部状态切换规则；
- A/R profile 的 ARM state。
