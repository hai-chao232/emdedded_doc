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

**链接器写 `e_entry` 时主动把最低位置 1 了。**

所以"符号地址"和"会被装进 PC 的入口值"**在这个工程里就同时存在、同时正确**，差 1 不是任何一方算错了。

> [!warning] 判据不是"是否跳转"，而是"这个值会不会被装进 PC"
> 一个很容易说过头的概括是：
> ```text
> ✗ 控制流要跳过去  →  奇数
> ```
> **这是错的。** 反例遍地都是——`bl` 也是控制流，但它的目标地址是偶数。
>
> 从你自己的反汇编里就能找到一对干净的反例：
> ```asm
> 8000258 <SystemInit>:          ← 偶数
> 8000278 <Reset_Handler>:       ← 偶数
>  8002dc:  bl   8000314 <_init> ← 直接跳转，目标偶数
>  8002f4:  blx  r3              ← 间接跳转，r3 是奇数
> ```
>
> 正确的区分是：
> ```text
> 这个值描述"代码放在哪"              →  偶数
>   nm 符号值、反汇编左侧地址、直接跳转的目标
>
> 这个值会被装进 PC 当作入口          →  bit0 是状态位
>   向量表项、函数指针、ELF 的 e_entry
> ```
>
> **关键差别在于"目标地址是怎么来的"**：
> ```text
> bl  →  目标以相对偏移编码在指令里，由 PC 算出
>         全程不经过"把某个值装进 PC"这一步
>         →  不涉及 bit0，目标必然是偶数
>
> blx rX  →  rX 里是一个"入口值"，要被装进 PC
>            →  必须解释 bit0
> ```
>
> 只有**间接**跳转（`blx rX`、异常入口、调试器直接设 PC）
> 才会去解释最低位。

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
