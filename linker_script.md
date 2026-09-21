# linker\_script

# 理解为什么 *linker\_script* 现在出现

前面 **Platform** 已经告诉构建系统：

```CMake
我是谁？                STM32F446xx
CPU 是谁？              Cortex-M4F
头文件去哪找？           CMSIS
怎么启动？               startup_stm32f446xx.s
SystemInit 在哪？       system_stm32f4xx.c
```

现在还缺一个问题：

> 编译出来的代码，应该放在 MCU 的什么地址
> 
> 

# MCU 的 "地址"

可以把 MCU 想象成一栋大楼：

```CMake
MCU 地址空间
0x0000 0000 ─────────────────
             一大片地址
             可以对应不同的硬件资源
             ↓
             Flash
             SRAM
             外设寄存器
             ...
0xFFFF FFFF ─────────────────
```

对于 **STM32F446** 来说，其是 *32位 Cortex\-M4* 

所以 *CPU*  可以看到 一个 32位地址空间

> `0x0000_0000 ~ 0xFFFF_FFFF`
> 
> 也就是大约 **4GB** 的地址范围
> 
> 

---

我们之前理解过 **总线系统** ，即 **CPU** 访问特殊地址

**总线系统**  会把这些特殊的地址 对应映射到 芯片硬件资源

---

但是我们知道 芯片资源是有限的，根本用不上 **4GB** 地址

所以会有大片区域是没有对应模块的

**CPU** 可以访问这些地址，但总线没有设备应答，这时就会触发：

> **硬件异常（BusFault）** 
> 
> 

---

这些特殊地址都是芯片规定好的，对于 **F446** ：

```CMake
0x0800 0000 → Flash
0x2000 0000 → SRAM
0x4000 0000 → 外设区域
...
```

# linker\_script 的作用

> 我们已经了解，这些硬件资源地址都是芯片确定好的，每个芯片略有差异
> 
> 那么 **linker\_script** 就是用来描述这些 地址的
> 
> 这样链接器就知道了这些地址，明白了程序应该怎么放
> 
> 

# 建立 *平台linker* 

> 1. 找到 **ST** 给我们的参考 **linker\_script** （属于芯片层级）
> 
>     执行：
> 
>     `ls third_party/STM32CubeF4/Projects/STM32F446ZE-Nucleo/Templates/STM32CubeIDE/STM32F446ZETX_FLASH.ld`
> 
>     这个就是 **ST** 官方给的参考文件的路径
> 
> 2. 建立我们自己平台 link目录，并把官方模板复制过来
> 
>     `mkdir -p platform/stm32f446ze/linker`
> 
>     `cp third_party/... platform/...`
> 
>     最终：
> 
>     ```CMake
>     platform/
>     └── stm32f446ze/
>         ├── CMakeLists.txt
>         ├── README.md
>         └── linker/
>             └── STM32F446ZE.ld
>     ```
> 
> 

---

> 这时就有一个疑问，之前强调过：
> 
> *“不要把 **`third_party`** 的官方文件复制到 **`platform`**”* 
> 
> 但 **linker\_script** 是例外
> 
> 我们现在拿 ST 官方模板作为正确起点，但未来很可能会根据自己的工程需要修改：
> 
> ```CMake
> heap 大小
> stack 大小
> 自定义 section
> bootloader 布局
> Flash 分区
> RAM 分区
> 特殊数据段
> ```
> 
> 

# 理解 *linker* 

