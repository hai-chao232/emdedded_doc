---
tags: [linker, KEEP, gc-sections]
---

# `KEEP`、`--gc-sections` 与为什么向量表不能被删

## 一句话先说清

`--gc-sections` 会让 linker 丢弃它认为不可达/未使用的 section；但有些内容是“硬件使用、不是普通代码引用”，例如向量表，所以 linker script 常用 `KEEP()` 强制保留。

---

## 1. 为什么要 garbage collection？

启用：

```text
-ffunction-sections
-fdata-sections
-Wl,--gc-sections
```

后，不被实际引用的函数/数据可以被丢弃。

优点：

```text
减小固件
避免把整个库全部拉进来
```

---

## 2. Linker 怎样判断“使用”？

它主要依据 symbol/reference 图。

但 CPU 硬件有些访问不表现为普通函数调用。

---

## 3. 向量表就是典型例子

CPU reset 时会：

```text
直接读固定地址
```

它不会在代码里执行：

```c
use(vector_table);
```

所以纯软件引用分析可能无法理解“硬件一定会用”。

因此：

```ld
KEEP(*(.isr_vector))
```

明确告诉 linker：

> 这个 section 无论普通引用分析如何，都保留。

---

## 4. `KEEP` 不是把 section 放到 Flash

位置仍由：

```ld
>ROM
```

等规则决定。

`KEEP` 只影响：

> 是否允许 GC 删除。

---

## 5. 其他可能需要 KEEP 的内容

例如：

- 特殊注册表；
- init arrays；
- boot metadata；
- 由 DMA/硬件/外部工具按地址访问的表；
- 自定义 linker-set。

具体要看系统设计。

---

## 6. 关联

- [[10_基础知识体系/02_构建与链接/12_SECTIONS 命令怎样拼装最终镜像]]
- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
