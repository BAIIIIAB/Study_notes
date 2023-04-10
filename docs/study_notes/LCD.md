# LCD

## 1. LCD相关知识介绍

### 1.1 LCD，OLED与Micro LED

**LCD:**

**根据液晶排列方式的不同**，可分为扭转式向列型（Twisted Nematic, TN）、超扭转式向列型（Super  Twisted Nematic, STN）、横向电场效应型（In Panel Switch，IPS）、垂直排列型 (Vertical Alignment，VA)。

**根据驱动形式的不同**，可分为无源矩阵液晶显示屏（Passive Matrix LCD，PMLCD），主要用于TN、 STN；有源矩阵液晶显示屏（Active Matrix LCD，AMLCD），主要用于TFT。

![image-20230410153250606](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101532689.png)



**OLED:**

OLED的工作原理比较简单，使用有机材料实现了类似半导体PN结 的功能效果，通电后有机发光二极管就发光，通的电越多，亮度越高，通过红、绿、蓝不同配比，实现组成各种颜色。

![image-20230410153518245](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101535293.png)



**LED显示屏和Micro LED显示屏：**

OLED使用有机发光二极管组成显示器，生活中常见的半导体发光二极管应用于户外广告牌等场景中（受结构、工艺、散热、成本等限制，无法做的很小，因此画质清晰度比较差，通常适合远距离观看）。

而Micro LED技术，将LED长度缩小到100μm以下，是常见LED的1%，比一粒沙子还要小。因为Micro LED单元过于微小，加大了制造的复杂性和更多的潜在问题，目前各企业正在积极研发中。

Micro LED没有LCD的液晶层，可以像OLED一样独立控制每个像素的开关和亮度，继承了几乎所有LCD和OLED的优点，如果后面Micro LED技术成熟，解决生产上的技术难题，那么Micro LED可能将会是下一代主流显示技术。

![image-20230410153935790](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101539826.png)



### 1.2 显示屏接口介绍

嵌入式图形系统组成示意图：

![image-20230410154531663](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101545704.png)

MCU根据代码内容计算需要显示的图像数据，然后将这些图像数据放入帧缓冲器。帧缓冲器本质是一块内存，因此也被称为GRAM（Graphic RAM）。帧缓冲器再将数据传给显示控制器，显示控制器将图像数据解析，控制显示屏对应显示。



帧缓冲器和显示控制器，可以集成在MCU内部，也可以和显示屏做在一起。对于大部分中、低端MCU， 不含显示控制器，内部SRAM也比较小，因此采用如下图所示的显示方案，将帧缓冲器、显示控制器和显示屏制作在一起，这样的屏幕习惯上称为“MCU屏”。

![image-20230410154739678](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101547720.png)



对于部分高端MCU或MPU，本身含有显示控制器，使用内部SRAM或外部SRAM，如图 38.1.10 所示， 通过并行的RGB信号和控制信号直接控制显示屏，这样的屏幕习惯上称为“RGB屏”。

![image-20230410154805982](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101548038.png)



MIPI（Mobile Industry Processor Interface，移动行业处理器接口）是ARM、ST、TI等公司成立的一个联盟，致力于定义和推广移动设备接口的规范标准化，从而减小移动设备的设计复杂度。



MIPI各接口总结：



![Snipaste_2023-04-10_16-05-07](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101605672.png)

对于STM32系列的MCU，不同型号对MIPI联盟显示接口的支持有所不同，总结如下：

-   所有STM32 MCU均支持MIPI-DBI C类接口（基于SPI协议）；
-   带FSMC的所有STM32 MCU均支持MIPI-DBI A类和B类接口；
-   带LTDC的STM32 MCU支持MIPI-DPI接口；
-   带DSI Host的STM32 MCU支持MIPI-DSI接口；

![image-20230410160707029](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101607080.png)



### 1.3 实验用显示屏控制器

STM32F103ZET6只有FSMC接口，没有LTDC或DSI Host，因此只能接MCU屏。

ILI9488是一款驱动IC，它可以驱动320*480分辨率，16.7M色彩深度的TFT  LCD，同时该芯片内部自带GRAM。

