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

driver_sram.c代码逻辑：

```C
//driver_sram.c主要用来实现读写字节，半字，字的函数
//函数声明分别为
void SRAM_WriteBufferBytes(uint8_t *pdata, uint32_t addr, uint16_t size);
void SRAM_ReadBufferBytes(uint8_t * pdata, uint32_t addr, uint16_t size);
void SRAM_WriteBufferHalfWord(uint16_t *pdata, uint32_t addr, uint16_t size);
void SRAM_ReadBufferHalfWord(uint16_t * pdata, uint32_t addr, uint16_t size);
void SRAM_WriteBufferWord(uint32_t *pdata, uint32_t addr, uint32_t size);
void SRAM_ReadBufferWord(uint32_t * pdata, uint32_t addr, uint32_t size);

//可以看出上述三组读写函数的主要区别在于数据存储地址的指针不同

//以读写字节函数为例
void SRAM_WriteBufferBytes(uint8_t *pdata, uint32_t addr, uint16_t size)
{
    uint32_t i = 0;
    __IO uint8_t offset = (uint8_t)addr;

    for(i = 0; i < size; i++)
    {
        *(__IO uint8_t *)(IS62WV51216_BASE_ADDR + offset) = pdata[i];
        offset ++;
    }
}

void SRAM_ReadBufferBytes(uint8_t * pdata, uint32_t addr, uint16_t size)
{
    uint16_t i = 0;
    __IO uint8_t offset = (uint8_t)addr;

    for(i = 0; i < size; i++)
    {
        pdata[i] = *(__IO uint8_t *)(IS62WV51216_BASE_ADDR + offset);
        offset ++;
    }
}
//另外两组读写函数的主要区别在于偏移地址的大小，半字对应的偏移地址为2，字对应的偏移地址为4，不再赘述
```



driver_fsmc.h中的宏定义：

```C
#define BANK1_AREA1_ADDR ((uint32_t)(0x60000000))
#define BANK1_AREA2_ADDR ((uint32_t)(0x64000000))
#define BANK1_AREA3_ADDR ((uint32_t)(0x68000000))
#define BANK1_AREA4_ADDR ((uint32_t)(0x6C000000))

#define FSMC_GPIOD_PIN ( GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_4 | GPIO_PIN_5 | GPIO_PIN_8 | GPIO_PIN_9 \
                        | GPIO_PIN_10 | GPIO_PIN_11 | GPIO_PIN_12 | GPIO_PIN_13 | GPIO_PIN_14 | GPIO_PIN_15)

#define FSMC_GPIOE_PIN ( GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_7 | GPIO_PIN_8 | GPIO_PIN_9 | GPIO_PIN_10 \
                        | GPIO_PIN_11 | GPIO_PIN_12 | GPIO_PIN_13 | GPIO_PIN_14 | GPIO_PIN_15)

#define FSMC_GPIOF_PIN ( GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2 | GPIO_PIN_3 | GPIO_PIN_4 | GPIO_PIN_5 \
                        | GPIO_PIN_12 | GPIO_PIN_13 | GPIO_PIN_14 | GPIO_PIN_15)

#define FSMC_GPIOG_PIN ( GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2 | GPIO_PIN_3 | GPIO_PIN_4 | GPIO_PIN_5 \
                        | GPIO_PIN_10)
```



driver_fsmc.c代码主体：

