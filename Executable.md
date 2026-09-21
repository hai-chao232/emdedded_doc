# Executable

## 建立一个最小 executable

例如：

```Plain Text
experiments/
└── 001_gpio_output/
    ├── CMakeLists.txt
    └── main.c
```

但是第一版 `main.c` **还不会立刻写 GPIO**。我们会先做一个极简程序：

```Plain Text
int main(void)
{
    while (1)
    {
    }
}
```

目的只有一个：

> **第一次验证完整平台构建链。**
> 
> 

也就是：

```Plain Text
main.c
  +
startup
  +
system
  +
CMSIS
  +
linker script
  +
Cortex-M4F 参数
      ↓
arm-none-eabi-gcc
      ↓
成功生成 ELF
```

等这个 ELF 真正出来以后，我们再开始往 `main()` 里面加入 GPIO 内容。

这样如果 GPIO 代码之后出问题，我们就知道：

> **platform / startup / linker / toolchain** 已经验证过了，
> 
> 问题应该在 GPIO 实验本身。
> 
> 

## 创建第一个实验目录

在仓库根目录执行：

```Plain Text
mkdir -p experiments/001_gpio_output
```

创建：

```Plain Text
experiments/
└── 001_gpio_output/
    ├── CMakeLists.txt
    └── main.c
```

写最小 `main.c`：

```Plain Text
int main(void)
{
    while (1)
    {
    }
}
```

这里故意什么都不 `#include`，也不操作 GPIO。

因为我们现在只想回答：

> **一个最普通的 ****`main()`****，能不能依靠 ****`platform_stm32f446ze`**** 成功变成 STM32F446ZE 的 ELF？**
> 
> 

---

## 写实验自己的 `CMakeLists.txt`

> ***experiments/001\_gpio\_output/CMakeLists\.txt：***
> 
> 

```Plain Text
add_executable(exp001_gpio_output
    main.c
)

target_link_libraries(
    exp001_gpio_output
    PRIVATE
    platform_stm32f446ze
)

set_target_properties(
    exp001_gpio_output
    PROPERTIES
    SUFFIX ".elf"
)
```

现在重点看这句：

```Plain Text
target_link_libraries(
    exp001_gpio_output
    PRIVATE
    platform_stm32f446ze
)
```

前面理解过它，现在终于实际用上了。

它表面只有一句：

```Plain Text
exp001_gpio_output
    ↓
依赖 platform_stm32f446ze
```

但实际上意味着这个最简单的 `main.c` 自动获得：

```Plain Text
STM32F446xx
+
CMSIS include
+
Cortex-M4F 编译参数
+
startup_stm32f446xx.s
+
system_stm32f4xx.c
+
STM32F446ZE.ld
```

也就是说：

```Plain Text
main.c
   │
   │ 只负责“应用”
   ▼
exp001_gpio_output
   │
   │ 我要运行在 STM32F446ZE 上
   ▼
platform_stm32f446ze
   │
   ├── third_party 中选 startup
   ├── third_party 中选 CMSIS
   ├── third_party 中选 system
   ├── CPU 参数
   └── linker script
```

这就是我们前面搭 `platform` 的意义第一次真正体现出来。

---

## 把实验加入根 CMake

你根目录的 `CMakeLists.txt` 原本有：

```Plain Text
set(EXPERIMENTS
  # 001_gpio_output
  # 002_gpio_input
)
```

现在只启用第一个：

```Plain Text
set(EXPERIMENTS
  001_gpio_output
  # 002_gpio_input
)
```

后面的实验先不要开。

于是根 CMake 会执行：

```Plain Text
add_subdirectory(experiments/001_gpio_output)
```

整个构建树现在第一次真正变成：

```Plain Text
embedded-lab
│
├── common
│
├── platform
│   └── stm32f446ze
│
└── experiments
    └── 001_gpio_output
            ↓
      exp001_gpio_output
```

---

## 重新配置

执行：

```Plain Text
cmake --preset stm32f446ze
```

如果仍然：

```Plain Text
-- Configuring done
-- Generating done
```

说明 CMake 已经正式认识：

```Plain Text
exp001_gpio_output
```

这个 executable target。

但仍然还没真正编译。

---

## 第一次真正 build

执行：

```Plain Text
cmake --build build --target exp001_gpio_output -v
```

这里的 `-v` 很重要。

