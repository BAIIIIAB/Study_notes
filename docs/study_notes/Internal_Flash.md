# Flash

## 1.知识铺垫

### 1.1 关于Flash

几乎所有的MCU内部都会有一块Flash，用于保存要运行的代码。在电脑上交叉编译得到二进制文件后， 使用烧写器将该文件烧写到内部Flash上。MCU上电后，自动读取Flash上的内容并加载到内存，CPU再从内存执行代码内容。由于Flash掉电数据不丢失的特性，每次上电后，MCU都能执行同一个程序，直至Flash上的内容被重新烧写。

习惯上，使用内部Flash存储运行代码，外部Flash存储需要掉电保存的数据。但这也不是绝对的，如果没有外部Flash，需要掉电保存的数据也可以放在内部Flash未使用的区域。

STM32内部也有一块Flash，不同的型号其Flash的容量大小也不一样。

![image-20230411143714587](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304111437674.png)

不同密度等级的Flash，其组织结构也不一样，以高密度（HD）为例，其组织结构如下图所示。主要由三部分组成：主存储器、信息块、闪存存储器接口寄存器。

![image-20230411143957958](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304111439014.png)

1.   **主存储器**：用于存储数据的区域，地址范围为0x08000000~0x0807FFFF，当启动引脚BOOT0为低电平时，系统将从0x08000000地址开始读取代码，也就是从主存储器（Main Flash memory）启动。
2.    **信息块** 由两部分组成：**系统存储**和**选项字节**。

-   系统存储：区域大小为2KB，存放有ST出厂固化的程序代码，该代码可实现从UART接收数据并烧写到主存储器区。当启动引脚BOOT0为高电平，BOOT1为低电平时，系统将从0x1FFFF000地址开始读取代码，该代码将从UART收到的数据写到主存储器。也就是通过UART实现程序的下载。		
-   选项字节：区域大小为16 Bytes，用于设置内存读写保护、硬件看门狗软硬模式等功能，可通过ST-Link等调试工具查看该部分寄存器状态。详细介绍参考配套资料“100ASK_STM32F103开发板资料 \2_官方资料\11_**STM32F10xx闪存控制器编程手册**.pdf”。

3.   **闪存存储器接口寄存器**：用于控制闪存的读写、管理等。



### 1.2 Flash读写操作介绍

1.   **读Flash**

     内核使用ICode指令总线访问Flash的指令，使用DCode数据总线访问Flash的数据。Flash存储器有两个64位缓存器组成的预取缓冲器，使得CPU可以工作在更高频率，同时需要根据不同的系统时钟（SYSCLK）频率设置对应的等待周期（这部分之前提到过），通常系统时钟为72MHz，则需要设置2个等待周期（LATENCY），否则读写Flash可能出错，导致死机等情况。

     ![image-20230411145005747](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304111450782.png)

2.   **写Flash**

​		步骤：解锁——>擦除——>写数据——>上锁

-   解锁：MCU复位后，闪存编程和擦除控制器（FPEC）处于锁定保护状态，此时闪存控制寄存器（FLASH_CR）不可操作。需要将两个特定的解锁序列号（Key1=0x45670123 ，Key2=0xCDE89AB）写入闪存键值寄存器 （FLASH_KEYR）以解锁FPEC模块。
-   擦除：MCU复位后，闪存编程和擦除控制器（FPEC）处于锁定保护状态，此时闪存控制寄存器（FLASH_CR） 不可操作。需要将两个特定的解锁序列号（Key1=0x45670123 Key2=0xCDE89AB）写入闪存键值寄存器 （FLASH_KEYR）以解锁FPEC模块。![image-20230411145453888](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304111454933.png)
-   写数据：擦除完后，便可以向FLASH写数据。![image-20230411145617326](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304111456362.png)

需要注意的是：在**擦除与写**的操作中，经检查当前若处于解锁状态时，都需要检查FLASH_SR[BSY]得知是否处于忙状态，如果忙则等待其变为空闲；然后再进行后续操作，而在擦除和写入过程中，都需要检查FLASH_SR[BSY]**直至擦除或写完成**；

-   上锁：写Flash操作结束后，需要设置FLASH_CR[LOCK]为1，使FPEC模块重新上锁，以防止数据被不小心修改。



## 2.软件设计

时钟配置过程中设置等待周期：

```C
if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK)
{
    while(1);
}
```

`driver_flash.c`主体

