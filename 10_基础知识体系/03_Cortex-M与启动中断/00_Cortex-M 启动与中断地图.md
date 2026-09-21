---
tags: [MOC, cortex-m, startup, interrupt]
---

# Cortex-M、启动与中断地图

这一组解决的是：

> STM32F446 上电/复位以后，CPU 到底如何开始执行？向量表谁用？Startup 做什么？中断以后又怎么回得来？

建议主线：

```text
[[10_基础知识体系/03_Cortex-M与启动中断/01_Cortex-M4 与 STM32F446 的关系]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/02_SP MSP PSP 是什么]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/03_PC LR xPSR 分别是什么]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/05_Cortex-M Reset 到底发生了什么]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/06_Startup 文件到底是什么]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/07_Reset_Handler 逐段看]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/08_data copy 与 bss clear 为什么由 Startup 做]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/09_SystemInit 到底处在什么位置]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/10_异常 Exception 与中断 Interrupt 的关系]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/11_异常进入时 CPU 自动做了什么]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/12_NVIC 到底负责什么]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/13_VTOR 向量表重定位]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/14_Weak Handler 为什么能被覆盖]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/16_从上电到 main 的完整动态图]]
        ↓
[[10_基础知识体系/03_Cortex-M与启动中断/17_一次外部中断发生时的完整流程]]
```

---

## 先记住三个“谁”

### CPU 硬件自己做

```text
Reset 时取 initial MSP / Reset vector
异常进入时自动保存基本现场
根据向量表取得 handler 地址
异常返回时恢复现场
```

### Startup 软件做

```text
SystemInit
.data copy
.bss clear
runtime init
main
```

### Linker 做

```text
决定向量表/代码/数据地址
解析 symbol
生成 ELF
```

这三个角色一旦混淆，启动流程就会彻底乱。

---

## 与 P0 主课的关系

- [[01_P0_仓库与构建基线/05_Linker Script 与内存布局]]
- [[01_P0_仓库与构建基线/06_Startup 与 Reset_Handler]]
- [[01_P0_仓库与构建基线/08_P0 完整启动链]]
- [[01_P0_仓库与构建基线/09_动手追踪一次启动]]