因为这是第一次，我们想直接看到 Ninja 最终调用了哪些：

```Plain Text
arm-none-eabi-gcc
```

命令。

这次 GCC 会真正处理：

```Plain Text
main.c
system_stm32f4xx.c
startup_stm32f446xx.s
        ↓
编译 / 汇编
        ↓
.o 文件
        ↓
STM32F446ZE.ld
        ↓
链接
        ↓
exp001_gpio_output.elf
```

这次如果报错，不要急着改

实际上，**第一次真正链接时出现问题是很有价值的**。

因为到目前为止：

```Plain Text
cmake --preset
```

只验证了 CMake 配置；现在第一次才会暴露：

```Plain Text
startup 是否完整
linker script 是否正确
C runtime 是否满足
链接参数是否够
标准库怎么处理
```

尤其我们使用的是 ST 的 startup，它里面有：

```Plain Text
bl __libc_init_array
```

而官方 linker script 又涉及 libc；

所以第一次 linker 真正运行后，

我们再根据**真实错误**决定是否需要给平台增加 `nano.specs / nosys.specs` 等配置，而不是现在提前猜。

## 大概率会报错

```Plain Text
undefined reference to `_exit'
undefined reference to `_close'
undefined reference to `_lseek'
undefined reference to `_read'
undefined reference to `_write'
undefined reference to `_sbrk'
```

**说明前面三步已经成功**：

```Plain Text
[1/4] startup_stm32f446xx.s    ✅ 汇编成功
[2/4] main.c                   ✅ 编译成功
[3/4] system_stm32f4xx.c       ✅ 编译成功
[4/4] 最终链接 ELF             ❌
```

所以现在的问题已经非常集中：

> **裸机程序没有操作系统，但 GCC 默认使用的 C 标准库在询问“操作系统服务在哪里？”**
> 
> 

例如：

```Plain Text
printf()
   ↓
C 标准库
   ↓
_write()
   ↓
正常 Linux 下：操作系统提供


但我们的 STM32：
   ↓
没有 Linux
没有操作系统
```

同理：

```Plain Text
malloc() → _sbrk()
exit()   → _exit()
read()   → _read()
```

- Arm GNU Toolchain 的 bare\-metal 环境使用 Newlib；

- STM32CubeIDE 对这种裸机工程也提供 `--specs=nosys.specs` 的“Minimal implementation”系统调用配置。

---

**为什么明明 ****`main()`**** 什么也没用，还是扯到了 libc？**

这里特别值得理解。

startup 中我们刚刚看过：

```Plain Text
bl __libc_init_array
bl main
```

也就是说，从 STM32 启动代码本身就已经进入了 C runtime 世界。

而你的最终链接命令现在是：

```Plain Text
arm-none-eabi-gcc
    ...
    startup.o
    system.o
    main.o
    ...
```

但没有告诉 GCC：

> “这是没有操作系统的 bare\-metal 程序，请给我裸机系统调用占位实现。”
> 
> 

于是默认 libc 被拉进来以后，就开始寻找 `_write/_read/_exit/...`。

---

**我们现在补两项**

回到：

```C
// platform/stm32f446ze/CMakeLists.txt

target_link_options(
    platform_stm32f446ze
    INTERFACE

    -mcpu=cortex-m4
    -mthumb
    -mfpu=fpv4-sp-d16
    -mfloat-abi=hard

    -T${STM32F446ZE_LINKER_SCRIPT}
)
```

在里面增加：

```Plain Text
--specs=nano.specs
--specs=nosys.specs
```

变成：

```Plain Text
target_link_options(
    platform_stm32f446ze
    INTERFACE

    -mcpu=cortex-m4
    -mthumb
    -mfpu=fpv4-sp-d16
    -mfloat-abi=hard

    --specs=nano.specs
    --specs=nosys.specs
)
```

ST 的典型 STM32 GCC 裸机链接配置也是同时使用 `nano.specs` 和 `nosys.specs`。

两个不要混：

```Plain Text
nano.specs
    ↓
选择体积更适合嵌入式的 Newlib Nano C 库配置


nosys.specs
    ↓
这是 bare-metal，没有 OS
给 _write / _read / _exit / _sbrk 等
提供最低限度的占位实现
```

因此现在：

```Plain Text
STM32        
  ↓
没有 OS
  ↓
nosys.specs
  ↓
“这些系统调用目前先用裸机 stub”
```