```C
static FLASH_EraseInitTypeDef EraseInitStruct;

void ReadFlash(uint32_t startAddr, uint32_t *pdata, uint32_t length)
{
#if 1
    //方法一：使用指针获取数据
    uint16_t i = 0;
    for(i = 0; i < length; i ++)
    {
        pdata[i] = *(uint32_t *)startAddr;
        startAddr += 4;	//地址＋4，即下一个32位数据位置
    }
#else
    //使用C库的内存拷贝函数获取数据
    memcpy((uint32_t *)pdata, (uint32_t *)startAddr, sizeof(uint32_t)* length);
#endif
}

void EraseFlash(uint32_t addr, uint32_t length)
{
    uint32_t nPageError = 0;
    EraseInitStruct.TypeErase = FLASH_TYPEERASE_PAGES;	//页擦除
    EraseInitStruct.PageAddress = addr;					//擦除的起始地址
    
#if 1
    //方式1：只擦除用到的页
    EraseInitStruct.NbPages = (addr + length * 4) / FLASH_PAGE_SIZE - addr / FLASH_PAGE_SIZE + (((addr + length * 4) % FLASH_PAGE_SIZE) ? 1 : 0);
    
#else
    //擦除整个测试页
    EraseInitStruct.NbPages = (TEST_PAGE_END_ADDR = addr) / FLASH_PAGE_SIZE;
#endif
    if(HAL_FLASHEx_Erase(&EraseInitStruct, &nPageError) != HAL_OK)
    {
        Error_Handler();
    }
}

void WriteFlash(uint32_t startAddr, uint32_t *pdata, uint32_t length)
{
    uint32_t i = 0;
    uint32_t address = startAddr;
    
    HAL_FLASH_Unlock();
    
    EraseFlash(startAddr, length);	//擦除
    
    for(i = 0; i < length; i ++)
    {
        //写入的数据类型支持半字，字，双字，但实际上都是以半字写入
        if(HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, address, pdata[i] == HAL_OK))
        {
            address += 4;
        }
    }
    HAL_FLASH_Lock();
}
```

在写Flash之前，需要进行擦除操作。擦除有两种方式，页擦除（Page Erase）和批量擦除（Mass Erase）， **这里不能使用批量擦除，会导致下载测试程序也被擦除**，无法执行代码，因此只能使用页擦除。页擦除的处理方式也有两个思路，**一个是根据要写入的大小，计算要擦除页数**，对应擦除；**另一个则是擦除整个测试区域的Flash**。

```C
#define TEST_CODE_SIZE(1024 * 8)//本程序代码约为7.89KB，通过生成的bin文件得知
#define TEST_PAGE_ADDR (0x8000000 + TEST_CODE_SIZE)//写起始地址需要在FLASH上代码后
#define TEST_PAGE_SIZE 	(1024 * 10)	//10KB，理论上余下的512-8=504KB空间都可以使用
#define TEST_PAGE_END_ADDR (TEST_PAGE_ADDR + 4 * TEST_PAGE_SIZE)//测试的Flash空间结束地址
```

程序代码大小可以通过bin文件大小得知，也可以通过Keil编译完成的提示，计算得到占用Flash大小，如图所示，**Code**表示代码占用空间，存放在Flash；**RO-data**表示只读常量占用空间，存放在Flash；**RW-data**表示初始化后的变量占用空间，存放在Flash；**ZI-data**表示未初始化的变量占用空间，它存放在**BSS段**，不占用Flash空间。

![image-20230411152415271](https://xiexun.oss-cn-hangzhou.aliyuncs.com/img2023/202304111524312.png)





main函数控制逻辑

```C
while(1)
{
    if(rw_flag)
    {
        rw_flag = 0;
        for(i = 0; i < TEST_NUM; i ++)
        {
            write_data[i] = rand();
        }
        
        //读写FLASH
        if(TEST_NUM > TEST_PAGE_SIZE)
        {
            printf("ERROR: 超出测试FLASH空间\r\n");
            break;
        }
        WriteFlash(TEST_PAGE_ADDR, write_data, TEST_NUM);
        ReadFlash(TEST_PAGE_ADDR, read_data, TEST_NUM);
        
        //判断读写的数据是否一致
        for(i = 0; i < TEST_NUM; i ++)
        {
            if(read_data[i] != write_data[i])
            {
                printf("Error! \r\n");
                printf("ReadData[%03d] = 0x%08x WriteData[%03d] = 0x%08x \r\n", i, read_data[i], write_data[i]);
                break;
            }
            else
            {
                printf("ReadData[%03d] = 0x%08x WriteData[%03d] = 0x%08x \r\n", i, read_data[i], write_data[i])
            }
        }
        if(i == TEST_NUM)	//每次校验都通过
        {
            printf("test pass!\r\n");
        }
    }
}
```