- `ENTRY(Reset_Handler)`

    > 意思是：
    > 
    > ```CMake
    > 链接器认为程序入口
    >         ↓
    > Reset_Handler
    > ```
    > 
    > 

    > 而 `Reset_Handler` 正好来自我们之前接进来的：
    > 
    > `startup_stm32f446xx.s`
    > 
    > 这里我们可以明显看到：
    > 
    > ```CMake
    > startup
    >    ↕
    > linker script
    > 
    > 它们其实不是两套互不相关的东西
    > 
    > startup_stm32f446xx.s
    >         │
    >         └── 定义 Reset_Handler
    >                     ↑
    >                     │
    > STM32F446ZE.ld      |
    >         │           |
    >         └── ENTRY(Reset_Handler)
    > ```
    > 
    > 

    > 不过这里有一个细节要提前区分：
    > 
    > **"MCU 上电真正从哪开始运行，并不是单靠这句 ****`ENTRY()`**** 决定的"** 
    > 
    > **Cortex\-M** 上电时主要看的是 **中断向量表** ：
    > 
    > ```CMake
    > 向量表第 1 项 → 初始 MSP
    > 向量表第 2 项 → Reset_Handler 地址
    > ```
    > 
    > `ENTRY(Reset_Handler)` 更多是告诉 ELF/linker：
    > 
    > *"这个程序的入口符号是它"* 
    > 
    > 等我们把 **startup** 对着 **linker** 一起看，你会更清楚。
    > 
    > 

- `_estack = ORIGIN(RAM) + LENGTH(RAM)`

    > `_estack` \-\-\-\-\> **Linker Symbl** \(链接器符号\)
    > 
    > 这里不能把`_estack` 当作 *"变量"* ，它更像：
    > 
    > **Linker 在最终程序里面贴的一个 "地址标签"** 
    > 
    > ---
    > 
    > 理解：先从 C 变量开始
    > 
    > 我们写：`int x = 10`
    > 
    > 假设 *x* 被放到了：**0x2000\_0000** 
    > 
    > 那么我们可以理解：
    > 
    > ```CMake
    > x
    > │
    > └──→ 0x20000000
    > ```
    > 
    > 在 C 语言里：`&x`
    > 
    > 就能得到：**0x2000\_0000** 
    > 
    > 所以我们可以说：
    > 
    > ***`x`****** 是一个变量，它对应某个内存地址。***
    > 
    > ---
    > 
    > 但是 `_estack` 不一样，这里：
    > 
    > - `_estack = ORIGIN(RAM) + LENGTH(RAM);`
    > 
    > 不是 C语言，这是 **Linker Script** 的语法，意思是：
    > 
    > - *创建一个叫 **`_estack`** 的 ****linker symbol****，让它的值等于 RAM 的末尾地址。* 
    > 
    > - 比如** RAM起始地址***：0x2000\_0000 。***大小**：*128KB 。***结束地址**：*0x2002\_0000*
    > 
    > - 那么 `-estack = 0x2002_0000;`
    > 
    > - 这样 *linker* 就知道：
    > 
    >     ```CMake
    >     _estack
    >        ↓
    >     0x20020000
    >     ```
    > 
    > 但是：
    > 
    > **这里没有创建一个存放数据的变量。**
    > 
    > **它只是创建了一个“名字 → 地址”的对应关系。**
    > 
    > *链接器符号可以参与地址计算、和段定义结合使用，比如用来告诉启动代码栈顶在哪里。*
    > 
    > ---
    > 
    > **startup** 汇编文件里面也可以直接写 *\_estack* 
    > 
    > 只不过汇编器在编译 startup\.s 时并不知道 `_estack` 的具体数值，
    > 
    > 但它会把 `_estack` 当作“未解析符号”留给链接器。
    > 
    > 链接器最后统一解析，把 `_estack` → `0x20020000`，于是汇编指令能正确拿到 RAM 顶端地址。
    > 
    > 

- `_Min_...`

    ```CMake
    _Min_Heap_Size = 0x200;
    _Min_Stack_Size = 0x400;
    
    
    # 换算一下：
    0x200 = 512 Bytes
    0x400 = 1024 Bytes
    
    
    # 所以这里表达的是：
    # 最小 Heap 需求  = 512 B
    # 最小 Stack 需求 = 1 KB
    ```

    > 这里只是定义了两个 *linker symbol* 
    > 
    > 后面会有某个 **section** 使用这两个 *linker symbol* 
    > 
    > 到那里它们才真正参与内存布局
    > 
    > ---
    > 
    > 这里的 **section（段）** ，可以理解：
    > 
    > **编译器和链接器用来组织程序内存的一块“区域”。**
    > 
    > 在嵌入式开发里，理解 `.text`、`.data`、`.bss`、heap、stack 等，
    > 
    > 本质上就是在理解各种 section 如何被放到 Flash 和 RAM 中。
    > 
    > 后面我们在详细理解 \-\-\-\> ***section***
    > 
    > ---
    > 
    > 