以后我们做串口 `printf` 时，完全可以真正实现：

```Plain Text
_write(...)
{
    // USART3 -> ST-Link VCP
}
```

那时 `printf()` 才真正有输出。

所以 `nosys.specs` 并不是说：

> “以后永远不能有 `_write()`。”
> 
> 

而是：

> **当前没有自己实现时，先给裸机一个最低限度的兜底。**
> 
> 

---

**`nano.specs`**** 为什么也现在加？**

复制的 ST linker script 末尾还有：

```Plain Text
/DISCARD/ :
{
    libc.a ( * )
    libm.a ( * )
    libgcc.a ( * )
}
```

而你当前错误中明确出现的是：

```Plain Text
.../libc.a(...)
```

STM32CubeIDE 的典型工程使用的是：

```Plain Text
--specs=nano.specs
```

从而采用面向嵌入式的 Newlib Nano 配置；

这和我们复制的 ST CubeIDE linker script 是匹配的一套构建环境。

所以既然我们选择了：

```Plain Text
ST CubeIDE 官方 linker script
```

就不要只复制半套规则，我们把它对应的 bare\-metal runtime 选择也补齐。





## 成功

意味着我们第一次真正打通了：

> ```C
> main.c
>   │
>   ├── C 编译
>   │
> system_stm32f4xx.c
>   │
>   ├── C 编译
>   │
> startup_stm32f446xx.s
>   │
>   ├── 汇编
>   │
>   ▼
> 三个 .o
>   │
>   │ + STM32F446ZE.ld
>   │ + Cortex-M4F 参数
>   │ + newlib-nano
>   │ + nosys
>   ▼
> exp001_gpio_output.elf
> ```
> 
> 

## 然后做第一次 ELF “体检”

- 先看大小：

    ```Shell
    /opt/arm-gnu-toolchain-15.3.rel1-x86_64-arm-none-eabi/bin/arm-none-eabi-size \
    build/experiments/001_gpio_output/exp001_gpio_output.elf
    
    # 如果我们的 shell 把工具链加入了 PATH，就可以直接使用：
    arm-none-eabi-size \
    build/experiments/001_gpio_output/exp001_gpio_output.elf
    ```

    你应该得到类似：

    ```Plain Text
    text    data     bss     dec     hex filename
    3508     100    1908    5516    158c build/experiments/001_gpio_output/exp001_gpio_output.elf
    ```

    这样理解：

    ```C
    text
        ≈ 代码 + 只读数据
        主要占 Flash 
        (程序的指令代码段大小（机器指令），通常存放在 Flash/ROM。约 3.4 KB。)
    
    data
        = 有初始值的可写数据
        Flash 要存一份初始值
        RAM 运行时也要占空间
        (编译时存放在 Flash，启动时拷贝到 RAM。这里是 100 字节。)
    
    bss
        = 零初始化/未初始化全局静态数据
        只占 RAM
        (启动时在 RAM 中清零。这里占用约 1.9 KB RAM。)
        
    // =====================================================================
    **// Flash 占用** = text + data = 3508 + 100 = **3608 字节 ≈ 3.5 KB**
    **// RAM 占用**   = data + bss  = 100 + 1908 = **2008 字节 ≈ 2 KB**
    ```

    这正好对应我们刚学过的 linker script。

---