<img src="https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101626693.png" alt="image-20230410162620638"  />

①为控制引脚和信号引脚，支持DBI B类接口（Intel 8080接口）、DPI接口、MIPI DSI接口输入。经过索引寄存器（IR）、控制寄存器（CR）、 地址寄存器（AC）、读数据寄存器（RDR）、写数据寄存器（WDR）到GRAM。最后，GRAM把显示内容 传输到LCD屏幕的显示。

控制引脚IM[2:0]用于设置控制器的接口模式，如表 38.1.4 所示。显示屏支持的接口为16 Bit的MIPI-DBI  B类，因此IM[2:0]值为010，该值由屏幕供应商生产时硬件设置，此时用到的数据引脚为DB[15:0]。

![image-20230410163952200](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101639238.png)



Intel 8080接口16位连接示意图：

![image-20230410164114231](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101641262.png)

MCU和控制器之间，有4根控制信号，16个数据信号。CSX为片选信号，低电平有效（X表示低电平有效）；D/CX为数据/命令切换信号，低电平时DB传输的为命令，高电平时DB传输的为数据；WRX为写信号，低电平有效；RDX为读信号，低电平有效；DB[15:0] 为数据传输信号。



Intel 8080接口16位数据传输示意图：

![image-20230410164237690](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101642744.png)

首先片选信号CS拉低，复位信号RES保持为高。数据/命令切换信号D/C拉低，写信号引脚拉低，此时DB[15:0]发送的就是指令（黄色部分）；数据/命令切换信号D/C拉高，写信号引脚拉低，此时DB[15:0]发送的就是一个像素点的颜色数据（红、绿、蓝部分）， 其中低5位为蓝色数据、中6位为绿色数据、高5位为红色数据。



FSMC信号与8080信号对比

![image-20230410164659265](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101646297.png)

片选、读/写使能、数据总线都能对应，FSMC的地址总线和8080信号的数据/命令切换信号对应不上， 如何让两者对应上呢？假设NEx为NE4，即片选的Bank1的第四区，此时FSMC的寻址范围为0x6C000000~0x6FFF FFFF。**假设A23与D/C连接**，此时CPU访问(0x6C000000 + (1<<24) - 2))地址时，**A23为低电平， 对应D/C为命令信号**；当CPU访问(0x6C000000 + (1<<24))地址时，**A23为高电平，对应D/C为数据信号**。通过这个方式，即可实现地址总线与D/C的对应。



## 2.硬件设计

1.根据LCD屏幕设计的底板；2.开发板的LCD接口

![image-20230410165559768](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101655828.png)

①是连接LCD屏幕的FPC排座；；②是触摸芯片电路，用于实现触摸功能，放在后面一章单独讲解；③是背光驱动电路，通过控制三极管的通断，实现背光的开关，需要注意的是，控制引脚**支持PWM功能**，可以 实现亮度的调节；④是连接开发板的排针接口，该接口与开发板的LCD接口一致。

开发板LCD原理图详见开发板原理图介绍。

STM32使用FSMC接口实现LCD的Intel 8080总线接口，以及使用GPIO控制LCD其它功能，两者引脚的对应关系如下表所示：

![image-20230410170108098](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101701145.png)



## 3.软件设计

软件设计思路：

1.   初始化定时器、PWM输出控制背光； 
2.   初始化FSMC接口； 
3.   初始化LCD控制器，LCD设置；
4.   实现LCD像素点显示、字符显示、数字显示、画线等功能； 
5.   主函数编写控制逻辑：清屏，使用LCD提供的函数，在指定位置显示字符、字符串、数字、图形等。

步骤1关键：

TIM3的计数器CNT从0开始计数到CCR，输出低电平，LCD背光亮，然后计数器CNT从CCR到ARR，输出高电平，LCD灭。即，CCR值越小，占空比越大，背光越暗，CCR值越大，占空比小，背光越亮，CCR值与亮度成正比。CCR的计数范围为0~2000，因此在TIM3中断回调函数里，修改CCR值，即可改变PWM占空比，实现背光不同亮度。

