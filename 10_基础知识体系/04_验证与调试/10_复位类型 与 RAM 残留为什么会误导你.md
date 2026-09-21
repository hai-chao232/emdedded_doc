---
tags: [debug, reset, RAM]
---

# 复位类型与 RAM 残留为什么会误导你？

## 一句话先说清

“Reset”不是一个单一物理事件。不同复位来源和调试器操作可能保留不同状态，尤其 RAM 可能没有像断电一样被清空。因此不能用“我这次看见 RAM 是 0”证明启动代码不需要初始化。

---

# 1. 常见复位来源

概念上可能包括：

```text
Power-on reset
NRST pin reset
Software system reset
Watchdog reset
Brown-out / power related reset
Debug reset
```

具体芯片的 reset tree 要查 RM。

---

# 2. 为什么 RAM 可能残留？

SRAM 是易失存储，但“易失”主要表示断电后不保证保持。

如果只是：

```text
CPU/system reset
```

而 SRAM 供电没有消失，某些内容可能仍然残留。

---

# 3. Debugger 还会改变情况

调试器可能：

- 下载程序时写内存；
- reset 后自动 halt；
- 自动初始化某些状态；
- 不同 reset 命令走不同路径。

因此：

```text
调试环境里的 reset
```

不一定和拔电重插完全相同。

---

# 4. 一个典型陷阱

你删掉 `.bss` clear。

然后在 main 里看到：

```text
flag == 0
```

错误结论：

> 原来 `.bss` 不清零也没事。

实际可能：

```text
RAM 上次本来就是 0
```

正确结论：

> 没有 startup clear 后，语言保证已经不存在；某次观测仍为 0 只是偶然。

---

# 5. 怎样做更好的故障实验？

至少比较：

```text
软件 reset
硬件 reset
重新下载
断电重上电
```

并重复多次。

重点不是追求“每次都坏”，而是证明：

> 没有初始化逻辑时，你无法依赖该状态。

---

# 6. 关联

- [[10_基础知识体系/04_验证与调试/08_怎样验证 data copy 与 bss clear]]
- [[10_基础知识体系/03_Cortex-M与启动中断/05_Cortex-M Reset 到底发生了什么]]