- 然后再执行：

    ```Plain Text
    arm-none-eabi-objdump \
    -h build/experiments/001_gpio_output/exp001_gpio_output.elf
    ```

    这个特别值得看。

    我们应该可以在最终 ELF 里面亲眼找到 ELF 文件的 **段表**：

    ```C
    .isr_vector
    .text
    .rodata
    .data
    .bss
    
    // ===================================================================
    // Idx：段索引号，表示这是段表里的第 几个 section。
    // VMA：程序运行时，这个 section 应该在哪里。
    // LMA：固件烧录后，这个 section 的初始化内容存在哪里。
    // File off：在 ELF 文件里的偏移量
    // Algn：对齐要求
    // 属性 (CONTENTS, ALLOC, LOAD, READONLY, DATA)
    //        **CONTENTS**：这个段在 ELF 文件里有内容。
    **//        ALLOC**：运行时需要分配内存。
    **//        LOAD**：需要加载到目标内存。
    **//        READONLY**：只读。
    **//        DATA**：数据段类型。   
    Sections:
    Idx Name          Size      VMA       LMA       File off  Algn
      0 .isr_vector   000001c4  08000000  08000000  00001000  2**0
                      CONTENTS, ALLOC, LOAD, READONLY, DATA
      1 .text         00000b80  080001c4  080001c4  000011c4  2**2
                      CONTENTS, ALLOC, LOAD, READONLY, CODE
      2 .rodata       00000060  08000d44  08000d44  00001d44  2**2
                      CONTENTS, ALLOC, LOAD, READONLY, DATA
      3 .ARM.extab    00000000  08000da4  08000da4  00001da4  2**0
                      CONTENTS, ALLOC, LOAD, READONLY, DATA
      4 .ARM          00000008  08000da4  08000da4  00001da4  2**2
                      CONTENTS, ALLOC, LOAD, READONLY, DATA
      5 .preinit_array 00000000  08000dac  08000dac  00002064  2**0
                      CONTENTS, ALLOC, LOAD, DATA
      6 .init_array   00000004  08000dac  08000dac  00001dac  2**2
                      CONTENTS, ALLOC, LOAD, READONLY, DATA
      7 .fini_array   00000004  08000db0  08000db0  00001db0  2**2
                      CONTENTS, ALLOC, LOAD, READONLY, DATA
      8 .data         00000064  20000000  08000db4  00002000  2**2
                      CONTENTS, ALLOC, LOAD, DATA
      9 .bss          00000170  20000064  08000e18  00002064  2**2
                      ALLOC
     10 ._user_heap_stack 00000604  200001d4  08000e18  000021d4  2**0
                      ALLOC
     11 .ARM.attributes 00000030  00000000  00000000  00002064  2**0
                      CONTENTS, READONLY
     12 .debug_line_str 000000fb  00000000  00000000  00002094  2**0
                      CONTENTS, READONLY, DEBUGGING, OCTETS
     13 .comment      00000046  00000000  00000000  0000218f  2**0
                      CONTENTS, READONLY
     14 .debug_frame  0000060c  00000000  00000000  000021d8  2**2
                      CONTENTS, READONLY, DEBUGGING, OCTETS
    ```

    也就是说，之前 linker script 里那些：

    ```Plain Text
    .isr_vector { ... } >ROM
    .text       { ... } >ROM
    .data       { ... } >RAM AT>ROM
    .bss        { ... } >RAM
    ```

    已经不再只是理论——**它们真的会出现在生成出来的 ELF 里。**

    **并且都符合我们的预期：**

    > 先看 `size`
    > 
    > ```Plain Text
    > text    data    bss     dec
    > 3508     100   1908    5516
    > ```
    > 
    > 这里千万不要简单理解成：
    > 
    > ```Plain Text
    > Flash = text
    > RAM   = data + bss
    > ```
    > 
    > 更准确的是：
    > 
    > ```Plain Text
    > Flash 占用
    > ≈ text + data
    > = 3508 + 100
    > = 3608 Bytes
    > ```
    > 
    > ---
    > 
    > 为什么 `data` 还占 Flash？
    > 
    > 因为我们刚刚学过：
    > 
    > ```Plain Text
    > .data
    > 运行时 → RAM
    > 
    > 但初始化值
    >         → 必须先存在 Flash
    > ```
    > 
    > 所以 `data = 100` 同时意味着：
    > 
    > ```Plain Text
    > Flash：需要保存 100 B 的初始化镜像
    > RAM：  运行时需要 100 B
    > ```
    > 
    > ---
    > 
    > 而 RAM 大致是：
    > 
    > ```Plain Text
    > data + bss
    > = 100 + 1908
    > = 2008 Bytes
    > ```
    > 
    > 但这里马上有一个特别有意思的地方。
    > 
    > 

    ---

    > 看 `objdump`：
    > 
    > ```Bash
    > .bss
    > Size = 00000170
    > 
    > # 换算：0x170 = 368 Bytes
    > ```
    > 
    > 那为什么 `size` 却说：`bss = 1908 Bytes`
    > 
    > 这是因为下面还有：
    > 
    > ```Bash
    > ._user_heap_stack
    > Size = 00000604
    > 
    > # 换算：0x604 = 1540 Bytes
    > ```
    > 
    > 两者相加：
    > 
    > ```Bash
    > 368 + 1540 = 1908
    > 
    > # **正好等于 size 输出的 bss。**
    > 
    > # 所以：
    > arm-none-eabi-size 的 bss 1908
    >         =
    > 真正的 .bss           368 B
    > +
    > ._user_heap_stack     1540 B
    > ```
    > 
    > 

    ---

    > 这又直接对应我们 linker script 里的：
    > 
    > ```Plain Text
    > _Min_Heap_Size  = 0x200;
    > _Min_Stack_Size = 0x400;
    > ```
    > 
    > 也就是说你第一次亲眼看到了：
    > 
    > linker script 里面给 heap / stack 留的空间，最后真的影响了 ELF 的 RAM 使用统计。
    > 
    > 但是我们预留的是：`0x200 + 0x400 = 0x600 = 1536 B`
    > 
    > 而`objdump` 里面给的是：`0x604 = 1540 B`
    > 
    > 因为在 **linker script** 里面这个 section 里面一开始还有：
    > 
    > ```Plain Text
    > . = ALIGN(8);
    > ```
    > 
    > 而这个 section 起点是：
    > 
    > ```Plain Text
    > 0x200001d4
    > ```
    > 
    > 它为了变成 8 字节对齐，要先走到：
    > 
    > ```Plain Text
    > 0x200001d8
    > ```
    > 
    > 于是多了：***4 Bytes**** *
    > 
    > 这说明 linker script 里的 `ALIGN()` 也真的体现到最终布局里了。
    > 
    > 

    ---

    > **再看最重要的 ****`VMA / LMA`**
    > 
    > 你这里最值得盯着看的其实是：
    > 
    > ```Plain Text
    > Idx Name      Size       VMA        LMA
    > 8   .data     00000064   20000000   08000db4
    > ```
    > 
    > 这个输出几乎就是我们之前 `.data` 理论的“实验证据”。
    > 
    > ```Plain Text
    > VMA = 0x20000000
    > ```
    > 
    > VMA 可以先理解为：
    > 
    > **程序运行时，这个 section 应该在哪里。**
    > 
    > ```Bash
    > 0x20000000
    >     ↓
    > RAM 起始地址
    > 
    > # 所以：
    > #     .data 运行时在 RAM
    > ```
    > 
    > LMA 可以先理解成：
    > 
    > **固件烧录后，这个 section 的初始化内容存在哪里。**
    > 
    > ```Plain Text
    > .data
    > 
    > 运行地址 VMA
    >     ↓
    > 0x20000000 RAM
    > 
    > 加载地址 LMA
    >     ↓
    > 0x08000db4 Flash
    > ```
    > 
    > 这就是 linker script：
    > 
    > ```Plain Text
    > .data
    > {
    >     ...
    > } >RAM AT> ROM
    > ```
    > 
    > 真实生成出来的效果。
    > 
    > 于是 startup 做的：
    > 
    > ```Plain Text
    > _sidata
    >    ↓
    > Flash
    > 
    > 复制
    > 
    > _sdata → _edata
    >    ↓
    > RAM
    > ```
    > 
    > 现在已经完全能对应上。
    > 
    > 

    ---

    > `.bss` 就更漂亮了
    > 
    > ```Plain Text
    > .bss
    > Size = 0x170
    > VMA  = 0x20000064
    > ```
    > 
    > 它在：`RAM`
    > 
    > 符合：
    > 
    > ```Plain Text
    > .bss
    > {
    >     ...
    > } >RAM
    > ```
    > 
    > 但更重要的是，你看它下面的属性：
    > 
    > ```Plain Text
    > .bss 有：
    >     ALLOC
    >     
    > .data 有：
    >     CONTENTS, ALLOC, LOAD, DATA
    > ```
    > 
    > `.bss` **没有 ****`CONTENTS`****，也没有 ****`LOAD`**。因为：
    > 
    > ```Plain Text
    > .data
    >         ↓
    > 里面真的有初始内容
    > 例如：
    > int x = 10;
    > 
    > 所以 Flash 固件里需要保存“10”
    > 
    > 
    > .bss
    >         ↓
    > 不需要保存初始内容
    > 例如：
    > int x;
    > 
    > 只需要启动时把 RAM 清零
    > ```
    > 
    > 所以：
    > 
    > `.bss` 占 RAM，但是它不需要占用对应大小的 Flash 文件内容。
    > 
    > 这就是为什么 linker/startup 要专门清 `.bss`。
    > 
    > 

    ---

    > 向量表也准确地到了 Flash 最前面
    > 
    > ```Bash
    > .isr_vector
    > VMA = 08000000
    > LMA = 08000000
    > 
    > # 就是：
    > #    STM32 Flash 起始地址
    > #    0x08000000
    > ```
    > 
    > 而 linker script：
    > 
    > ```Plain Text
    > .isr_vector :
    > {
    >     KEEP(*(.isr_vector))
    > } >ROM
    > ```
    > 
    > 又是 `SECTIONS` 里的第一个 ROM section。
    > 
    > 所以：
    > 
    > ```Plain Text
    > Flash
    > 0x08000000
    >     │
    >     ├── .isr_vector
    >     │
    >     ├── .text
    >     │
    >     ├── .rodata
    >     │
    >     ├── ...
    >     │
    >     └── .data 的初始镜像
    > ```
    > 
    > 这非常重要，因为 Cortex\-M Reset 时需要从向量表取得：
    > 
    > ```Plain Text
    > 初始 MSP
    > Reset_Handler
    > ```
    > 
    > 

    ---

    > `.text` / `.rodata` 也完全符合预期
    > 
    > ```Plain Text
    > .text
    > VMA = 0x080001c4
    > 
    > .rodata
    > VMA = 0x08000d44
    > ```
    > 
    > 全部都是：
    > 
    > ```Bash
    > 0x080xxxxx
    > 
    > # 说明它们全部位于 Flash。
    > ```
    > 
    > 所以现在真实 ELF 基本就是：
    > 
    > ```Plain Text
    > FLASH 0x08000000
    > │
    > ├── .isr_vector
    > │   0x08000000
    > │
    > ├── .text
    > │   0x080001c4
    > │
    > ├── .rodata
    > │   0x08000d44
    > │
    > ├── ARM runtime sections
    > │
    > └── .data 初始化镜像
    >     0x08000db4
    > 
    > 
    > RAM 0x20000000
    > │
    > ├── .data
    > │   0x20000000
    > │   size = 100 B
    > │
    > ├── .bss
    > │   0x20000064
    > │   size = 368 B
    > │
    > └── ._user_heap_stack
    >     0x200001d4
    > ```
    > 
    > 这几乎就是 linker script 在现实里的展开图。
    > 
    > 

    ---

    > 这些数字也特别漂亮
    > 
    > ```Bash
    > # .data：
    > VMA = 0x20000000
    > Size = 0x64
    > 
    > # 那么下一块 RAM 理论上就是：
    > 0x20000000 + 0x64 = 0x20000064
    > 
    > # 而 .bss 恰好：
    > VMA = 0x20000064
    > ```
    > 
    > 再算：
    > 
    > ```Bash
    > # .bss 起点 = 0x20000064
    > # .bss 大小 = 0x170
    > 0x20000064 + 0x170 = 0x200001d4
    > 
    > # 而下一段 ._user_heap_stack 正好：
    > VMA = 0x200001d4
    > ```
    > 
    > 所以 RAM 真实排列就是严丝合缝的：
    > 
    > ```Plain Text
    > 0x20000000
    >     ↓
    > .data
    >     ↓ 0x64 bytes
    > 0x20000064
    >     ↓
    > .bss
    >     ↓ 0x170 bytes
    > 0x200001d4
    >     ↓
    > ._user_heap_stack
    > ```
    > 
    > 这就是 linker 真正在做“摆放”。
    > 
    > 

