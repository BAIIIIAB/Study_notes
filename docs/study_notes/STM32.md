# STM32裸机开发

[TOC]



1.   认识到了嵌入式开发过程中**模块化设计、层次化设计**的重要性。
2.   寄存器编程更偏底层一点，虽然运行速度可能稍有提高，但是不利于后续开发及维护。
3.   优秀的编程习惯会使整个项目的开发事半功倍。



## 知识铺垫

1.   **单片机入门(HAL)**

     简单、快速，实际上工作中涉及单片机编程时，也提倡使用HAL库。对于学习来说，HAL封装了很多技术细节，对技术成长帮助不大。

  可能接触不到这些知识：

   重定位、代码段数据段BSS段、位置无关码、相对跳转、绝对跳转、 设置栈、中断上下文、保存/恢复中断现场、ARM架构。

这些知识，是单片机的核心，学习了它，有助于在RTOS领域发展。

2.   **单片机深入(基于寄存器)**

​    抛开HAL库，从芯片手册开始，自己写出一切代码。   **注意**：工作中绝对不建议这样做，但是学习时，这才能学到更深刻的知识。

​    这些知识，也是后续学习RTOS、学习Linux的u-boot的必备知识。

3.   **拓展：不学单片机，直接上手Linux：**

​    如果你有一定的硬件基础，或者对硬件操作不感兴趣，那么可以不学单片机。

​    Linux驱动 = 面向对象的编程思想 + 良好的程序框架 + 硬件操作(这就跟单片机类似)

​    如果你要学习Linux驱动开发，单片机虽然是基础，但是也可以在学习驱动过程中掌握。

### 学习内容

-   既使用keil，也使用GCC编译器，让单片机开发能无缝切换到Linux开发。

-   单片机的课程大部分是基于HAL库的，太简单，无法切换到各类RTOS、u-boot

-   实际上，如果掌握了Linux，掌握了u-boot，任何单片机程序都易如反掌。

-   从开发板上电的第一条指令开始，讲解整个程序涉及的一切知识，包括但不限于：  重定位、代码段数据段BSS段、位置无关码、相对跳转、绝对跳转、设置栈、中断上下文、保存/恢复中断现场、ARM架构、I2C协议与编程、SPI协议与编程、LCD、触摸屏、各类外设编程。

    **这些是RTOS的基础，也是Linux的u-boot的基础，有助于学习Linux驱动。**

而掌握一款MCU的开发需要重点关注四大模块：**时钟复位、中断异常、存储映射和外设寄存器组。**



## 嵌入式系统的启动

**该部分内容请结合SRAM文档中存储器的分类学习**

嵌入式系统支持多种启动方式。XIP设备启动、非XIP设备启动等等。比如：Nor Flash、SD卡、SPI Flash, 甚至支持UART、USB、网卡启动。（所谓XIP，全称为execute in Place，本地执行，可以不用将代码拷贝到内存，而是直接在代码的存储空间运行）

但上述设备中，很多都不是XIP设备，CPU无法直接执行里面的代码，那为什么说**CPU可以从非XIP设备启动**呢？答案是：上电后，CPU运行的第1条指令、第1个程序，位于片内ROM（Read-only memory）中，它是XIP设备。（该程序使用C语言来写）
       这个程序会执行必要的初始化，比如：

-   初始化硬件、设置时钟、设置内存（DDR需要初始化才能使用）；
-   初始化其他硬件，比如看门狗、SD卡等
-   再从"非XIP设备"中把程序读到内存；

-   最后启动上面的程序。

**问题1：**从外设把程序复制到内存的过程中，支持那么多的启动方式，怎么选择呢？

-   通过跳线，选择某个设备；
-   通过跳线，选择一个设备列表，按列表顺序逐个尝试
-   不让客户选择，按固定顺序逐个尝试

**问题2：**内存那么大，把程序从SD卡等设备，复制到内存哪个位置？复制多长？

-   烧写在SD卡等设备上的程序，含有一个头部信息，里面指定了内存地址和长度；
-   不给客户选择，程序被复制到内存固定的位置，长度也固定。