```C
static FSMC_NORSRAM_TimingTypeDef hfsmc_timing;
static SRAM_HandleTypeDef	hsram;
static uint8_t fsmc_rw_flag = 0;

//FSMC控制器初始化
void FSMC_SRAM_Init(void)
{
    hfsmc_timing.AddressSetupTime = 0x00;	//ADDSET range: 0 ~ 15 mode A need
    //hfsmc_timing.AddressHoldTime = 0x01;	//ADDHLD range: 1 ~ 15 
    hfsmc_timing.DataSetupTime = 0x02; 	//DATAST range:1~255 mode A need
    //hfsmc_timing.BusTurnAroundDuration = 0x00;	//BUSTURN range: 0 ~ 15
    //hfsmc_timing.CLKDivision = 0x02;		//CLKDIV range: 2~16
    //hfsmc_timing.DataLatency = 0x02;		//数据产生时间 ACCMOD range:2~17
    hfsmc_timing.AccessMode = FSMC_ACCESS_MODE_A;	//mode A
    
    hsram.Instance = FSMC_NORSRAM_DEVICE;			//实例类型：NOR/SRAM设备
    hsram.Extended = FSMC_NORSRAM_EXTENDED_DEVICE;
    hsram.Init.NSBank = FSMC_NORSRAM_BANK3;	//使用NE3,对应BANK1(NORSRAM)的区域3
    //地址/数据线不复用
    hsram.Init.DataAddressMux = FSMC_DATA_ADDRESS_MUX_DISABLE;
    hsram.Init.MemoryType = FSMC_MEMORY_TYPE_SRAM;//存储器类型为SRAM
    hsram.Init.MemoryDataWidth = FSMC_NORSRAM_MEM_BUS_WIDTH_16;//16位位宽
    
    //是否使能突发访问，仅对同步突发存储器有效，此处未用到
    //hsram.Init.BurstAccessMode = FSMC_BURST_ACCESS_MODE_DISABLE;
    //等待信号的极性，适用于突发模式访问，此处未用到
    //hsram.Init.WaitSignalPolarity = FSMC_WAIT_SIGNAL_POLARITY_LOW;
    // 存储器是在等待周期之前的一个时钟周期还是等待周期期间使能 NWAIT,适用于突发模式访问,此处未用到
    //hsram.Init.WaitSignalActive = FSMC_WAITING_TIMING_BEFORE_WS;
    
    hsram.Init.WriteOperation = FSMC_WRITE_OPERATION_ENABLE;//存储器写使能
    //hsram.Init.WaitSignal = FSMC_WAIT_SIGNAL_DISABLE;//等待使能位，适用于突发模式访问，此处未用到
    hsram.Init.ExtendedMode = FSMC_EXTENDED_MODE_ENABLE;//启用扩展模式，适用模式A
    
    //是否使能异步传输模式下的等待信号，此处未用到
    //hsram.Init.AsynchronusWait = FSMC_ASYNCUSING_WAIT_DISABLE;
    //禁止突发写，适用于突发模式访问，此处未用到
    //hsram.Init.WriteBurst = FSMC_WRITE_BURST_DISABLE;
    
    if(HAL_SRAM_Init(&hsram, &hfsmc_timing, &hfsmc_timing))
    {
        Error_Handler();
    }
}

//SRAM初始化
void HAL_SRAM_MspInit(SRAM_HandleTypeDef *hsram)
{
    GPIO_InitTypeDef    GPIO_InitStruct = {0};
    
    __HAL_RCC_FSMC_CLK_ENABLE();                     // 使能FSMC时钟
	__HAL_RCC_GPIOD_CLK_ENABLE();                    // 使能GPIO端口D时钟
	__HAL_RCC_GPIOE_CLK_ENABLE();                    // 使能GPIO端口E时钟
	__HAL_RCC_GPIOF_CLK_ENABLE();                    // 使能GPIO端口F时钟
	__HAL_RCC_GPIOG_CLK_ENABLE();                    // 使能GPIO端口G时钟
    
    GPIO_InitStruct.Mode      = GPIO_MODE_AF_PP;     // 配置为复用推挽功能
    GPIO_InitStruct.Pull      = GPIO_PULLUP;         // 上拉
    GPIO_InitStruct.Speed     = GPIO_SPEED_FREQ_HIGH;// 引脚翻转速率快
    
    // PDx
    GPIO_InitStruct.Pin = FSMC_GPIOD_PIN;            // 按上面参数配置D组所有GPIO
    HAL_GPIO_Init(GPIOD, &GPIO_InitStruct);
    
    // PEx
    GPIO_InitStruct.Pin = FSMC_GPIOE_PIN;            // 按上面参数配置E组所有GPIO
    HAL_GPIO_Init(GPIOE, &GPIO_InitStruct);
    
    // PFx
    GPIO_InitStruct.Pin = FSMC_GPIOF_PIN;            // 按上面参数配置F组所有GPIO
    HAL_GPIO_Init(GPIOF, &GPIO_InitStruct);
    
    // PGx
    GPIO_InitStruct.Pin = FSMC_GPIOG_PIN;            // 按上面参数配置G组所有GPIO
    HAL_GPIO_Init(GPIOG, &GPIO_InitStruct);
}
```