- `MEMORY{ ...`

    ```CMake
    MEMORY
    {
      RAM (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
      ROM (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
    }
    
    
    # 就是在向 linker 描述：
        STM32F446ZE
        │
        ├── RAM
        │   起始：0x20000000
        │   大小：128 KB
        │
        └── ROM
            起始：0x08000000
            大小：512 KB
    ```

    > 这里虽然名字写的是 **ROM** 
    > 
    > 对于 *STM32F446ZE* 来说，我们这里指的就是：
    > 
    > **片内 Flash**
    > 
    > ---
    > 
    > 在 *linker script* 中
    > 
    > ```CMake
    > ROM
    > FLASH
    > MYFLASH
    > ABC
    > 
    > 这些 region 名字本身是作者定义的
    > 
    > 真正决定硬件内存的是：
    >     ORIGIN
    >     LENGTH
    > ```
    > 
    > 

    ---

    `(xrw)` 和 `(rx)`

    > 可以先记：
    > 
    > ```Plain Text
    > r = readable   可读
    > w = writable   可写
    > x = executable 可执行
    > ```
    > 
    > 因此：
    > 
    > ```Plain Text
    > RAM → xrw
    > 可以读 / 写 / 执行
    > 
    > ROM → rx
    > 可以读 / 执行
    > ```
    > 
    > 这也符合基本直觉：
    > 
    > ```Plain Text
    > Flash
    >     ↓
    > 主要放程序代码、常量
    > 
    > RAM
    >     ↓
    > 主要放运行时变量
    > ```
    > 
    > 

- `SECTIONS{ ...`

    > 在之前我们是这样理解的：
    > 
    > ```CMake
    > 这里的 **section（段）** ，可以理解：
    > **  编译器和链接器用来组织程序内存的一块“区域”。**
    >   在嵌入式开发里，理解  *.text*、 *.data*、 *.bss*、 *heap*、 *stack* 等，
    >   本质上就是在理解各种 section 如何被放到 Flash 和 RAM 中。
    > ```
    > 
    > 

    从 C 代码到 section 举例：

    ```C++
    int global_init = 10;
    
    int global_uninit;
    
    const int value = 100;
    
    void foo(void)
    {
        int local = 1;
    }
    
    /* ************************************************
     *编译后，不同内容通常会进入不同的 section：
     *
     *内容               常见 section        通常放在哪里
     ****************************************************
     *函数代码              .text                Flash
     *已初始化全局变量       .data                RAM
     *未初始化全局变量       .bss                 RAM
     *const 数据            .rodata             Flash
     *局部变量              Stack                RAM
     *malloc() 内存         Heap                 RAM
     ***************************************************** */
    ```

    可以想象：

    ```SQL
    
    Flash
    +------------------+
    | .text            |  程序代码
    +------------------+
    | .rodata          |  const 数据
    +------------------+
    | .data 初始值      |
    +------------------+
    
    
    RAM
    +------------------+
    | .data            |  已初始化全局变量
    +------------------+
    | .bss             |  未初始化全局变量
    +------------------+
    | Heap             |  malloc 使用
    |                  |
    |        ↓         |
    |                  |
    |        ↑         |
    | Stack            |  函数调用、局部变量
    +------------------+
    ```

    **可以把 section 理解成“内存分类盒子”** 

    > 假设有两个源文件：
    > 
    > ```Plain Text
    > main.c
    > foo.c
    > ```
    > 
    > 编译后：
    > 
    > ```Plain Text
    > main.o
    > ├── .text
    > ├── .data
    > └── .bss
    > 
    > foo.o
    > ├── .text
    > ├── .data
    > └── .bss
    > ```
    > 
    > 链接器把它们组合：
    > 
    > ```Plain Text
    > 所有 .text
    >       ↓
    > +--------------------+
    > | 最终 .text         |
    > | main.o 的代码      |
    > | foo.o 的代码       |
    > +--------------------+
    > 
    > 所有 .data
    >       ↓
    > +--------------------+
    > | 最终 .data         |
    > | main.o 的变量      |
    > | foo.o 的变量       |
    > +--------------------+
    > ```
    > 
    > 这就是 linker 的重要工作之一：
    > 
    > **把各个目标文件里的 section 收集、合并，然后放到指定的内存地址。**
    > 
    > 

    > ```CMake
    > MEMORY
    > =
    > “我有哪几个仓库？”
    > 
    > 
    > SECTIONS
    > =
    > “各种货物分别放哪个仓库？”
    > ```
    > 
    > 