**问题3**：程序在SD卡上是如何存储的？

-    原始二进制(raw bin)；
-    作为一个文件保存在分区里；

对于复杂的设备，芯片内部都集成了相应的控制器。CPU只需要专注于逻辑运算即可。（上述提到的是Soc芯片，与PC主板作区分）

与PC类比：

![image-20230403160356499](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304031603548.png)

## 下载方式的区别

![l1-2](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202303211646885.png)

## 嵌入式开发的三大方向

1.   裸机程序

2.   运行RTOS的MCU项目

3.   运行Android/Linux的MPU项目

     开发成本不一样。

     **裸机程序**用到的外设模块较少，任务调度不频繁，实时性要求不高。

     **RTOS项目**整体比较复杂，应用层和驱动层区分不是特别明显（可以自己仿照编写出类Linux项目框架）特别强调了实时性与任务调度，可使用一些封装好的通讯协议等等。

     **Linux项目**应用层和驱动层划分明显，功能更加复杂，相较于RTOS项目来说更方便裁剪与移植，可以修改协议，自由度更高。

     

### MCU、MPU、AP

​		MCU开发人员需要具备C语言、数字电子技术、模拟电子技术、微机原理等相关理论基础，还可能需要 电路板绘制等专业技能。涉及电子信息工程、自动化、测量控制等相关专业。 

​		MCU开发主要涉及的内容包含：GPIO、UART、I2C、SPI、LCD等外设接口，时钟、中断、定时器、 ADC/DAC、看门狗等内部资源，以及RTOS(FreeRTOS、RT-Thread、uCOS、LiteOS等)嵌入式实时操作系统。 

​		MPU开发人员需要具备、数据结构、操作系统、计算机网络等相关理论基础。涉及计算机科学、软件工 程、物联网等相关专业。 MPU开发主要涉及的内容包含：Bootload移植、Linux内核移植、Linux设备驱动开发（GPIO、UART、 I2C、SPI等）、Linux应用开发（文件I/O、多任务编程、进程间通信、网络编程、Qt界面设计等），甚至包括Android驱动、Android应用编程。 

​	下图列举MCU开发和MPU开发的一些区别。MPU开发通常没有仿真器，也没有集成开发环境IDE 工具，但MPU资源丰富，反而下载方式比MCU丰富。MCU可以使用仿真器在线调试，也可以使用串口打印调试，而MPU一般只能使用串口调试打印。MCU的启动过程比较简单，而MPU的复杂很多，就Bootloader部分就相当于一个大型MCU开发工程。两者相互补充，占据了嵌入式设备的大部分份额。

![image-20230322124948999](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202303221249109.png)



MCU将CPU，RAM，ROM, I/O，中断系统，定时器等等外设资源集中到一个芯片上，而MPU必须外接RAM和FLASH电源等电路。

**AP**（Application Processors）相较于MCU，其内部集成了更多的模块，比如用于数据处理的DSP、用于图形显示的GPU，甚至有多个处理器。它还可以外接容量更大的内存（通过DDR）与Flash。

#### 拓展：DSP与FPGA

DSP快速进行数字信号处理，采用特殊软硬件结构：

**哈弗结构、多级流水线、专用的硬件乘法器，特殊的DSP指令**，使得DSP芯片在计算处理上远超同主频的MCU或MPU。

适用领域：调制/解调，数据加密/解密，图形处理，数字滤波，音频处理等计算密集型场景。

DSP功能比较单一，一般用不到RTOS。

**FPGA**开发成本更高，主要使用verilog进行开发。优势：**高速，灵活**。

某些通信领域需要处理高速的通信协议，同时通信协议随时有可能修改，不适合做成专门的芯片，这时FPGA优势就很突出。

#### 复杂嵌入式架构

例如“MPU + FPGA”, "MPU + DSP", "MCU + FPGA", "MCU + DSP"等。

适用场景：

**a. 控制、显示、通信用MCU或MPU**

**b. 通信和数据处理算法用DSP**