main.c关键代码段：

```C
if(FSMC_RW_GetFlag())
{
    FSMC_RW_SetFlag(0);
    
    //准备随机生成的写入数据
    for(i = 0; i < TEST_NUM; i ++)
    {
        WriteDataBytes[i] = rand();
        WriteDataHalfWord[i] = rand();
        WriteDataWord[i] = rand();
    }
    
    //读写N个字节并判断是否一致
    SRAM_WriteBufferBytes(WriteDataBytes, 0, TEST_NUM);
    SRAM_ReadBufferBytes(ReadDataBytes, 0, TEST_NUM);
    printf("**********************************************\n\r");
    for(i=0;i<TEST_NUM;i++)
    {
		if(WriteDataBytes[i] != ReadDataBytes[i])
		printf("Error! \n\r");
        printf("WriteBytes[%03d]=0x%02x, ReadBytes[%03d]=0x%02x\n\r", i, WriteDataBytes[i], i,ReadDataBytes[i]);
    }
    
    //读写N个半字，字代码与上述字节十分相似，在表示上有些许不同，如：
    //printf("WriteHalfWord[%03d]=0x%04x, ReadHalfWord[%03d]=0x%04x\n\r", i, WriteDataHalfWord[i], i, ReadDataHalfWord[i]);
    //printf("WriteWord[%03d]=0x%08x, ReadWord[%03d]=0x%08x\n\r", i, WriteDataWord[i], i, ReadDataWord[i]);
}
```



## 附：存储器的分类

版权声明：本文为CSDN博主「十一和柴柴」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_42973834/article/details/108676693

![image-20230410145854257](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304101459492.png)**RAM**
RAM：随机访问存储器(Random Access Memory)，易失性。是与CPU直接交换数据的内部存储器，它可以随时读写，而且速度很快，通常作为操作系统或其他正在运行中的程序的临时数据存储媒介。它的作用是当开机后系统运行占一部分外，剩余的运行内存越大，手机速度越快，运行的程序越多，剩余越少。

**SRAM**
静态随机存取存储器（Static Random Access Memory，SRAM）是随机存取存储器的一种。所谓的“静态”，是指这种存储器只要保持通电，里面储存的数据就可以恒常保持。SRAM不需要刷新电路即能保存它内部存储的数据，而DRAM（Dynamic Random Access Memory)每隔一段时间，要刷新充电一次，否则内部的数据即会消失，因此SRAM具有较高的性能，但是SRAM也有它的缺点，即它的集成度较低，功耗较DRAM大，相同容量的DRAM内存可以设计为较小的体积，但是SRAM却需要很大的体积。同样面积的硅片可以做出更大容量的DRAM，因此SRAM显得更贵。然而，当电力供应停止时，SRAM储存的数据还是会消失（被称为volatile memory），这与在断电后还能储存资料的ROM或闪存是不同的。

**从晶体管的类型分，SRAM可以分为双极性与CMOS两种。**
从功能上分，SRAM可以分为**异步SRAM和同步SRAM**（SSRAM）。异步SRAM的访问独立于时钟，数据输入和输出都由地址的变化控制。同步SRAM的所有访问都在时钟的上升/下降沿启动。地址、数据输入和其它控制信号均于时钟信号相关。
SRAM的速率高、性能好，它主要有如下应用：
1）CPU与主存之间的高速缓存。
2）CPU内部的L1／L2或外部的L2高速缓存。



