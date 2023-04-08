# STM32F103读写外部SRAM实验



## 1.关于外部SRAM



### 1.1 FSMC接口介绍

STM32F103ZET6的内部有64KB的SRAM，可以满足大部分的应用场景，但是在一些**采集数据比较多**的项目、**应用算法**或**GUI**等应用场景中，内部SRAM可能就不足了，这时候就需要外部扩展SRAM。



STM32F103系列中，100脚及以上的MCU，都有一个FSMC(Flexible Static Memory Controller)，该外设是STM32设计的一种**存储器控制**技术，仅适用STM32系列的MCU。它可以驱动SRAM、NOR Flash、NAND Flash等存储器，但是不能驱动SDRAM等动态存储器。



FSMC内部框图：

![image-20230408153158976](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081531037.png)

主要由四部分组成：

1.   **时钟源**：FSMC挂载在AHB总线下。
2.   **AHB总线接口**

​		AHB总线接口是CPU、DMA等AHB总线主设备访问FSMC的通道，它负责将AHB总线信号转换成**外设通信协议**。配置寄存器则描述了扩展外设的具体形式、通信协议和接口形式。

3.   **NOR\PSRAM存储器控制器**

​		配置寄存器描述外设的特征和时序后，控制器将自动生成对应的**驱动时序**，以驱动8位，16位，32位的异步SRAM、异步PSRAM或NOR闪存。

4.   **NAND\PC卡存储器控制器**

​		与【NOR\PSRAM存储器控制器】不同的是，它驱动的是8位、16位的NAND存储器或16位的PC卡兼容设备。



FSMC所接外设共用一组地址总线、数据总线、输出使能、写使能和输入等待。前四者功能不再赘述。输入等待用于**同步数据**。并且，FSMC为所接外设提供一个片选信号，如NOR/PSRAM中的NE信号、NAND/PC卡中的NCE信号。通过片选信号，可以实现总线的**分时复用**。



剩下的其他外设信号，与所接外设有关，不同外设需要不同的控制信号，下表为NOR/PSRAM存储器控制器接口信号总结

![image-20230408155309280](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081553321.png)



------

由STM32存储器映射结构可知，0x6000 0000~0x9FFF FFFF共计1GB空间为FSMC的存 储器映射范围。FSMC把这1GB空间，分为了4个固定大小的存储区域（Bank1~4），每个大小为256MB，每 个Bank都由一组独立的配置寄存器控制，相互之间不受干扰，如图所示。

![image-20230408155614305](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081556346.png)

-   Bank1（0x6000 000~0x6FFF FFFF）：用于NOR/PSRAM设备，该区域又被分为4块，每块64MB， 可连接4个NOR Flash设备或PSRAM设备，每个区域通过片选引脚NE[4:1]进行选择。下列Bank也被分为4块，机制一致，不再赘述
-   Bank2（0x7000 000~0x7FFF FFFF）：用于NAND Flash设备；
-   Bank3（0x8000 000~0x8FFF FFFF）：用于NAND Flash设备；
-   Bank4（0x9000 000~0x9FFF FFFF）：用于PC卡设备；



**本次重点学习如何驱动SRAM，因此重点分析NOR\PSRAM控制器，即Bank1。**

CPU需要28根AHB地址总线，才能寻址完Bank1的256MB空间（2^28^=256MB），这里用HADDR[27:0]表 示需要的地址总线。HADDR[25:0]对应连接外部存储器的FSMC_A[25:0]，HADDR[26:27]对应片选信号引脚 NE[4:1]，如表所示。

![image-20230408160317974](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081603009.png)



**CPU寻址问题**

CPU访问一个地址，只能读取1个字节的数据，因此当外部存储器数据宽度不同时，实际向存储器发送的地址也将有所不同，如表所示。

![image-20230408160509654](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081605689.png)

疑问：**为什么存储器数据宽度为16位时，向存储器发送的地址是`HADDR[25:1] >> 1`	呢？**