**c. 大量数据处理和特定实现用FPGA**



### 哈弗架构与冯诺依曼架构

CPU架构可以分为哈弗架构与冯诺伊曼架构。哈弗架构中指令与数据分开存放，CPU可以同时读入指令、读写数据。冯诺伊曼架构中指令、数据混合存放，CPU依次读取指令、读写数据，不可同时操作指令和数据。

ARM公司的芯片，ARM7及之前的芯片是冯诺伊曼架构，ARM7之后使用“改进的哈弗架构”。“改进的哈弗结构”如下所示：

![image-20230403152304558](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304031523685.png)

在“改进的哈弗架构”里，指令和数据在外部存储器中混合存放；CPU运行时，从指令cache中获得指令，从数据cache中读写数据。



### STM32最小系统

用最少的电路组成单片机可以工作的系统。电路组成：电源电路，时钟电路，复位电路，调试/下载电路，启动选择电路（STM32需要）。

启动选择电路：不同于51系列单片机只能从内置存储器读取数据启动，STM32可以从从内置存储器启动（默认），可以从系统存储器（用于从USART1 下载程序），可以从内部SRAM启动（掉电消失，可用于调试），因此需要选择。

**选择启动方式：**

1.   通常使用主存储器启动；

2.   系统存储器启动，实现从串口下载程序逐渐被淘汰（STM32高端的MCU已不支持）。

3.   SRAM启动无必要。烧写次数：SRAM > FLASH >> 用户实际烧写次数。



### 原理图

步骤：首先要保证最小系统功能正常，其次再实现与各个外设模块的控制或数据传输。暂时先理清原理图板块、网络标号；熟悉常见元件，理解电阻、电容属性。

![image-20230322125400440](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202303221254487.png)





### STM32结构

#### 总线结构

![image-20230322154543197](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202303221545160.png)

 **System总线**：用于访问指令、数据以及调试模块接口；

 **Bus matrix（总线矩阵）**：用于总线之间的访问优先级管理控制；

**APB总线**：用于外设接口的数据传输；ARM公司推出AMBA片上总线结构，该总线主要包含先进高 速总线（Advanced High-speed Bus，AHB）和先进外设总线（Advanced Peripheral Bus，APB），分别连接 高速设备和低速设备。基于这个总线结构，ICode、Dcode、System Bus都是AHB总线。这里AHB系统总线经 过两个AHB-APB桥转换成了两个APB总线。APB1上挂接有DAC、UART等外设，其最高频率可达36MHz； APB2上挂接有ADC、GPIO等外设，其最高频率可达72MHz。 

**在MCU每次复位后，所有的外设时钟都会默认处于关闭状态。**因此，在使用外设前需要操作复位和时 钟寄存器(Reset and Clock Control，RCC)开启所需外设的时钟

#### 存储结构

通过存储器地址访问外设的方式，称为**存储器地址映射**

![image-20230322155208015](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633767.png)

![1](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633024.jpeg)



#### STM32寄存器

ARM Cortex-M3微处理器的内部寄存器，又分为普通寄存器和特殊功能寄存器。

![image-20230322160205245](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633835.png)

-    R0-R12（General-Purpose Registers）：用于数据操作的32位通用寄存器；一些16位的Thumb指令， 只能访问低寄存器（R0~R7）； 

-   R13（Stack Pointers，SP）Cortex-M3包含两个堆栈指针寄存器；同一时刻只能看到其中一个；

     主堆栈指针寄存器（Main Stack Pointer，MSP）：操作系统（OS）内核和异常处理程序使用的 默认堆栈指针； 

     进程堆栈指针寄存器（Process Stack Pointer，PSP）：用于用户代码；

-    R14（Link Register，LR）：链接寄存器；调用子例程时，返回地址将存储在链接寄存器中。

![image-20230322160532299](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633674.png)

