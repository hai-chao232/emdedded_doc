# startup\-Reset\_Handler

# Reset\_Handler

```Assembly language
Reset_Handler:  
  ldr   sp, =_estack      /* set stack pointer */
  
/* Call the clock system initialization function.*/
  bl  SystemInit  

/* Copy the data segment initializers from flash to SRAM */  
  ldr r0, =_sdata
  ldr r1, =_edata
  ldr r2, =_sidata
  movs r3, #0
  b LoopCopyDataInit

CopyDataInit:
  ldr r4, [r2, r3]
  str r4, [r0, r3]
  adds r3, r3, #4

LoopCopyDataInit:
  adds r4, r0, r3
  cmp r4, r1
  bcc CopyDataInit
  
/* Zero fill the bss segment. */
  ldr r2, =_sbss
  ldr r4, =_ebss
  movs r3, #0
  b LoopFillZerobss

FillZerobss:
  str  r3, [r2]
  adds r2, r2, #4

LoopFillZerobss:
  cmp r2, r4
  bcc FillZerobss
  
/* Call static constructors */
    bl __libc_init_array
/* Call the application's entry point.*/
  bl  main
  bx  lr    
.size  Reset_Handler, .-Reset_Handler
```

1. `ldr   sp, =_estack`

    > ***`/* set stack pointer */`***
    > 
    > 

    而 ***linker script*** 里有：`_estack = ORIGIN(RAM) + LENGTH(RAM);`

    即 `_estack` 定义为 RAM 的最高地址

    Cortex\-M 的栈是向低地址增长的，所以 `_estack` \-\-\-\-\> 栈顶

    ```Assembly language
    Cortex-M RAM
                  128 KB
    
    0x2002_0000  ───────────────────────  ← _estack
                  │                     │
                  │      Stack          │
                  │        ↓            │
                  │        ↓            │
                  │        ↓            │
                  │                     │
                  │                     │
                  │                     │
                  │                     │
    0x2000_0000  ───────────────────────
                  ↑
                  RAM 起始地址
    ```

    所以这里就是：

    > ```Plain Text
    > linker script
    >     定义 _estack = RAM 顶部地址
    >             ↓
    > startup
    >     把 _estack 装进 sp
    >             ↓
    > CPU 得到初始栈指针
    > ```
    > 
    > 也就是说，startup 并不知道 RAM 顶部到底是多少，
    > 
    > 它只认 `_estack` 这个符号；真正的地址由 linker script 决定。
    > 
    > 

2. `bl SystemInit`

    > ***`/* Call the clock system initialization function.*/`***
    > 
    > 

    > 这里调用：
    > 
    > ```Plain Text
    > system_stm32f4xx.c
    >     ↓
    > SystemInit()
    > ```
    > 
    > 所以启动链开始变成：
    > 
    > ```Plain Text
    > Reset
    >   ↓
    > Reset_Handler
    >   ↓
    > 设置 SP
    >   ↓
    > SystemInit()
    > ```
    > 
    > 这里有一个值得记住的点：
    > 
    > 这份 ST startup 是**先调用 ****`SystemInit()`****，再初始化 ****`.data/.bss`**
    > 
    > 所以以后如果修改 `SystemInit()`，最好不要依赖尚未初始化好的普通全局变量。
    > 
    > 

3. `.data`

    > ```Assembly language
    > /* Copy the data segment initializers from flash to SRAM */  
    > ldr r0, =_sdata
    > ldr r1, =_edata
    > ldr r2, =_sidata
    > movs r3, #0
    > b LoopCopyDataInit
    > 
    > CopyDataInit:
    >   ldr r4, [r2, r3]
    >   str r4, [r0, r3]
    >   adds r3, r3, #4
    > 
    > LoopCopyDataInit:
    >   adds r4, r0, r3
    >   cmp r4, r1
    >   bcc CopyDataInit
    >   
    > # **把 .data 段从 Flash 搬到 RAM** 
    > ```
    > 
    > 

    > 这些都是 ***linker script**** *定义好的 符号：
    > 
    > ```Assembly language
    > r0 = _sdata
    >      ↓
    > .data 在 RAM 中的起点
    > 
    > 
    > r1 = _edata
    >      ↓
    > .data 在 RAM 中的终点
    > 
    > 
    > r2 = _sidata
    >      ↓
    > .data 初始内容在 Flash 中的起点
    > ```
    > 
    > 

    ---

    > 后面就是搬移数据 **copy data** ：
    > 
    > ```Plain Text
    > Flash
    > _sidata
    >    │
    >    │ copy
    >    ▼
    > RAM
    > _sdata → _edata
    > ```
    > 
    > 

    ---

    > 比如：
    > 
    > ```C
    > int counter = 10;
    > 
    > /* ***********************
    >  *Flash：
    >  *保存“10”这个初始内容
    >  *
    >  *上电：
    >  *startup 把它复制到 RAM
    >  *
    >  *运行：
    >  *counter 真正存在 RAM 中
    >  ***************************** */
    > ```
    > 
    > 