**DRAM**

DRAM（Dynamic Random Access Memory），即动态随机存取存储器，最为常见的系统内存。DRAM 只能将数据保持很短的时间。为了保持数据，DRAM使用电容存储，所以必须隔一段时间刷新（refresh）一次，如果存储单元没有被刷新，存储的信息就会丢失。
**SDRAM**
SDRAM 是DRAM 的一种，它是同步动态存储器，利用一个单一的系统时钟同步所有的地址数据和控制信号。动态是指存储阵列需要不断的刷新来保证数据不丢失；随机是指数据不是线性依次存储，而是自由指定地址进行数据读写。

DDR，DDR2以及DDR3就属于SDRAM的一类。当然DRAM对应的还有异步DRAM，只是好像比较少见。
同步SDRAM根据时钟边沿读取数据的情况分为SDR和DDR技术，DDR从发展到现在已经经历了四代，分别是：第一代DDR SDRAM，第二代DDR2 SDRAM，第三代DDR3 SDRAM，第四代，DDR4 SDRAM。
**DDR**
DDR是Double Data Rate的缩写，中文含义是双倍数据率的意思，一听这个名字就知道这个东西很快。全称为DDR SDRAM，即同步动态随机存取存储器，听这个名字也知道，DDR是从SDRAM发展而来。严格的说DDR应该叫SDRAM DDR，人们习惯称为DDR，部分初学者也常看到SDRAM DDR，就认为是SDRAM。SDRAM DDR是SDRAM Rate Data Double的缩写，是双倍速率同步动态随机存储器的意思。DDR内存是在SDRAM内存基础上发展而来的，仍然沿用SDRAM
生产体系。SDRAM在一个时钟周期内只传输一次数据，它是在时钟的上升期进行数据传输；而DDR内存则是一个时钟周期内传输两次次数据，它能够在时钟的上升期和下降期各传输一次数据，因此称为双倍速率同步动态随机存储器。DDR内存可以在与SDRAM相同的总线频率下达到更高的数据传输率。
**DDR2**
DDR2 是 DDR SDRAM 内存的第二代产品。它在 DDR 内存技术的基础上加以改进，从而其传输速度更快（可达 667MHZ ），耗电量更低，散热性能更优良。DDR3，DDR4，DDR5性能依次上升。
与DDR相比，DDR2最主要的改进是在内存模块速度相同的情况下，可以提供相当于DDR内存两倍的带宽。这主要是通过在每个设备上高效率使用两个DRAM核心来实现的。作为对比，在每个设备上DDR内存只能够使用一个DRAM核心。技术上讲，DDR2内存上仍然只有一个DRAM核心，但是它可以并行存取，在每次存取中处理4个数据而不是两个数据。
**DDR3**
DDR3（double-data-rate three synchronous dynamic random access memory）是应用在计算机及电子产品领域的一种高带宽并行数据总线。DDR3在DDR2的基础上继承发展而来，其数据传输速度为DDR2的两倍。同时，DDR3标准可以使单颗内存芯片的容量更为扩大，达到512Mb至8Gb，从而使采用DDR3芯片的内存条容量扩大到最高16GB。此外，DDR3的工作电压降低为1.5V，比采用1.8V的DDR2省电30%左右。说到底，这些指标上的提升在技术上最大的支撑来自于芯片制造工艺的提升，90nm甚至更先进的45nm制造工艺使得同样功能MOS管可以制造的更小，从而带来更快、更密、更省电的技术提升。
在系统设计方面DDR3与DDR2最大的区别在于DDR3将时钟、地址及控制信号线的终端电阻从计算机主板移至内存条上，这样一来在主板上将不需要任何端接电阻。为了尽可能减小信号反射，在内存条上包括时钟线在内的所有控制线均采用Fly-by拓扑结构。同时，也是因为Fly-by的走线结构致使控制信号线到达每颗内存颗粒的长度不同从而导致信号到达时间不一致。这种情况将会影响内存的读写过程，例如在读操作时，由于从内存控制器发出的读命令传送到每颗内存芯片的时间点不同，将导致每颗内存芯片在不同的时间向控制器发送数据。为了消除这种影响，需要在对内存进行读写操作时对时间做补偿，这部分工作将由内存控制器完成。