-   xPSR（Program Status registers）：程序状态寄存器；用于存放程序运作中的各种状态信息以及中断 等状态；由应用状态寄存器（APSR）、中断状态寄存器（IPSR）和执行状态寄存器（EPSR）组成；
-   PRIMASK、FAULTMASK和BASEPRI：中断屏蔽寄存器；用于控制异常和中断的屏蔽；
-   CONTROL：控制寄存器；用于定义特权状态和当前使用哪一个堆栈指针。



### GPIO引脚操作方法

![image-20230403161941395](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633754.png)

-   使能对应的时钟；
-   配置引脚功能；
-   决定引脚状态。

### GPIO工作模式

手册P143，需要时再看

STM32F103系列的I/O引脚共有8种工作模式，其中输出模式有四种：推挽输出、开漏输出、复用推挽输出、复用开漏输出；

（复用主要是为了节省引脚的占用，单片机的主要作用就是控制和通信）

输入模式有四种：上拉输入、下拉输入、浮空输入、模拟输入。

**推挽输出**：输出电流增大，提高输出引脚的驱动能力，负载能力，驱动速度。

![image-20230325101257031](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633985.png)



### STM32的外设引脚重映射

比如STM32 USART1控制器使用的收发引脚是PA9和PA10，

为了用户更好的分配引脚，比如PA9用作了其它用途。STM32就加入外设引脚重映射的功能，即，USART1除了能通过PA9和PA10收发以外，还能通过PB6、PB7收发。

一般在引脚分配图里总结了所有引脚的默认功能、复用功能，重映射功能，以及在本开发板硬件中，用作什么功能。



### 引脚复用与重映射总结

简单的看 ，复用功能和重映射差不多，都是让引脚除了IO输入输出，还支持其它功能。

代码上不同的是，重映射需要开启相应的AFIO时钟，
比如开发板上，把PB5重映射为SPI引脚。

![image-20230324142430778](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633960.png)

![image-20230324142445455](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633415.png)

![image-20230324142458898](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633717.png)



### GPIO输出速度

该输出速度不是输出信号的速度，而是I/O口驱动电路的响应速度。

越快则功耗越高，噪声越大，电磁干扰越强。

通常简单外设，比如LED灯、蜂鸣器灯，建议使用2MHz的输出速度，而复用为I^2^C、SPI等通信信号引脚时，建议使用10MHz或50MHz以提高响应速度。



### 配置时钟树

#### 查看时钟频率的两个方法

1.   HAL库提供的函数HAL_RCC_GetSysClockFreq()，
     获取系统时钟频率，再通过串口打印或者debug调试显示结果。
2.   硬件方法：PA8可以复用为MCO引脚，对外提供时钟输出。（不同芯片引脚不一样）用示波器监控该引脚的输出来判断我们的系统时钟是否设置正确。

#### 时钟配置参数FLASH_LATENCY_x的选择

![image-20230324141755926](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633706.png)

分别对应0， 1， 2



### 中断与异常

NVIC（Nested Vectored Interrupt Controller， 嵌套向量中断控制器）

NMI（Non Maskable Interrupt，不可屏蔽中断）

#### 优先级配置

通过应用中断和复位控制寄存器（Application Interrupt and Reset Control Register，AIRCR）的Bits[10:8] （PRIGROUP）将优先级分组。分组决定每个可编程中断的PRI_n的Bits[7:0]的高低位分配，从而影响抢占优 、先和子优先级的级数

![image-20230323094532516](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633626.png)

![image-20230323094555851](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633645.png)

抢占优先级**决定是否可以产生中断嵌套**，子优先级**决定中断响应顺序**，若两种优先级一样则**看中断在中断异常表中的位置，越靠前越先响应**

当这两个中断同时到达，则中断控制器会根据它们的子优先级决定先处理哪个

如果两个中断的优先级都设置为一样了，那么谁先触发的就谁先执行；如果是同时触发的，那么就根据 中断异常表的位置（靠前）来决定谁先执行。



### STM32的中断

![image-20230323095411884](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633133.png)



![image-20230323095451599](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633452.png)