4. `.bss`

    > ```Assembly language
    > /* Zero fill the bss segment. */
    >   ldr r2, =_sbss
    >   ldr r4, =_ebss
    >   movs r3, #0
    >   b LoopFillZerobss
    > 
    > FillZerobss:
    >   str  r3, [r2]
    >   adds r2, r2, #4
    > 
    > LoopFillZerobss:
    >   cmp r2, r4
    >   bcc FillZerobss
    >   
    > # 清零了 .bss
    > ```
    > 
    > 

    > ***linker script**** *定义好的 符号：
    > 
    > ```Assembly language
    > _sbss = .;    # .bss 起点
    > ... 
    > _ebss = .;    # .bss 终点
    > ```
    > 
    > 

    ---

    > 后面就是清零 \.bss 段 **zero bss：**
    > 
    > ```Assembly language
    > 往当前 RAM 地址写 0
    >         ↓
    > 地址 +4
    >         ↓
    > 继续
    > 
    > 直到 _sbss → _ebss
    > 全部清零
    > ```
    > 
    > 

    ---

    > 所以：
    > 
    > ```Plain Text
    > int flag;
    > static int count;
    > ```
    > 
    > 之所以进 `main()` 时天然是 `0`，并不是 RAM 上电后神奇地自动等于 0，而是：
    > 
    > **startup 清零了 ****`.bss`****。**
    > 
    > 

5. `bl __libc_init_array`

    > 这个和刚才*** linker script*** 里的：
    > 
    > ```Plain Text
    > .preinit_array
    > .init_array
    > ```
    > 
    > 开始对应起来了。
    > 
    > ---
    > 
    > - 它主要负责调用 C/C\+\+ 运行时需要的一些初始化函数
    > 
    > - 尤其是 C\+\+ 全局构造函数之类。
    > 
    > 我们目前写纯 C，可以先知道它是：
    > 
    > **进入 ****`main()`**** 前的 C/C\+\+ runtime 初始化步骤。**
    > 
    > 【暂时不用深挖。】
    > 
    > 

最后：

> `bl main`
> 
> 进入：
> 
> ```Plain Text
> int main(void)
> {
>     ...
> }
> ```
> 
> 

# 完整启动链：

```Assembly language
MCU Reset
   │
   ↓
CPU 从向量表取得 Reset_Handler
   │
   ↓
Reset_Handler
   │
   ↓
sp = _estack
   │
   ↓
SystemInit()
   │
   ↓
复制 .data
Flash(_sidata)
   │
   ↓
RAM(_sdata → _edata)
   │   
   ↓
清零 .bss
_sbss → _ebss
   │
   ↓
__libc_init_array()
   │
   ↓
main()
```

现在把三个文件放一起看，你会发现职责非常清楚：

```Plain Text
startup_stm32f446xx.s
    ↓
“启动过程做什么”


STM32F446ZE.ld
    ↓
“这些东西具体在哪”


system_stm32f4xx.c
    ↓
“芯片系统初始化怎么做”
```

---

# 总结

> 而 `platform/stm32f446ze/CMakeLists.txt` 的作用则是：
> 
> ```Plain Text
> 把这三样
> +
> CMSIS
> +
> CPU 编译参数
> +
> MCU 宏
> 
> 组合成：
> 
> platform_stm32f446ze
> ```
> 
> 所以现在 `platform` 这条链已经越来越完整：
> 
> ```Plain Text
> platform_stm32f446ze
> │
> ├── CMSIS Core
> ├── STM32F446 Device
> ├── startup
> ├── system
> ├── Cortex-M4F 编译参数
> └── linker script        ← 我们现在正准备正式接进去
> ```
> 
> 

---