---

- 最后再看看几个我们刚学过的符号：

    ```Plain Text
    arm-none-eabi-nm \
    -n build/experiments/001_gpio_output/exp001_gpio_output.elf \
    | grep -E 'Reset_Handler|main|_estack|_sidata|_sdata|_edata|_sbss|_ebss'
    ```

    输出：

    ```Bash
    080001cc T _mainCRTStartup
    08000240 T main
    080003f4 W Reset_Handler
    08000db4 A _sidata
    20000000 D _sdata
    20000064 D _edata
    20000064 B _sbss
    200001d4 B _ebss
    20020000 R _estack
    ```

    现在 startup 那几句已经全部有真实地址了：

    ```Bash
    ldr sp, =_estack✅
                ↓
            0x20020000
    
    _sidata✅
        ↓
    0x08000db4
        │
        │ copy
        ▼
    0x20000000  _sdata✅
        ↓
    0x20000064  _edata✅
    
    
    0x20000064  _sbss✅
        │
        │ zero
        ▼
    0x200001d4  _ebss✅
    ```

    另外：

    > ```Plain Text
    > 080003f4 W Reset_Handler
    > ```
    > 
    > 也没问题。`W` 表示 **weak symbol**。ST 官方这份 startup 本身就明确写了：
    > 
    > ```Plain Text
    > .weak Reset_Handler
    > ```
    > 
    > 所以这是 ST 原始启动文件的设计，不是我们的链接出了问题。
    > 
    > 

    而：

    > ```Plain Text
    > 08000240 T main
    > 080003f4 W Reset_Handler
    > ```
    > 
    > 也不要因为 `main` 地址比 `Reset_Handler` 小就觉得执行顺序反了。
    > 
    > **Flash 中地址排列顺序 ≠ CPU 的执行顺序。**
    > 
    > CPU Reset 后通过向量表进入：
    > 
    > ```Plain Text
    > Reset_Handler
    >       ↓
    > ...
    >       ↓
    > bl main
    > ```
    > 
    > 所以还是 `Reset_Handler → main`。
    > 
    > 

    但 `_mainCRTStartup` 值得处理

    > ```Bash
    > 080001cc T _mainCRTStartup
    > ```
    > 
    > 这是我们原本**不需要**的。
    > 
    > 原因是我们现在实际上有两套“启动相关东西”被链接进 ELF：
    > 
    > ```Plain Text
    > 我们明确提供的：
    > startup_stm32f446xx.s
    >         ↓
    > Reset_Handler        ← 真正要用的
    > 
    > 
    > GCC 默认 startup files
    >         ↓
    > _mainCRTStartup      ← 目前也被带进来了
    > ```
    > 
    > 因为我们是通过：
    > 
    > ```Plain Text
    > arm-none-eabi-gcc
    > ```
    > 
    > 来完成最终链接，而目前没有告诉 GCC：
    > 
    > “启动代码我自己已经提供了，不要再自动加入系统默认 startup files。”
    > 
    > GCC 官方的 `-nostartfiles` 就是专门解决这个问题的：
    > 
    > **链接时不使用标准系统 startup files，但标准库仍然正常使用。** 
    > 
    > 我们的裸机 STM32 已经明确拥有：
    > 
    > ```Plain Text
    > startup_stm32f446xx.s
    >         ↓
    > Reset_Handler
    >         ↓
    > SystemInit
    >         ↓
    > .data/.bss
    >         ↓
    > __libc_init_array
    >         ↓
    > main
    > ```
    > 
    > 所以 GCC 默认的 `_mainCRTStartup` 是多余的。
    > 
    > 在 `platform/stm32f446ze/CMakeLists.txt` 中再加一项，把链接参数调整成：
    > 
    > ```Bash
    > target_link_options(
    >     platform_stm32f446ze
    >     INTERFACE
    > 
    >     -mcpu=cortex-m4
    >     -mthumb
    >     -mfpu=fpv4-sp-d16
    >     -mfloat-abi=hard
    > 
    >     -nostartfiles
    > 
    >     --specs=nano.specs
    >     --specs=nosys.specs
    > )
    > 
    > # 去掉 GCC 默认 startup（_mainCRTStartup）
    > # 代码体积也会下降
    > ```
    > 
    > 这里三个东西职责不要混：
    > 
    > ```Plain Text
    > -nostartfiles
    >     ↓
    > 不要 GCC 默认 startup
    > 因为我们自己有 STM32 startup
    > 
    > 
    > nano.specs
    >     ↓
    > 使用适合嵌入式的 Newlib Nano 配置
    > 
    > 
    > nosys.specs
    >     ↓
    > 这是裸机，没有 OS
    > 给系统调用提供最低限度 stub
    > ```
    > 
    > 

    直接 “把 GCC startup 全砍掉” 不可取

    > 现在的 startup 里有：
    > 
    > ```Plain Text
    > bl __libc_init_array
    > ```
    > 
    > 而 Newlib 的 `__libc_init_array()` 在当前配置下还会调用：`_init()`
    > 
    > 原先 `_init` 是由 GCC 默认 startup files 中的启动片段提供的，我们加了：
    > 
    > ```Plain Text
    > -nostartfiles
    > ```
    > 
    > 以后，GCC 官方定义就是“链接时不使用标准系统 startup files”。
    > 
    > 于是变成：
    > 
    > ```Plain Text
    > STM32 startup
    >     ↓
    > __libc_init_array()
    >     ↓
    > 需要 _init
    >     ↓
    > -nostartfiles 把提供它的启动片段也去掉了
    >     ↓
    > undefined reference to _init
    > ```
    > 
    > **不要自己补一个假的 ****`_init()`**** 来糊住这个问题**。
    > 
    > 对我们当前阶段，更合理的方案是：
    > 
    > - 保留 GCC/Newlib 必要的 runtime startup 片段，
    > 
    > - 但用 linker garbage collection 把真正没有用到的 `_mainCRTStartup` 等代码清掉。
    > 
    > 这也更接近 STM32CubeIDE 实际生成的 GCC 链接方式：
    > 
    > - 它通常使用 `--specs=nosys.specs`、
    > 
    > - `--specs=nano.specs`、
    > 
    > - `--gc-sections`，
    > 
    > - 而不是 `-nostartfiles`。
    > 
    > 

    解决方案：

    > 首先把：`-nostartfiles` 删除
    > 
    > 然后在 `target_compile_options()` 里增加：
    > 
    > ```Plain Text
    > target_compile_options(
    >     platform_stm32f446ze
    >     INTERFACE
    > 
    >     -mcpu=cortex-m4
    >     -mthumb
    >     -mfpu=fpv4-sp-d16
    >     -mfloat-abi=hard
    > 
    >     -ffunction-sections⭐
    >     -fdata-sections⭐
    > )
    > ```
    > 
    > 它们的作用是让编译器尽量：
    > 
    > ```Plain Text
    > 每个函数 → 自己的 section
    > 每份数据 → 自己的 section
    > ```
    > 
    > 例如原来可能都是：
    > 
    > ```Plain Text
    > .text
    > ```
    > 
    > 现在可以变成：
    > 
    > ```Plain Text
    > .text.main
    > .text.SystemInit
    > .text.some_function
    > ...
    > ```
    > 
    > 这样 linker 才能更精细地判断：
    > 
    > **哪个函数真正没用，可以删掉。**
    > 
    > ---
    > 
    > 然后链接参数添加：
    > 
    > ```Plain Text
    > target_link_options(
    >     platform_stm32f446ze
    >     INTERFACE
    > 
    >     -mcpu=cortex-m4
    >     -mthumb
    >     -mfpu=fpv4-sp-d16
    >     -mfloat-abi=hard
    > 
    >     --specs=nano.specs
    >     --specs=nosys.specs
    > 
    >     -Wl,--gc-sections⭐
    > )
    > ```
    > 
    > 这里：`-Wl,--gc-sections` 意思是：
    > 
    > ```Plain Text
    > GCC
    >  ↓
    > 把 --gc-sections 参数传给 linker
    >  ↓
    > 删除没有被使用的 section
    > ```
    > 
    > 于是我们的目标变成：
    > 
    > ```Plain Text
    > 默认 runtime startup files
    >         ↓
    > 仍然存在
    >         ↓
    > 因此 _init 等 runtime 支持正常 ✅
    > 
    > 但是
    > 
    > 没被 Reset_Handler 启动链使用的代码
    > 例如可能的 _mainCRTStartup
    >         ↓
    > --gc-sections
    >         ↓
    > 从最终 ELF 清除
    > ```
    > 
    > 

    ---

    最后的输出：

    ```Assembly language
    haichao@haichao-ThinkPad-E480:/data/10_WorkArchive/embedded-lab$ /opt/arm-gnu-toolchain-15.3.rel1-x86_64-arm-none-eabi/bin/arm-none-eabi-nm -n build/experiments/001_gpio_output/exp001_gpio_output.elf | grep -E 'Reset_Handler|main|_estack|_sidata|_sdata|_edata|_sbss|_ebss'
    08000250 T main
    08000278 W Reset_Handler
    08000334 A _sidata
    20000000 D _edata
    20000000 B _sbss
    20000000 D _sdata
    2000001c B _ebss
    20020000 R _estack
    haichao@haichao-ThinkPad-E480:/data/10_WorkArchive/embedded-lab$ arm-none-eabi-size build/experiments/001_gpio_output/exp001_gpio_output.elf 
       text    data     bss     dec     hex filename
        820       0    1568    2388     954 build/experiments/001_gpio_output/exp001_gpio_output.elf
    ```