**通常中断优先级分组只会设置一次，它针对的是系统中所有的中断。**后续设置某个中断的中断优先级时，只需要在这个组规定的抢占优先级数和子优先级级数范围内分配优先级级数。**后续代码中，不应该再修改中断优先级分组，否则导致中断顺序不按预期触发。**本手册所有实验，将设置中断优先级分组放在了 “**HAL_Init()**”里



```c
HAL_NVIC_SetPriority(IRQn_Type IRQn, uint32_t PreemptPriority, uint32_t SubPriority)
```

该函数的三个参数分别为中断号，抢占优先级和子优先级



中断是否会优先执行依据：首先是抢占先式优先级等级，其次是子优先级等级，只有**抢占优先级才可能出现中断嵌套**。



#### EXTI

EXTI（External interrupt/event controller，外部中断/事件控制器）

每个中断/事件 都有独立的触发和屏蔽设置，支持中断模式和事件模式。

中断模式是指外部信号产生电平变化时，EXTI将该信号给NVIC处理，从而触发中断，执行中断服务函数，完成对应操作。 

事件模式是指外部信号产生电平变化时，EXTI根据配置，联动ADC或TIM执行相关操作。 

中断和事件的产生源是一样的，中断需要**软件实现相应功能**，而事件是由**硬件触发后执行相应操作**。前者需要CPU参与功能实现，可以实现的功能更多，后者无需CPU参与，具有更高的响应速度



外部信号输入后，首先经过边缘检测电路，可以实现对上升沿或下降沿信号进行检测，从而得到硬件触发，也可由软件中断事件寄存器产生软件触发信号。无论是硬件触发还是软件触发，如果中断屏蔽寄存器允许，则产生中断给NVIC 处理（绿色路线）；如果事件屏蔽寄存器允许，则产生事件，脉冲发生器产生脉冲供其它模块使用（黄色路线）

![image-20230323113728478](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633674.png)



同组的GPIO共享一条中断线，比如EXTI0 组，PA0作为了中断源，**则此时PB0~PG0不能作为中断源**





### SysTick定时器

又名系统滴答定时器。使用内核的SysTick定时器来**实现延时**，可以不占用系统定时器，节约资源。

由 于SysTick是在CPU核内部实现的，跟MCU外设无关，因此它的代码可以在不同厂家之间**移植**。



### 硬件延时与软件延时

**硬件延时**指利用具有计数功能的硬件进行延时。如：定时器（Timer）、 实时时钟（RTC）、 系统滴答定时器（SysTick）等具有计数功能的硬件。

**软件延时**就是写一段软件代码，通过消耗CPU时间进行延时。

比如软件延时函数。

实际应用中，延时分阻塞和非阻塞延时。

**阻塞延时**

**指CPU一直停留阻塞，不去做其它事情，直到延时结束结束。**

像软件延时就是一个典型的阻塞延时，一直消耗CPU，直到延时结束。

**非阻塞延时**

**指在延时期间，没有阻塞CPU，也就是说CPU在延时期间可以执行其它代码。**

比如：利用定时器中断延时，只需要开启定时器，在中断（计数）到来之前，CPU可以执行其它代码。

利用定时器也能实现**阻塞延时**，比如STM32的HAL自带的阻塞延时：void HAL_Delay(uint32_t Delay)

利用RTOS自带的系统延时实现**非阻塞延时**,这个实现原理实际是利用了硬件延时（系统滴答定时器）。

**区别**：硬件延时相对软件延时更普遍。

1.软件相对硬件延时精度更差；

2.软件延时为阻塞延时，硬件延时可阻塞，也可非阻塞延时；

3.硬件延时应用更灵活、更广泛；



### 按键中断例程

在HAL库中，EXTI0~4这五个中断是各自独 立的中断服务函数，EXTI5 - 9共用一个中断服务函数，EXTI10 - 15共用一个中断服务函数。

中断处理函数的实现，可以放在“stm32f1xx_it.c”中，同时注意在“stm32f1xx_it.h”中声明。



### 蜂鸣器

有源蜂鸣器：内部有震荡源，只要通电即可自动发出固定频率的声音。