- `.isr_vector`

    ```CMake
    .isr_vector :
    {
      . = ALIGN(4);
      KEEP(*(.isr_vector))
      . = ALIGN(4);
    } >ROM
    ```

    `.isr_vector` 就是：

    > **中断向量表**
    > 
    > 

    startup 汇编里会有类似一个 `.isr_vector` section，里面放：

    ```Plain Text
    _estack
    Reset_Handler
    NMI_Handler
    HardFault_Handler
    ...
    USART IRQ
    Timer IRQ
    ...
    ```

    linker 看到：

    ```Plain Text
    *(.isr_vector)
    ```

    意思可以先理解成：

    > 把所有输入文件里的 `.isr_vector` 内容收集过来。
    > 
    > 【**输入文件**：就是编译出来的 `.o` 文件，Linker 的原材料。】
    > 
    > 

    然后：

    ```Plain Text
    >ROM
    ```

    告诉 linker：

    > 把最终的 `.isr_vector` 放进 `ROM`。
    > 
    > 

    也就是 Flash。

    于是出现了：

    ```Plain Text
    startup_stm32f446xx.s
            │
            │ 产生 .isr_vector
            ▼
    linker script
            │
            │ 放到 ROM
            ▼
    Flash 0x08000000...
    ```

    这就是 startup 和 linker 的第一次真正合作。

- `KEEP()`

    > `KEEP(*(.isr_vector))`
    > 
    > 

    可以先记：

    > **无论 linker 是否觉得它“没人引用”，都必须保留它。**
    > 
    > 

    因为现代编译经常使用：

    ```Plain Text
    --gc-sections
    ```

    去掉看起来没用的 section。

    但向量表非常特殊：

    - C代码不一定显式调用它

    - CPU 却必须使用它。

    所以：

    ```Plain Text
    KEEP(...)
    ```

    等于：

    > “别自作聪明删掉这个。”
    > 
    > 

---