**FRAM**
FRAM(Ferroelectric RAM): 铁电随机存取存储器
动态随机存取存储器（DRAM）是以一个晶体管加上一个电容来储存一个位（1bit）的资料，由于传统 DRAM 的电容都是使用“氧化矽”做为绝缘体，氧化矽的介电常数不够大（K 值不够大），因此不容易吸引（储存）电子与电洞，造成必须不停地补充电子与电洞，所以称为“动态”，只要电脑的电源关闭，电容所储存的电子与电洞就会流失，DRAM 所储存的资料也就会流失。
要解决这个问题，最简单的就是使用介电常数够大（K 值够大）的材料来取代“氧化矽”为绝缘体，让电子与电洞可以储存在电容里不会流失。目前业界使用“钛锆酸铅”（PZT）或“钽铋酸锶”（SBT）这种介电常数很大（K 值很大）的“铁电材料”（Ferroelectric material）来取代氧化矽，则可以储存电子与电洞不会流失，让原本“易失性”的动态随机存取存储器（DRAM）变成“非易失性”的存储器称为“铁电随机存取存储器”（Ferroelectric RAM，FRAM）。



**ROM**
ROM：只读存储器(Read Only Memory)，非易失性。一般是装入整机前事先写好的，整机工作过程中只能读出，而不像随机存储器那样能快速地、方便地加以改写。ROM所存数据稳定，断电后所存数据也不会改变。计算机中的ROM主要是用来存储一些系统信息，或者启动程序BIOS程序，这些都是非常重要的，只可以读一般不能修改，断电也不会消失。
**PROM**
PROM（Programmable ROM）：可编程ROM，只能被编程一次
**EPROM**
EPROM（Erasable Programmable ROM，EPROM）：可擦写可编程ROM，擦写可达1000次。
**EEPROM**
EEPROM（Electrically Erasable Programmable ROM）电子可擦除EPROM。
**FLASH**
闪存（flash memmory）：基于EEPROM，它已经成为一种重要的存储技术。固态硬盘（SSD)U盘等就是一种基于闪存的存储器。

**NOR FLASH**
NOR Flash的读取和我们常见的SDRAM的读取是一样，**用户可以直接运行装载在NOR FLASH里面的代码**，这样可以减少SRAM的容量从而节约了成本。
Nor Flash，根据外部接口分，可分为**普通接口**和**SPI接口普通接口**的Nor Flash，多数支持CFI接口，所以，一般也叫做CFI接口。CFI接口，相对于串口的SPI来说，也被称为parallel接口，并行接口；另外，CFI接口是JEDEC定义的，所以，有的又成CFI接口为JEDEC接口。所以，可以简单理解为：对于Nor Flash来说，CFI接口＝JEDEC接口＝Parallel接口 = 并行接口
NOR flash占据了容量为1～16MB闪存市场的大部分，而NAND flash只是用在8～128MB的产品当中，这也说明**NOR主要应用在代码存储介质中，NAND适合于数据存储**，NAND在CompactFlash、Secure Digital、PC Cards和MMC存储卡市场上所占份额最大
**NAND FLASH**
NAND Flash没有采取内存的随机读取技术，它的读取是以一次读取一块的形式来进行的，通常是一次读取512个字节，采用这种技术的Flash比较廉价。用户不能直接运行NAND Flash上的代码，因此好多使用NAND Flash的开发板除了使用NAND Flash以外，还用上了一块小的NOR Flash来运行启动代码。

