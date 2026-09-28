---
tags: [power-on, reset, debugger, ram]
stage: P0
---

# 上电、Reset 与重新执行程序的区别

## 1. 为什么要单独区分

调试启动代码时，如果把下面几件事都叫“重启”，很容易得出错误结论：

```text
断电再上电
硬件复位
软件触发 system reset
调试器 reset
调试器 halt/restart
把 PC 改回某地址
```

它们对 CPU、外设、RAM、调试状态的影响不一定相同。

## 2. Power-on reset

真正掉电再上电时：

- 电源域重新建立；
- 芯片按照上电复位规则进入已定义起点；
- RAM 内容通常不能依赖；
- 外设状态按照芯片规定重新初始化。

具体行为应以目标 STM32 的官方文档为准。

## 3. System reset / hardware reset

Reset 通常会让处理器重新走复位入口，但：

> Reset 不自动等价于“所有物理状态都像断电一样消失”。

特别是 RAM，在不同复位源下可能保留先前内容，或者至少不能把“必定随机/必定清零”当作普遍事实。

## 4. Debugger reset 更要小心

调试器可能提供多种 reset strategy，例如：

```text
system reset request
hardware reset pin
connect under reset
reset and halt
```

IDE 上一个叫“Restart”的按钮甚至可能只是执行若干调试器命令。

因此做启动验证时必须知道：

> 我按的这个按钮到底发了什么 reset 操作？

## 5. 为什么这影响 `.data/.bss` 实验

如果 `.bss` clear 之前你看到：

```text
flag == 0
```

不能立刻推导：

> “看来芯片硬件自动把 .bss 清零了。”

可能只是：

```text
上一次运行留下的 0
```

正确实验应该观察：

```text
clear loop 是否真的执行写 0
```

而不是依赖 reset 前的 RAM 恰巧是什么。

同理 `.data`：

```text
counter copy 前已经是 10
```

也不能证明 copy 无意义。

它可能只是 RAM 残留。

## 6. 怎样做更强的验证

可以：

1. 在 `.data` copy 前人为改变 RAM 目标值。
2. 继续执行 copy。
3. 观察它被恢复成 Flash 初始化镜像。
4. 在 `.bss` clear 前人为把目标区域改成非零。
5. 执行 clear。
6. 观察变为 0。

这样比“猜 reset 后 RAM 会是什么”可靠得多。

## 7. 当前阶段需要记住的边界

P0 只要求：

- 知道 power-on/reset/debug restart 不是同义词；
- 不把 RAM 初始观察值当成硬件必然；
- 用“动作前后变化”证明 startup。

不要求现在掌握 STM32 所有复位源和电源域细节。