- `. = ALIGN(4)`

    > 这里的 `.` 非常重要
    > 
    > 

    在 linker script 中 `.` ：

    > 链接器当前正在放置内容的地址
    > 
    > ---
    > 
    > 例如 RAM：
    > 
    > ```Plain Text
    > 0x20000000
    > ```
    > 
    > 现在链接器开始放 `.data`：
    > 
    > ```Plain Text
    > .data :
    > {
    >     *(.data)
    > }
    > ```
    > 
    > 假设 `.data` 放了 100 字节：
    > 
    > ```Plain Text
    > 开始：
    > .
    > ↓
    > 0x20000000
    > 
    > 放入 100 Bytes 后：
    > .
    > ↓
    > 0x20000064
    > ```
    > 
    > 所以可以理解为：
    > 
    > ```Plain Text
    > . = 当前地址
    > ```
    > 
    > 但是更准确地说：
    > 
    > `.` 是 **location counter（位置计数器）**。
    > 
    > 

    `. = ALIGN(4)` 可以理解：

    > 找到当前地址之后，第一个满足 4 字节对齐的地址。
    > 
    > 

    例如当前：

    ```Plain Text
    0x08000101
    
    对齐之后
    
    0x08000104
    ```

    **为什么要 4字节 对齐** 

    > STM32 的 Cortex\-M 内核是 32 位的。
    > 
    > 天然喜欢处理 `4 Bytes`
    > 
    > 比如：
    > 
    > ```Plain Text
    > uint32_t x;
    > 
    > 内存中：
    >     地址
    >     0x20000000   Byte 0
    >     0x20000001   Byte 1
    >     0x20000002   Byte 2
    >     0x20000003   Byte 3
    > ```
    > 
    > CPU 一次：读取 32 bit
    > 
    > 这就是：4 字节对齐
    > 
    > ---
    > 
    > 假设：
    > 
    > `uint32_t x;` \-\-\-\-\> 放到了 *0x20000001*
    > 
    > ```Plain Text
    > 内存中：
    >     地址
    >     0x20000001   Byte 0
    >     0x20000002   Byte 1
    >     0x20000003   Byte 2
    >     0x20000004   Byte 3
    > ```
    > 
    > 那么就会：
    > 
    > ```Plain Text
    > 一个 4-byte 边界
    > 
    > 0x20000000 │
    > 0x20000001 │ ← x 开始
    > 0x20000002 │
    > 0x20000003 │
    > -----------┼────────
    > 0x20000004 │
    > ```
    > 
    > 于是这个这个 `uint32_t` 横跨了两个 ***4\-byte*** 的边界
    > 
    > CPU 需要：
    > 
    > ```Plain Text
    > 第一次读取
    > 0x20000000 ~ 0x20000003
    > 
    > 第二次读取
    > 0x20000004 ~ 0x20000007
    > 
    > 然后：
    >     组合数据
    > ```
    > 
    > - 访问速度更慢 
    > 
    > - CPU 需要额外处理 
    > 
    > - 某些 Cortex\-M 或某些访问类型可能产生 Fault
    > 
    > ---
    > 
    > 也就是说 ***对齐 ***是用 空间 换 CPU效率
    > 
    > 对齐 一般 使用 填充：
    > 
    > - 为了对齐（Alignment），
    > 
    > - 所以加入填充（Padding）
    > 
    > ---
    > 
    > 在嵌入式 RAM 很紧张时，**结构体 padding 值得关注**
    > 
    > 因为一个结构体多浪费 3\~7 字节，乘以几百、几千个元素后，影响就会非常明显。
    > 
    > 一般经验：
    > 
    > **结构体成员按照从大到小排列，可以减少 padding。**
    > 
    > 

- `.text`

    > ```Plain Text
    > .text :
    > {
    >     *(.text)
    >     *(.text*)
    >     ...
    > } >ROM
    > ```
    > 
    > 

    > `.text` 基本可以先等同于 ***程序代码 ***比如：
    > 
    > ```Plain Text
    > int add(int a, int b)
    > {
    >     return a + b;
    > }
    > ```
    > 
    > 编译之后产生的机器指令，通常就进入 `.text`，因此：
    > 
    > ```Plain Text
    > 你的 C 函数
    >     ↓
    > 编译器
    >     ↓
    > .text
    >     ↓
    > linker
    >     ↓
    > ROM / Flash
    > ```
    > 
    > 很合理，因为程序代码平时不需要改变，烧进 Flash 就行。
    > 
    > 

    > ---
    > 
    > ```Plain Text
    > *(.text)
    > *(.text*)
    > ```
    > 
    > 第一个 `*` 表示：
    > 
    > **所有输入文件**
    > 
    > 也就是：
    > 
    > ```Plain Text
    > main.o
    > foo.o
    > bar.o
    > ...
    > ```
    > 
    > 第二个 `*` 这里的 `*` 是通配符：
    > 
    > **所有以 ****`.text`**** 开头的 section 名字**
    > 
    > ---
    > 
    > 简单理解就是把：
    > 
    > 把所有输入目标文件中名称以 `.text` 开头的 ***input sections***，
    > 
    > 收集到最终的 ***output section*** `.text` 中。
    > 
    > 

- `_etext = .;`

    > 当前位置定义符号 `_etext`，即：
    > 
    > ```Plain Text
    > _etext
    >     ↓
    > 程序代码区域结束地址
    > ```
    > 
    > 