无源蜂鸣器：内部无震荡源，需要外部脉冲信号驱动发声，声音频率可变。



### 通信

同步通信，通信速率由时钟信号决定，时钟信号越快，传输速度就越快。

对于异步通信，需要收发双方提前统一通信速率，这也就是我们串口调试时，波特率不对显示乱码的原因

#### 常见通信协议

![image-20230323125431321](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633732.png)



#### 常见通信接口标准

![image-20230323125551255](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633684.png)

![image-20230323125607138](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633892.png)



#### STM32的串口

通用同步异步收发器 （Universal synchronous asynchronous receiver transmitter，USART）

通用异步收发器（Universal  asynchronous receiver transmitter，UART）

USART和UART的主要区别在于，USART支持同步通信，该模 式有一根时钟线提供时钟。

![image-20230323130207005](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633023.png)

##### USART内部结构

![image-20230323131204210](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633453.png)

![USART](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633817.png)



##### 串口传输数据的两种方式

1.   使用超时管理模式，串口收发数据: HAL UART Receive () /HAL UART Transmit (）可以理解为按键查询方式。
     使用中断模式，串口收发数据:HAL UART Receive IT( ）/HAL UART Transmit IT()效率更高。
2.   DMA模式，减轻CPU负载，使用DMA自动收发数据HAL UART Receive DMA() /HAL UART Transmit DMA()



##### “UART_HandleTypeDef”结构体（官方也称句柄）变量husart

句柄内容：

![image-20230323143412020](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633759.png)

故传址：

![image-20230323143456999](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633723.png)

##### 重定向打印函数

以上初始化完成后，就可以使用HAL库提供的“HAL_UART_Transmit()”从串口发送数据，使用 “HAL_UART_Receive()”接收数据，但这样使用不方便，需要自己处理数据类型。在学习C语言时，通常 使用printf将数据格式化打印，比较方便。因此，这里需要重定向打印函数，在使用printf时调用 “HAL_UART_Transmit()”打印

**printf和scanf会分别调用“fputc()”和“fgetc()”**，因此这里覆写这两个函数，使用HAL提供的函数实现收发数据



##### STM32使用printf: MicroLIB与printf重定向

以下内容来自[STM32 使用printf：MicroLIB与printf重定向 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/565613666)

**printf函数重定向，加入函数如下所示（HAL库）**

```C
int fputc(int ch, FILE *f)
{
  HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xffff);	//长度可改变
  return ch;
}
int fgetc(FILE * f)
{
  uint8_t ch = 0;
  HAL_UART_Receive(&huart1,&ch, 1, 0xffff);
  return ch;
}
```

**MicroLIB与C标准库的区别：**

MicroLib 专为深度嵌入式应用而设计；

MicroLib 经过优化，可以使用比使用 ARM 标准库更少的代码和数据存储器；

MicroLib 设计为**无需操作系统**即可工作，但这并不妨碍它与任何操作系统或 RTOS（例如 Keil RTX）一起使用；（存疑。文档中写道Microlib不能与操作系统一起使用）

MicroLib 不包含文件 I/O 或宽字符支持；

由于 MicroLib 已经过优化以最小化代码大小，因此某些函数的执行速度将比 ARM 编译工具中可用的标准 C 库例程更慢；

MicroLib 和 ARM 标准库都包含在 Keil MDK-ARM 中；

有关更多详细信息，请参阅与默认 C 库的差异

**printf重定向目的**

C语言程序编写过程中需要打印调试，最为常用的是printf函数，该函数是将信息打印到屏幕上，然而嵌入式系统没有计算机常规定义的屏幕，因此需要将printf函数进行改写，主要是通过串口进行输出。

此外，printf函数位于标准库中，基于嵌入式的printf同样位于MicroLIB中，在嵌入式系统中使用printf函数，需要添加MicroLIB。

**printf重定向操作**

就是改写如下的函数

int fputc(int ch, FILE *f)

int fgetc(FILE * f)



#### I^2^C

I²C由两条线组成，一条双向串行**数据线**SDA，一条串行**时钟线**SCL。每个连接到总线的设备都有一个**独立的地址**，主机可以通过该地址来访问不同设备。因为I²C协议比较简单，常常用GPIO来模拟I²C时序，这种 方法称为**模拟I²C**。如果使用MCU的I²C控制器，设置好I²C控制器, I²C控制器就自动实现协议时序，这种方式 称为**硬件I²C**。因为I²C设备的速率比较低，通常两种方式都可以，**模拟I²C方便移植，硬件I²C工作效率相对较高**。

-   数据有效性：SCL 为高电平时表示有效数据，SDA为高电平表示“1”，低电平表示“0”；SCL为低电平时表示无效数据，此时SDA可以进行电平切换，为下次数据表示做准备。
-   开始信号：当SCL高电平时，SDA由高电平向低电平转换；
-   停止信号：当SCL高电平时，SDA由低电平向高电平转换；
-   应答信号：I²C每次传输的8位数据，每次传输后需要从机反馈一个应答位，以确认从机是否正常接收了数据。当主机发送了8位数据后，会再产生一个时钟，此时主机放开SDA的控制，读取SDA电平，在上拉电阻的影响下， 此时SDA默认为高，必须从机拉低，以确认收到数据。
-   ![image-20230406104607817](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633900.png)

![image-20230406104542177](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633791.png)

![image-20230406095835192](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633633.png)

注意：此外，I^2^C的两个脚SCL和SDA都进行了上拉处理，从而保证I^2^C总线空闲时，两根线都必须为高电平。 如果没有上拉，在主机发送完数据后，放开SDA，此时SDA的电平状态不确定，可能为高，也可能为低，无法确定是从机拉低给出应答信号

![image-20230406101309773](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633641.png)

##### 模拟I^2^C见手册

##### 硬件I^2^C

I^2^C控制器

-   STM32F103系列的I²C控制器，可作为通信主机或从机，因此有四种工作模式可选择：主机发送模式、 主机接收模式、从机发送模式、从机接收模式。
-   传输速度：支持标准模式与快速模式。
-   支持SMBus（系统管理总线）与PMPus(电源管理总线)

![image-20230406130600714](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633710.png)

①引脚：I²C协议只需要两个引脚（SDA和SCL），SMBA引脚仅用于SMBus模式的Alert引脚，通常不用管。 

②数据收发：主要涉及到数据寄存器（Data Register，DR）和数据移位寄存器（Data Shift Register，DSR）。 当发送数据时，将发送的字节写入DR寄存器，硬件会把DR中的字节搬到DSR中，然后在时钟信号的配 合下，把DSR最高位的数据放到数据线SDA上，并对DSR进行移位操作。 当接收数据时，数据控制器(Data Control)根据时钟信号，把SDA线上的高低电平转换为“1”或“0”的 数据，写到DSR的最低位，同时DSR移位操作，当接收完一个字节的8位数据后，把DSR中的数据搬到DR寄 存器中。 

③时钟信号：时钟控制器(Clock Control)用于驱动同步时钟信号线SCL。通过配置时钟控制寄存器（Clock Control Register，CCR），可以调整SCL的频率。

 ④控制逻辑：有两个控制寄存器（Control Register 1，CR1）和（Control Register 2，CR2）用于控制逻辑。通过它们可以触发起始和停止信号，做出ACK响应，配置外设时钟频率，开启DMA和中断的功能。同 时控制逻辑的状态会反馈到（Status Register 1，SR1）和（Status Register 2，SR2）两个状态寄存器上，根据它们可以知道当前总线是否被占用，本机是主设备还是从设备，数据是否发送完毕等

#### SPI

全双工同步串行通信接口。

SPI接口主要应用在EEPROM、FLASH、实时时钟、网络控制器、OLED显示驱动器、AD转换器，数字信号处理器、数字信号解码器等设备之间。

SPI通常由四条线组成，一条主设备输出与从设备输入（Master Output Slave Input，MOSI），一条主设备输入与从设备输出（Master Input Slave Output，MISO），一条时钟信号（Serial Clock，SCLK），一条从设备使能选择（Chip Select，CS）。

![image-20230406172206378](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633650.png)

**数据交换**：在SCK时钟周期的驱动下，MOSI和MISO同时进行，可看作一个虚拟的环形拓扑结构。

![image-20230406172619048](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633843.png)

无论主机还是从机， 发送和接收都是同时进行的，如同一个“环”。 如果主机只对从机进行写操作，主机只需忽略接收的从机数据即可。如果主机要读取从机数据，需要主 机发送一个空数据来引发从机发送数据。

**传输模式**：SPI有四种传输模式，如表 21.1.2 所示，主要差别在于CPOL和CPHA的不同。

CPOL（Clock Polarity，时钟极性）表示SCK在空闲时为高电平还是低电平。当CPOL=0，SCK空闲时为低电平，当CPOL=1，SCK空闲时为高电平。

 CPHA（Clock Phase，时钟相位）表示SCK在第几个时钟边缘采样数据。当CPHA=0，在SCK第一个边沿采样数据，当CPHA=1，在SCK第二个边沿采样数据。

![image-20230406173035818](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633990.png)

**不同SPI工作模式下引脚功能**

![image-20230406174029754](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633497.png)

![image-20230406174312468](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633706.png)

##### 用GPIO模拟SPI通信时序

首先主机和从机都选择同一传输 模式。然后主机片选拉低，选中从机。接着在时钟的驱动下，MOSI发送数据，同时MISO读取接收数据。最 后完成传输，取消片选。

### 寄存器编程

**原则：不要影响其他位！！！**

基础操作：

-   设置指定位：a |= 1 << bit
-   清除指定位：a &= ~(1 << bit)
-   测试指定位：if(a & 1 << bit)

有些芯片支持 set-and-clear protocol，内置了两个寄存器：

`clr-reg`与`set-reg`，该操作更快（传统的需要三步：读出，修改，写入）

#### 使用寄存器操作STM32F103pro的LED

1.   查看硬件原理图，找到对应外设资源引脚

![image-20230403163714672](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633774.png)

2.   以PB0为例，我们需要首先使能GPIOB，再设置PB0的引脚状态。通过查阅芯片手册得知GPIOB挂载在APB2下，且该寄存器的`bit:3`位控制着GPIOB的使能

![image-20230403165320471](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633812.png)

### 疑问

1.   嵌入式软件开发过程中，.elf  .hex .bin文件的转换。







### 

#### EEPROM、RAM、FLASH

**RAM**是存变量以及变量的运算的地方，flash是存程序的地方（flash只读，不能简单的写，得通过一系列擦除之类的步骤）。

只读的常量会保存在flash里（防止浪费内存空间），但优化时有可能放在内存里（增快速度）

**EEPROM**通常用于存放用户配置信息数据，开发板断电后配置信息不丢失，下次启动，开发板会自动读取EEPROM的配置 信息，就不需要重新配置。

FLASH包括MCU内部的Flash和外部扩展的Flash。从功能上，Flash通常存放运行代码，运行过程中不会修改，而EEPROM存放用户数据，可能会反复修改。从结构上，Flash按扇区操作，EEPROM通常按字节操作。

**Flash**有个物理特性：只能写0，不能写1。如果把Flash的每个Bit，都看作一张纸，bit=1表示纸没有内容， bit=0表示纸写入了内容。当纸为白纸时（bit=1），这时往纸上写东西是可以的，写完后纸的状态变为bit=0。 当纸有内容时（bit=0），这时往纸上写东西只能让数据越乱，也就无法正常写数据。此时需要橡皮檫，进行 擦除操作，将有内容的纸（bit=0）变为白纸（bit=1），使得以后可以重新写入数据。

因此，对Flash写数据前，通常需要擦除操作。对于W25Q64，数据擦除可以以Sector为单位也可以以Block 为单位。数据写入只能按照Page来写入，也就一次最多只能写256个Byte。

![image-20230406173444608](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304071633820.png)





学习使用万用表、示波器/逻辑分析仪、电烙铁