答：假设CPU想访问存储器0b0000地址数据，得到16位数据，此时只能获取此16位的低8位。接着想通过0b0001地址访问剩下的8位， 却访问的是下一个16位数据。但如果将访问地址右移1位，即无论是0b0000还是0b0001，都是0b0000，再通过NBLx切换高低8位，即可完整的获取存储器0b0000处的16位数据。

![image-20230408173237980](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081732028.png)



------

NOR/PSRAM控制器支持同步访问和异步访问，同步访问不再赘述。异步访问通过提前制定规则保证数据传输的准确，因此FSMC需要设置多个时间参数， 比如数据建立时间（Data-Phase Duration，DATAST）、地址保持时间（Address-hold Phase Duration，ADDHLD） 和地址建立时间（Address Setup Phase Duration ，ADDSET）。



NOR/PSRAM控制器支持的器件类型众多，**不同的器件读写操作时序会有差异**，因此控制器通过切换不同的模式，以支持不同的器件。

![image-20230408173619691](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081736728.png)

非扩展模式跟扩展模式的区别在于FSMC_BCRx寄存器的EXTMOD值设置不同。另外，在扩展模式下，可以混用模式A、B、C、D的读写操作，比如可以使用模式A进行读，模式B进行写。

​		FSMC读外部存储器时，首先Bank1区域内片选信号引脚NEx拉低，选中外部存储器所在Bank；同时读 使能信号引脚NOE拉低，写使能信号引脚NWE保持高电平；接着地址总线A[25:0]在高低字节选择信号 NBL[1:0]的配合下寻址，经历（ADDSET+1）个HCLK周期完成寻址；在随后的（DATASET+1）个HCLK周 期里，外部存储器控制数据总线D[15:0]发送数据；最后，再经历2个HCLK周期后，Bank区域内片选信号引 脚NEx和读使能信号引脚NOE都恢复为高电平。 

​		FSMC写外部存储器时，首先Bank区域内片选信号引脚NEx拉低，选中外部存储器所在Bank；同时读使 能信号引脚NOE拉高，写使能信号引脚NEW先暂时保持高电平；接着地址总线A[25:0]在高低字节选择信号 NBL[1:0]的配合下寻址，经历（ADDSET+1）个HCLK周期完成寻址；然后写使能信号引脚NWE拉低，在随 后的（DATASET+1）个HCLK周期里，FSMC控制数据总线D[15:0]发送数据，使能信号引脚NEW提前1个 HCLK周期拉高；最后，Bank区域内片选信号引脚NEx恢复为高电平，读使能信号引脚NOE都恢复为低电平。

![image-20230408174402068](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081744128.png)

模式A读写访问外部存储器的时序和模式1基本类似，如下图所示。**主要区别在于**FSMC读外部存储时，读使能信号引脚NOE不是一开始就拉高，而是等寻址完后才拉低。

![image-20230408174610347](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081746398.png)





### 1.2 板载SRAM芯片介绍

注：16位数据宽度

![image-20230408175102765](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081751811.png)

1.   **地址、数据部分**

-   A0-A18：寻址范围2^19^/1024 = 512K，每位地址存放16Bit数据，总容量1MB。	
-   I0-I15：传输16Bit数据。

2.   **控制部分**

-    $\overline{\text{CS1}}$ ,CS2：为片选信号，其中有上划线的CS1为低电平有效，CS2为高电平有效；CS2仅存在于48脚 的mini BGA封装，开发板上使用的44脚的TSOP封装没有该脚；
-    $\overline{\text{OE}}$：输出（读）使能信号，低电平有效； 
-   $\overline{\text{WE}}$ ：写使能信号，低电平有效；
-    $\overline{\text{UB}}$ ：高字节控制信号（I/O8 - I/O15），低电平有效；
-    $\overline{\text{LE}}$ ：低字节控制信号（I/O0 - I/O17），低电平有效；





## 2.硬件设计

不同硬件配置原理图不一样，根据具体情况而定。

实验所用SRAM原理图：

![image-20230408180334143](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304081803195.png)





## 3.软件设计

待更新...

## 附：关于SRAM、NOR Flash、 NAND Flash

待更新...