- `.rodata`

    ```CMake
    .rodata :
    {
        *(.rodata)
        *(.rodata*)
    } >ROM
    
    
    # read-only data （只读数据）
    
    # .rodata
    #     ↓
    # ROM / Flash
    ```

    比如：

    ```C
    const int x = 10;
    
    printf("hello");
    ```

    都属于只读数据

- `.ARM.extab` 和 `.ARM.exidx`

    > 这两个现阶段**不要花太多精力**。
    > 
    > 它们主要和 ARM 的：
    > 
    > ```Plain Text
    > 异常展开
    > stack unwinding
    > C++ 异常支持
    > 调试/回溯
    > ```
    > 
    > 等机制有关。
    > 
    > 现在只需要知道：
    > 
    > ```Plain Text
    > 它们也是编译器/ARM ABI 可能产生的元数据
    >         ↓
    > 也被放在 ROM
    > ```
    > 
    > 即可。
    > 
    > 

---

- `_sidata = LOADADDR(.data);`

    > 这里的 `_sidata` 表示：
    > 
    > `.data` 这批“有初始值的全局/静态变量”，它们的初始内容在 Flash 里的起始地址。
    > 
    > 比如：
    > 
    > `int g_count = 10;`
    > 
    > - 程序运行时，`g_count` 必须能被修改，所以它应该待在 RAM；
    > 
    > - 但 MCU 掉电后 RAM 不保存内容，因此初始值 `10` 必须先存放在 Flash。
    > 
    > ```C
    > 所以它有两个地址概念：
    > 
    >     Flash 里的初始镜像
    >         ↓
    >     _sidata
    >     
    >     
    >     上电后真正运行的位置
    >         ↓
    >     RAM 里的 .data
    > ```
    > 
    > 

- `.data`

    ```C
    .data :
    {
        _sdata = .;
    
        *(.data)
        *(.data*)
        *(.RamFunc)
        *(.RamFunc*)
    
        _edata = .;
    } >RAM AT> ROM
    ```

    这里最关键的是：

    ```Plain Text
    >RAM AT> ROM
    ```

    可以直接读成：

    > `.data` **运行时放在 RAM，但它的初始内容存储在 ROM/Flash。**
    > 
    > 

    所以：

    ```Plain Text
    Flash
               ROM
                │
                │ _sidata
                │
                │ 保存：
                │ g_count = 10
                ▼
            上电启动时复制
                │
                ▼
    RAM
    │
    ├── _sdata
    │
    │   .data
    │
    └── _edata
    ```

    因此三个符号的意义非常清楚：

    ```Plain Text
    _sidata
        = .data 初始镜像在 Flash 的起点
    
    _sdata
        = .data 在 RAM 的起点
    
    _edata
        = .data 在 RAM 的终点
    ```

    这三个符号就是 ***linker script*** 专门留给 ***startup\(真正运行的第一段代码\)*** 使用的。

- `.bss`

    > ```C
    > .bss :
    > {
    >     _sbss = .;
    > 
    >     *(.bss)
    >     *(.bss*)
    >     *(COMMON)
    > 
    >     _ebss = .;
    > } >RAM
    > ```
    > 
    > 

    > `.bss` 是：
    > 
    > **没有显式初始值，或者初始值为 0 的全局/静态变量。**
    > 
    > 例如：
    > 
    > ```Plain Text
    > int g_count;
    > static int flag;
    > ```
    > 
    > 按照 C 语言规则，它们启动后应该是：
    > 
    > ```Plain Text
    > g_count = 0
    > flag    = 0
    > ```
    > 
    > ---
    > 
    > 但注意 Flash 里没必要真的保存几万个 `0`。
    > 
    > 所以 linker 只告诉 startup：
    > 
    > ```Plain Text
    > _sbss
    >     ↓
    > .bss 起始地址
    > 
    > _ebss
    >     ↓
    > .bss 结束地址
    > ```
    > 
    > startup 启动时自己做：
    > 
    > ```Plain Text
    > 从 _sbss 开始
    >         ↓
    > 一路写 0
    >         ↓
    > 直到 _ebss
    > ```
    > 
    > 所以 `.data` 和 `.bss` 的启动逻辑完全不同：
    > 
    > ```Plain Text
    > .data
    > 有初始值
    >     ↓
    > Flash 保存初始值
    >     ↓
    > 启动时复制到 RAM
    > 
    > 
    > .bss
    > 初始值全为 0
    >     ↓
    > Flash 不保存一堆 0
    >     ↓
    > 启动时直接把 RAM 清零
    > ```
    > 
    > 这就是你以后经常会看到：
    > 
    > ```Plain Text
    > copy data
    > zero bss
    > ```
    > 
    > 的原因。
    > 
    > 

---

```C
MCU Reset
   │
   ▼
Reset_Handler
   │
   ├── 复制 .data
   │
   │   Flash
   │   _sidata
   │      │
   │      ▼
   │   RAM
   │   _sdata → _edata
   │
   │
   ├── 清零 .bss
   │
   │   RAM
   │   _sbss → _ebss
   │
   ├── SystemInit()
   │
   ▼
 main()
```

所以：

> **startup 并不知道 RAM 和 Flash 的具体地址。**
> 
> 

它只知道这些名字：

```Plain Text
_sidata
_sdata
_edata
_sbss
_ebss
_estack
```

而真正决定这些符号对应什么地址的是：

```Plain Text
STM32F446ZE.ld
```

这就是 linker script 和 startup 的分工。

一句话概括：

> **linker script 决定“东西在哪里”；startup 根据这些地址完成上电初始化。**
> 
> 

---

- `._user_heap_stack`

    ```C
    ._user_heap_stack :
    {
        . = ALIGN(8);
    
        PROVIDE ( end = . );
        PROVIDE ( _end = . );
    
        . = . + _Min_Heap_Size;
        . = . + _Min_Stack_Size;
    
        . = ALIGN(8);
    } >RAM
    ```

    前面我们看到：

    ```Plain Text
    _Min_Heap_Size  = 0x200;
    _Min_Stack_Size = 0x400;
    ```

    现在它们终于被使用了。

    可以理解为 linker 在 RAM 中预留检查空间：

    ```C++
    PROVIDE(end = .);
    // 如果外部没有定义 end，那么 linker 帮你定义一个 end symbol。
    // _end 与 end 是为了兼容不同 C 的实现
    // ----------------- 记录了当前 RAM 地址。-----------------
    
    
    预留 512B heap
          ↓
    预留 1024B stack
    ```

- `.preinit_array` 、`.init_array` 、`.fini_array`

    > 这里先不做过多理解，只用知道：
    > 
    > linker 把编译器生成的这些初始化函数表统一收集起来，并放进 ROM。
    > 
    > 

- `/DISCARD/`

    > ```JavaScript
    > /DISCARD/ :
    > {
    >     libc.a ( * )
    >     libm.a ( * )
    >     libgcc.a ( * )
    > }
    > // 表示某些匹配内容被丢弃。
    > 
    > .ARM.attributes 0 : { *(.ARM.attributes) }
    > // 是 ARM 目标文件的一些架构属性信息，现在也不用深挖。
    > ```
    > 
    > 

# *linker script*

```Markdown
MEMORY
    ↓
定义 STM32F446ZE 有哪些内存


SECTIONS
    ↓
决定各种内容放在哪里


.isr_vector
    ↓
向量表 → Flash


.text / .rodata
    ↓
代码和常量 → Flash


.data
    ↓
初始值存在 Flash
运行时放 RAM


.bss
    ↓
运行时放 RAM
启动时清零


_estack / _sidata / _sdata / ...
    ↓
把地址信息提供给 startup
```

建立一条关系：

```Plain Text
linker script
                     │
                     │定义地址和符号
                     ▼
_estack  _sidata  _sdata  _edata  _sbss  _ebss
                     ▲
                     │ 使用这些符号
                     │
                +----------+
                |  ***startup*** |
                +----------+
                     │
                     ▼
                SystemInit()
                     │
                     ▼
                   main()
```



