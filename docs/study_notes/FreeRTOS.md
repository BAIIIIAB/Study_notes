# 学习笔记

## 0.数据类型和编程规范

### 0.1 数据类型

每个移植的版本都含有自己的 portmacro.h 头文件，里面定义了2个数据类型： 

-   TickType_t：

    -   FreeRTOS配置了一个周期性的时钟中断：Tick Interrupt 

    -   每发生一次中断，中断次数累加，这被称为tick count，tick count这个变量的类型就是TickType_t 

    -   TickType_t可以是16位的，也可以是32位的，FreeRTOSConfig.h中定义configUSE_16_BIT_TICKS时，TickType_t就是uint16_t，否则TickType_t就是uint32_t 

    -   对于32位架构，建议把TickType_t配置为uint32_t 

- BaseType_t： 这是该架构最高效的数据类型 

    - 32位架构中，它就是uint32_t，16位架构中，它就是uint16_t，8位架构中，它就是uint8_t

    - BaseType_t通常用作简单的返回值的类型，还有逻辑值，比如 pdTRUE/pdFALSE

### 0.2 变量名

| 变量名前缀 | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| c          | char                                                         |
| s          | int16_t，short                                               |
| l          | int32_t，long                                                |
| x          | BaseType_t， 其他非标准的类型：结构体、task handle、queue handle等 |
| u          | unsigned                                                     |
| p          | 指针                                                         |
| uc         | uint8_t，unsigned char                                       |
| pc         | char指针                                                     |

### 0.3 函数名

| 函数名前缀        | 含义                                               |
| ----------------- | -------------------------------------------------- |
| vTaskPrioritySet  | 返回值类型：void                    在task.c中定义 |
| xQueueReceive     | 返回值类型：BaseType_t       在queue.c中定义       |
| pvTimerGetTimerID | 返回值类型：pointer to void 在tmer.c中定义         |

### 0.4 宏名

| 宏的前缀                          | 含义：在哪个文件里定义  |
| --------------------------------- | ----------------------- |
| port (比如portMAX_DELAY)          | portable.h或portmacro.h |
| task (比如taskENTER_CRITICAL())   | task.h                  |
| pd (比如pdTRUE)                   | projdefs.h              |
| config (比如configUSE_PREEMPTION) | FreeRTOSConfig.h        |
| err (比如errQUEUE_FULL)           | projdefs.h              |

**通用宏定义**

| 宏      | 值   |
| ------- | ---- |
| pdTRUE  | 1    |
| pdFALSE | 0    |
| pdPASS  | 1    |
| pdFAIL  | 0    |



## 1.任务管理

### 1.1 任务的创建与删除

```C
void ATaskFunction(void *pvParameters);
```

-   任务函数无返回类型

-   多个任务可以运行同一个函数

-   函数内部尽量要使用局部变量

    -   每个任务都有自己的栈

    -   每个任务运行该函数时

        -   任务A的局部变量放在任务A的栈里、任务B的局部变量放在任务B的栈里
        -   不同任务的局部变量有自己的副本

    -   若函数使用全局变量、静态变量
        -   只有一个副本：多个函数使用的是同一个副本
        -   要防止冲突

#### 动态创建任务

```C
BaseType_t xTaskCreate(TaskFunction_t pxTaskCode, const char * const pcName, const configSTACK_DEPTH_TYPE ucStackDepth, void * const pvParameters, UBaseType_t uxPriority, TaskHandle_t * const pxCreatedTask);
```

**参数说明**

| 参数          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| pvTaskCode    | 函数指针，可以简单地认为任务就是一个C函数。<br />特殊点：永远不退出，或者退出时要调用"vTaskDelete(NULL)" |
| pcName        | 任务的名字，FreeRTOS内部不使用它，仅仅起调试作用。<br />长度为：configMAX_TASK_NAME_LEN |
| usStackDepth  | 每个任务都有自己的栈，这里指定栈大小。<br />单位是word，1 word = 4 Bytes。<br />最大值为uint16_t的最大值。<br />**确定栈的大小并不容易**，很多时候是估计。 精确的办法是看反汇编码。 |
| pvParameters  | 调用pvTaskCode函数指针时用到：pvTaskCode(pvParameters)       |
| uxPriority    | 优先级范围：0~(configMAX_PRIORITIES – 1) <br />数值越小优先级越低， 如果传入过大的值，xTaskCreate会把它调整为(configMAX_PRIORITIES – 1) |
| pxCreatedTask | 用来保存xTaskCreate的输出结果：task handle。<br />以后如果想操作这个任务，比如修改它的优先级，就需要这个handle。     如果不想使用该handle，可以传入NULL。 |
| 返回值        | 成功：pdPASS；<br />失败：errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY(失败原因只有内存 不足) <br />**注意**：文档里都说失败时返回值是pdFAIL，这不对。 pdFAIL是0，errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY是-1。 |

#### 静态创建任务

相较于动态创建任务函数`xTaskCreate()`在任务运行时才分配任务所需的栈以及TCB等所需内存而言，静态创建任务函数在编译时就将这些内存分配好，其使用方法为，先在`FreeRTOSconfig.h`中将宏`configSUPPORT_STATIC_ALLOCATION`定义为1，此时编译会报错：

-   Error:  Undefined symbol vApplicationGetIdleTaskMemory (referred from tasks.o).
-   Error: Undefined symbol vApplicationGetTimerTaskMemory (referred from timers.o).

需自行实现这两个函数：

```C
#include "FreeRTOS.h"
#include "task.h"

/* GetIdleTaskMemory prototype (linked to static allocation support) */
void vApplicationGetIdleTaskMemory( StaticTask_t **ppxIdleTaskTCBBuffer, StackType_t **ppxIdleTaskStackBuffer, uint32_t *pulIdleTaskStackSize );

/* USER CODE BEGIN GET_IDLE_TASK_MEMORY */
static StaticTask_t xIdleTaskTCBBuffer;
static StackType_t xIdleStack[configMINIMAL_STACK_SIZE];

void vApplicationGetIdleTaskMemory( StaticTask_t **ppxIdleTaskTCBBuffer, StackType_t **ppxIdleTaskStackBuffer, uint32_t *pulIdleTaskStackSize )
{
  *ppxIdleTaskTCBBuffer = &xIdleTaskTCBBuffer;
  *ppxIdleTaskStackBuffer = &xIdleStack[0];
  *pulIdleTaskStackSize = configMINIMAL_STACK_SIZE;
  /* place for user code */
}
/* USER CODE END GET_IDLE_TASK_MEMORY */

/* USER CODE BEGIN GET_TIMER_TASK_MEMORY */
static StaticTask_t xTimerTaskTCBBuffer;
static StackType_t xTimerStack[configTIMER_TASK_STACK_DEPTH];

void vApplicationGetTimerTaskMemory( StaticTask_t **ppxTimerTaskTCBBuffer, StackType_t **ppxTimerTaskStackBuffer, uint32_t *pulTimerTaskStackSize )
{
  *ppxTimerTaskTCBBuffer = &xTimerTaskTCBBuffer;
  *ppxTimerTaskStackBuffer = &xTimerStack[0];
  *pulTimerTaskStackSize = configTIMER_TASK_STACK_DEPTH;
  /* place for user code */
}
/* USER CODE END GET_TIMER_TASK_MEMORY */

```



**函数原型**

```C
TaskHandle_t xTaskCreateStatic(TsakFunction_t pvTaskCode, const char * const pcName, uint32_t ulStackDepth, void *pvParameters, UBaseType_t uxPriority, StackType_t * const puxStackBuffer, StaticTask_t * const pxTaskBuffer);
```

较动态创建任务多了两个参数：

-   `StackType_t * const puxStackBuffer`栈位置
-   `StaticTask_t * const pxTaskBuffer`TCB位置





#### 删除任务

```C
void vTaskDelete(TaskHandle_t xTaskToDelete);
```

参数说明：

| 参数       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| pvTaskCode | 任务句柄，使用xTaskCreate创建任务时可以得到一个句柄。也可以传入NULL，表示删除自己。 |

如果使用`vTaskDelete()`来删除任务，就要确保空闲任务有机会执行，否则无法释放被删除任务的内存。



### 1.2 任务优先级和Tick

FreeRTOS的调度器可以使用2种方法来快速找出优先级最高的、可以运行的任务。

使用不同的方法时， `configMAX_PRIORITIES `的取值有所不同。

-   通用方法

​		使用C函数实现，对所有的架构都是同样的代码。对`configMAX_PRIORITIES`的取值没有限制。但是`configMAX_PRIORITIES`的取值还是尽量小，因为取值越大越浪费内存，也浪费时间。 `configUSE_PORT_OPTIMISED_TASK_SELECTION`被定义为0、或者未定义时，使用此方法。

- 架构相关的优化的方法

​        架构相关的汇编指令，可以从一个32位的数里快速地找出为1的最高位。使用这些指令，可以快速找出优先级最高的、可以运行的任务。 使用这种方法时，`configMAX_PRIORITIES`的取值不能超过32。` configUSE_PORT_OPTIMISED_TASK_SELECTION`被定义为1时，使用此方法。

#### 任务调度

-   FreeRTOS会确保最高优先级的、可运行的任务，马上就能执行
-   对于相同优先级的、可运行的任务轮流执行

每个任务可执行的时间由时间片大小决定，时间片长度由宏`configTICK_RATE_HZ`决定。

#### Tick

```C
vTaskDelay(2);	//等待两个Tick,假设configTICK_RATE_HZ = 100, Tick周期为10ms，则等待20ms
//宏pdMS_TO_TICKS把ms转换为tick
vTaskDelay(pdMS_TO_TICKS(100));		//等待100ms
```

注：基于Tick实现的延时并不精确，比如`vTaskDelay(2)`的本意是延迟2个Tick周期，有可能经过1个Tick多点就返回了。

使用`vTaskDelay`函数时，建议以ms为单位，使用`pdMS_TO_TICKS`把时间转换为Tick。 

这样代码与`configTICK_RATE_HZ`无关，即使配置项`configTICK_RATE_HZ`改变，也无需修改代码。



#### 获取优先级

```C
UBaseType_t uxTaskPriorityGet(const TaskHandle_t xTask);
```

#### 设置优先级

```C
void vTaskPrioritySet(TaskHandle_t xTask, UBaseType_t uxNewPriority);
```

参数xTask来指定任务，设为NULL表示获取自身优先级。

参数uxNewPriority表示新的优先级，取值范围是0~（configMAX_PRIORITIES - 1)



### 1.3 任务状态

#### 阻塞状态

实际产品中不会让一个任务一直运行，而是使用"事件驱动"的方法让它运行：

-   任务要等待某个事件，事件发生后它才能运行
-   在等待事件过程中，它不消耗CPU资源
-   在等待事件的过程中，这个任务就处于阻塞状态(Blocked)

在阻塞状态的任务，它可以等待两种类型的事件：

-   时间相关的事件
    -    可以等待一段时间：我等2分钟 
    -   也可以一直等待，直到某个绝对时间：我等到下午3点 
-   同步事件：这事件由别的任务，或者是中断程序产生
    -   例子1：任务A等待任务B给它发送数据
    -   例子2：任务A等待用户按下按键 
    -   同步事件的来源有很多(这些概念在后面会细讲)：
        -   队列(queue) 
        -   二进制信号量(binary semaphores) 
        -   计数信号量(counting semaphores) 
        -   互斥量(mutexes) 
        -   递归互斥量、递归锁(recursive mutexes) 
        -   事件组(event groups) 
        -   任务通知(task notifications) 

在等待一个同步事件时，可以加上超时时间。比如等待队里数据，超时时间设为10ms：

-   10ms之内有数据到来：成功返回
-   10ms到了，还是没有数据：超时返回



#### 暂停状态

```C
void vTaskSuspend(TaskHandle_t xTaskToSuspend);
```

参数xTaskToSuspend表示要暂停的任务，NULL为自身。

退出暂停状态，只能由**别人**操作：

-   别的任务调用：`vTaskResume`
-   中断程序调用：`vTaskResumeFromISR`



#### 就绪状态



#### 状态转换图

<img src="img\任务状态转换图.png" alt="任务状态转换图" style="zoom:50%;" />

### 1.4 延时函数

应用场景：固定时间期限处理某件事，中断处理程序耗时很短（微秒级以下可以），但如果处理程序耗时较长（毫秒级），则在中断里处理不现实。

![相对延时与绝对延时对比](img\相对延时与绝对延时对比.png)

-   相对延时：`void vTaskDelay(const TickType_t xTicksToDelay)` 指每次延时都是从执行函数vTaskDelay开始，直到延时指定的时间。

如果执行TEST任务的过程中发生中断，或者具有更高优先级的任务抢占了，那么TEST任务执行的周期就会变长，所以使用相对延时函数vTaskDelay()，不能周期性的执行TEST任务。

<img src="img\相对延时.png" alt="相对延时" style="zoom:50%;" />

-   绝对延时：`vTaskDelayUntil(TickType_t * const pxPreviousWakeTime, const TickType_t xTimeIncrement)`指每隔指定的时间（参数：滴答值），执行一次调用vTaskDelayUntil()函数。

<img src="img\绝对延时.png" alt="绝对延时" style="zoom:50%;" />

绝对延时参数说明：

-   `pxPreviousWakeTime`：上次被唤醒的时间
-   `xTimeIncrement`：阻塞到（`pxPreviousWakeTime + xTimeIncrement`）单位为TickCount



### 1.5 空闲任务与钩子函数

空闲任务由`vTaskStartScheduler()`创建，它的任务优先级为0——不能阻碍用户任务运行

空闲任务要么处于就绪态，要么处于运行态，永远不会阻塞。

空闲任务的钩子函数：每当空闲任务的循环未执行，就会调用一次钩子函数。钩子函数的作用有：

-   执行一些低优先级的、后台的、需要连续执行的函数
-   测量系统的空闲时间：空闲任务能被执行意味着所有高优先级任务都停止了，所以测量空闲任务占据的时间，就可以算出处理器占用率。
-   让系统进入省电模式：同上，无高优先级任务执行即可进入省电模式。

空闲任务钩子函数的限制：

-   不能导致空闲任务进入阻塞状态、暂停状态
-   如果使用`vTaskDelete()`来删除任务，那么钩子函数要非常高效地执行，如果空闲任务一直卡在钩子函数里的话，就无法释放内存。

**使用空闲任务钩子函数步骤**（逻辑位于tasks.c中）：

1.  将宏`configUSE_IDLE_HOOK`定义为1
2.  实现函数`vApplicationIdleHook()`



### 1.6 任务调度配置

由宏`configUSE_PREEMPTION、configUSE_TIME_SLICING、configIDLE_SHOULD_YIELD、configUSE_TICKLESS_IDLE`定义，`configUSE_TICKLESS_IDLE`是高级选项，用于关闭Tick中断来实现省电。

-   `configUSE_PREEMPTION`是否可抢占
-   可抢占：高优先级就绪任务马上执行
    
-   不可抢占：合作调度模式（协商）
-   `configUSE_TIME_SLICING`是否时间片轮转，注：如果不轮转，当前任务会一直执行，直到主动放弃或被高优先级任务抢占
-   `configIDLE_SHOULD_YIELD`空闲任务是否让步于用户任务

| 配置项                  | A    | B      | C      | D      | E        |
| ----------------------- | ---- | ------ | ------ | ------ | -------- |
| configUSE_PREEMPTION    | 1    | 1      | 1      | 1      | 0        |
| configUSE_TIME_SLICING  | 1    | 1      | 0      | 0      | x        |
| configIDLE_SHOULD_YIELD | 1    | 0      | 1      | 0      | x        |
| 说明                    | 常用 | 很少用 | 很少用 | 很少用 | 几乎不用 |





## 2.同步与互斥

能实现同步、互斥的内核方法有：任务通知（task notification）、队列（queue）、事件组（event group）、信号量（semaphore）、互斥量（mutex）它们都有类似的操作方法：获取/释放、阻塞/唤醒、超时。

| 内核对象 | 生产者    | 消费者 | 数据/状态                                                    | 说明                                                         |
| -------- | --------- | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 队列     | ALL       | ALL    | 数据：若干个数据<br />谁都可以往队列里扔数据， 谁都可以从队列里读数据 | 用来传递数据， 发送者、接收者无限制， 一个数据只能唤醒一个接收者 |
| 事件组   | ALL       | ALL    | 多个位：或、与<br />谁都可以设置(生产)多个位， 谁都可以等待某个位、若干个位 | 用来传递事件， 可以是N个事件<br />发送者、接受者无限制， 可以唤醒多个接收者：像广播 |
| 信号量   | ALL       | ALL    | 数量：0~n <br />谁都可以增加一个数量， 谁都可消耗一个数量    | 用来维持资源的个数， 生产者、消费者无限制， 1个资源只能唤醒1个接收者 |
| 任务通知 | ALL       | 只有我 | 数据、状态都可以传输<br />使用任务通知时， 必须指定接受者    | N对1的关系： 发送者无限制， 接收者只能是这个任务             |
| 互斥量   | 只能A开锁 | A上锁  | 位：0、1 <br />我上锁：1变为0， <br />只能由我开锁：0变为1   | 谁使用谁上锁， 也只能由他开锁                                |

使用图形对比如下：

-   队列：

    -   里面可以放任意数据，可以放多个数据
    -   任务、ISR（中断服务处理）都可以放入数据；任务、ISR都可以从中读出数据

- 事件组：

    -   一个事件用一bit表示，1表示事件发生了，0表示事件没发生
    -   可以用来表示事件、事件的组合发生了，不能传递数据
    -   有广播效果：事件或事件的组合发生了，等待它的多个任务都会被唤醒
- 信号量：
  -   核心是“计数值”
  -   任务、ISR释放信号量时让计数值加1
  -   任务、ISR获得信号量时让计数值减1
-   任务通知：
    -   核心是任务的**TCB里**的数值
    -   会被覆盖
    -   发通知给谁？必须指定接收任务
    -   只能由接收任务本身获取该通知


-   互斥量：
    -   数值只有0或1
    -   谁获得互斥量，就必须由谁释放同一个互斥量

<img src="img\各种同步互斥方式对比示意.png" alt="各种同步互斥方式对比示意" style="zoom:67%;" />



### 2.1 队列

**传输数据的两种方法：**

-   拷贝： 把数据、变量的**值**复制进队列里
-   引用： 把数据、变量的**地址**复制进队列里

FreeRTOS使用**拷贝值**的方法，这更简单：

-   局部变量的值可以发送到队列中，后续即使函数退出、局部变量被回收，也不会影响队列中的数据
-   无需分配buffer来保存数据，队列中有buffer
-   局部变量可以马上再次使用
-   发送任务、接受任务解耦：接收任务不需要知道数据来源，发送任务也不需要释放数据
-   如果数据实在**太大**，可以使用队列传输它的地址
-   队列的空间由FreeRTOS内核分配，无需任务操心
-   对于有内存保护功能的系统，如果队列使用引用方法，也就是使用地址，必须确保双方任务对这个地址都有访问权限。使用拷贝方法时，则无此限制：内核有足够的权限，把数据复制进队列、再把数据复制出队列。

#### 2.1.1 队列的阻塞访问

任何任务、ISR都可以读写队列，可以多个任务读写队列（只需将队列句柄传给任务/ISR即可）

有任务要读队列，可队列中无数据怎么办？有任务要往队列中写数据，可队列满怎么办？

这两种情况任务若一直在执行的话会白白浪费处理器资源，如何解决？阻塞访问机制

任务如何重新回到就绪态？超时机制或有"空间"

多个任务的情况如何处理？哪个任务会率先从阻塞态进入到就绪态？

-   优先级最高的任务
-   相同优先级的情况下，等待时间最久的任务



#### 2.1.2 队列函数



**创建**

两种方法：动态分配内存、静态分配内存

**动态分配内存**

```C
QueueHandle_t xQueueCreate(UBaseType_t uxQueueLength, UBaseType_t uxItemSize);
```

| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| uxQueueLength | 队列长度，最多能存放多少个数据（item）                       |
| uxItemSize    | 每个数据（item）的大小，以字节为单位                         |
| 返回值        | 非0：成功，返回句柄，以后使用句柄来操作队列<br />NULL：失败，因为内存不足 |

**静态分配内存**

```C
QueueHandle_t xQueueCreateStatic(UBaseType_t uxQueueLength, UBaseType_t uxItemSize, uint8_t *pucQueueStorageBuffer, StaticQueue_t *pxQueueBuffer);
```

| 参数                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| pucQueueStorageBuffer | 如果uxitemSize非0，pucQueueStroageBuffer必须指向一个uint8_t数组，此数组大小至少为"uxQueueLength * uxItemSize" |
| pxQueueBuffer         | 必须指向一个StaticQueue_t结构体，用来保存队列的数据结构      |
| 返回值                | 非0：成功，返回句柄，以后使用句柄来操作队列<br />NULL: 失败，因为pxQueueBuffer为NULL |



**复位**

使用过程中可以调用`xQueueReset()`把队列恢复为初始状态，此函数原型为

```C
/* pxQueue: 复位哪个队列
 * 返回值： pdPASS（必定成功）
 */
BaseType_t xQueueReset(QueueHandle_t pxQueue);
```



**删除**

只能删除**使用动态方法创建**的队列，它会释放内存。

```C
void vQueueDelete(vQueueHandle_t xQueue);
```



**写队列**

```C
/* 等同于xQueueSendToBack 往队列尾部写入数据，如果没有空间，阻塞时间为xTicksToWait */
BaseType_t xQueueSend(QueueHandle_t xQueue, const void *pvItemToQueue, TickType_t xTicksToWait);

BaseType_t xQueueSendtoBack(QueueHandle_t xQueue, const void *pvItemToQueue, TickType_t xTicksToWait);

/* 往队列尾部写入数据，此函数可以在中断函数中使用，不可阻塞 */
BaseType_t xQueueSendToBackFromISR(QueueHandle_t xQueue, const void *pvItemToQueue, BaseType_t *pxHigherPriorityTaskWoken);

/*往队列头部写入数据，如果没有空间，阻塞时间为xTicksToWait*/
BaseType_t xQueueSendToFront(QueueHandle_t xQueue, const void *pvItemToQueue, TickType_t xTicksToWait);

/*往队列头部写入数据，此函数可以在中断函数中使用，不可阻塞*/
BaseType_t xQueueSendToFrontFromISR(QueueHandle_t xQueue, const void *pvItemToQueue, BaseType_t *pxHigherPriorityTaskWoken);
```

**参数说明**

| 参数                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| xQueue                    | 队列句柄，要写哪个队列                                       |
| pvItemToQueue             | 数据指针，这个数据的值会被复制进队列                         |
| xTicksToWait              | 表示阻塞的最大时间（Tick Count）<br />如果被设为0，无法写入数据时函数会立刻返回<br />如果被设为portMAX_DELAY，则会一直阻塞直到有空间可写 |
| 返回值                    | pdPASS：数据成功写入队列<br />errQUEUE_FULL：写入失败，因为队列满 |
| pxHigherPriorityTaskWoken | ISR调用该函数时，通知调用者有没有一个更高优先级的任务被唤醒，如果有，该参数将会被设置为非零值，以指示ISR需要切换上下文以唤醒优先级更高的任务，从而及时处理紧急任务或减少响应时间 |



**读队列**

```C
/* pvBuffer：buffer指针，队列的数据会被复制到这个buffer
 * pxTaskWoken：与pxHigherPriorityTaskWoken类似
 */
BaseType_t xQueueReceive(QueueHandle_t xQueue, void * const pvBuffer, TickType_t xTicksToWait);
BaseType_t xQueueReceive(QueueHandle_t xQueue, void *pvBuffer, BaseType_t *pxTaskWoken);
```



**查询**

```C
/* 返回队列中可用数据的个数 */
UBaseType_t uxQueueMessagesWaiting(const QueueHandle_t xQueue);

/* 返回队列中可用空间的个数 */
UBaseType_t uxQueueSpacesAvailable(const QueueHandle_t xQueue);
```





**覆盖/窥视**

当队列长度为1时，可以使用 `xQueueOverwrite()` 或 `xQueueOverwriteFromISR()` 来覆盖数据。 注意，队列长度必须为1。当队列满时，这些函数会**覆盖**里面的数据，这也意味这些函数不会被阻塞。

```C
BaseType_t xQueueOverWrite(QueueHandle_t xQueue, const void * pvItemToQueue);
BaseType_t xQueueOverWriteFromISR(QueueHandle_t xQueue, const void * pvItemToQueue, BaseType_t *pxHigherPriorityTaskWoken);
```



如果想让队列中的数据供多方读取，也就是说读取时不要移除数据，要留给后来人。那么可以使用"窥视"，这些函数会从队列中复制出数据，但是不移除数据。

```C
BaseType_t xQueuePeek(QueueHandle_t xQueue, void * const pvBuffer, TickType_t xTicksTowait);
BaseType_t xQueuePeekFromISR(QueueHandle_t xQueue, void *pvBuffer);
```



示例：**队列的基本使用**，见程序`8_queue Dynamic&static`

<img src="img/队列的基本使用任务调度情况.png" alt="队列的基本使用任务调度情况" style="zoom:67%;" />





示例：**数据来源**，见程序`9_queue_Datasource`

<img src="img/数据来源.png" alt="数据来源" style="zoom: 50%;" />

示例：传输大块数据，见程序`10_queue_bigtransfer`

使用地址来间接传输数据时，这些数据放在RAM里，对于RAM要保证：

-   RAM的所有者、操作者必须清晰明了，这块内存被称为”共享内存“，要确保不能同时修改RAM，比如，在写队列之前只能由发送者修改这块RAM，在读队列之后只能由接收者访问这块RAM。
-   RAM要保持可用。这块RAM应该是全局变量，或者是动态分配的内存，对于动态分配的内存，要确保它不能提前释放，要等到接收者用完后再释放。另外，不能是局部变量。

这个程序故意设置接受任务的优先级更高，**在它访问数组的过程中，发送任务无法执行、无法写入这个数组**。





示例：**Mailbox**，见程序`11_queue_mailbox`

-   它是一个队列，队列长度只有1
-   写邮箱：新数据**覆盖**旧数据，无论邮箱中是否有数据，这些函数总能成功写入。
-   读邮箱：**窥视**，读数据时数据不会被溢出，第一次调用会因为无数据而阻塞，一旦成功写入数据，以后读邮箱时总能成功。



### 2.2 信号量

**信号量与队列的对比**

| 队列                                                         | 信号量                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 可以容纳多个数据<br />创建队列时有两部分内存：队列结构体、存储数据的空间 | 只有计数值，无法容纳其他数据<br />创建信号量时，只需分配信号量结构体 |
| 生产者：没有空间存入数据时可以阻塞                           | 生产者：永远不阻塞，计数值达到最大时返回失败                 |
| 消费者：没有数据时可以阻塞                                   | 消费者：没有资源时可以阻塞                                   |



注意区分**二进制信号量**与**计数型信号量**

-   二进制信号量被创建时初始值为0
-   计数型信号量被创建时初始值可以设定



#### 信号量函数

**创建**

|          | 二进制信号量                                        | 计数型信号量                   |
| -------- | --------------------------------------------------- | ------------------------------ |
| 动态创建 | xSemaphoreCreateBinary<br />计数值初始值为0         | xSemaphoreCreateCounting       |
|          | vSemaphoreCreateBinary(已过时)<br />计数值初始值为1 |                                |
| 静态创建 | xSemaphoreCreateBinaryStatic                        | xSemaphoreCreateCountingStatic |

```C
/*返回句柄，函数内部会分配信号量结构体 返回值：返回句柄，非NULL表示成功*/
SemaphoreHandle_t xSemaphoreCreateBinary(void);

/*需要先有一个StaticSemaphore_t结构体*/
SemaphoreHandle_t xSemaphoreCreateBinaryStatic(StaticSemaphore_t* pxSemaphoreBuffer);

/*uxMaxCount：最大计数值 uxInitialCount：初始计数值*/
SemaphoreHandle_t xSemaphoreCreateCounting(UBaseType_t uxMaxCount, UBaseType_t uxInitialCount);

SemaphoreHandle_t xSemaphoreCreateCountingStatic(UBaseType_t uxMaxCount, UBaseType_t uxInitialCount, StaticSemaphore_t* pxSemaphoreBuffer);
```



**删除**

```c
void vSemaphoreDelete(SemaphoreHandle_t xSemaphore);
```



**give/take**

|      | 在任务中使用   | 在ISR中使用           |
| ---- | -------------- | --------------------- |
| give | xSemaphoreGive | xSemaphoreGiveFromISR |
| take | xSemaphoreTake | xSemaphoreTakeFromISR |

```C
BaseType_t xSemaphoreGive(SemaphoreHandle_t xSemaphore);

/*如果释放信号量导致更高优先级的任务变成了就绪态，则*pxHigherPriorityTaskWoken=pdTRUE*/
BaseType_t xSemahoreGiveFromISR(SemaphoreHandle_t xSemaphore, BaseType_t *pxHigherPriorityTaskWoken);

BaseType_t xSemaphoreTake(SemaphoreHandle_t xSemaphore, TickType_t xTicksToWait);

BaseType_t xSemahoreTakeFromISR(SemaphoreHandle_t xSemaphore, BaseType_t *pxHigherPriorityTaskWoken);
```



**示例：防止数据丢失** 见程序`14_Semaphore_circle_buffer`

在示例中，发送任务发出3次"提醒"，但是接收任务只接收到1次"提醒"，其中2次"提醒"丢失了。 

这种情况很常见，比如每接收到一个串口字符，串口中断程序就给任务发一次"提醒"，假设收到多个字符、发出了多次"提醒"。当任务来处理时，它只能得到1次"提醒"。 

你需要使用其他方法来防止数据丢失，比如： 

-   在串口中断中，把数据放入缓冲区
-   在任务中，一次性把缓冲区中的数据都读出,简单地说，就是：你提醒了我多次，我太忙只响应你一次，但是我一次性拿走所有数据



### 2.3 互斥量

**核心**在于：谁上锁，就只能由谁开锁

FreeRTOS的互斥锁并没有在代码上实现这一点：

-   即使任务A获得了互斥锁，任务B竟然也可以释放互斥锁
-   谁上锁谁释放只是一个约定

**关键**：对变量的非原子化访问

修改变量、设置结构体、在16位的机器上写32位的变量，这些操作都是非原子的。

**函数重入**：“可重入的函数”是指：多个任务同时调用它、任务和中断同时调用它，函数的运行也是**安全**的。可重入函数也被称为“线程安全”（thread safe）。每个任务都维持自己的栈、自己的CPU寄存器，如果一个函数只使用局部变量，那么它就是线程安全的。函数中一旦使用了全局变量、静态变量、其他外设，那它就不是“可重入的”，如果该函数正在被调用，就必须**阻止其他任务、中断再次调用它**。



#### 互斥量函数

**创建**

```C
SemaphoreHandle_t xSemaphoreCreateMutex(void);

SemaphoreHandle_t xSemaphoreCreateMutexStatic(StaticSemaphore_t *pxMutexBuffer);
```



需注意的是：互斥量不能在ISR中使用

其删除，give/take与一般信号量是一样的





示例：**优先级反转**

假设任务A、B都想使用串口，A优先级比较低： 

-   任务A获得了串口的互斥量 

-   任务B也想使用串口，它将会阻塞、等待A释放互斥量

-    高优先级的任务，被低优先级的任务延迟，这被称为"优先级反转"(priority inversion) 

如果涉及3个任务，可以让"优先级反转"的后果更加恶劣。

互斥量可以通过"优先级继承"，可以很大程度解决"优先级反转"的问题，这也是FreeRTOS中互斥量和二进制信号量的差别。

程序`17_mutex_inversion`使用二进制信号量来演示“优先级反转的恶劣后果”

程序运行流程：

-   A：HPTask优先级最高，它最先运行。在这里故意打印，这样才可以观察到flagHPTaskRun的脉冲。 

-   HP Delay：HPTask阻塞

-   B：MPTask开始运行。在这里故意打印，这样才可以观察到flagMPTaskRun的脉冲。

-   MP Delay：MPTask阻塞

-   C：LPTask开始运行，获得二进制信号量，然后故意打印很多字符 

-   D：HP Delay时间到，HPTask恢复运行，它无法获得二进制信号量，一直阻塞等待 

-   E：MP Delay时间到，MPTask恢复运行，它比LPTask优先级高，一直运行。导致LPTask无法运行，自然无法释放二进制信号量，于是HPTask用于无法运行。 

**总结**：

-   LPTask先持有二进制信号量， 
-   但是MPTask抢占LPTask，使得LPTask一直无法运行也就无法释放信号量
-   导致HPTask任务无法运行
-   优先级最高的HPTask一直无法运行

优先级反转运行流程图：

<img src="img/优先级反转运行流程.png" alt="优先级反转运行流程" style="zoom:67%;" />

优先级反转时序图

<img src="img/优先级反转时序图.png" alt="优先级反转时序图" style="zoom:50%;" />



示例：**优先级继承**

优先级继承： 

-   假设持有互斥锁的是任务A，如果更高优先级的任务B也尝试获得这个锁 
-   任务B说：你既然持有宝剑，又不给我，那就继承我的愿望吧 
-   于是任务A就继承了任务B的优先级 
-   这就叫：优先级继承 
-   等任务A释放互斥锁时，它就恢复为原来的优先级 
-   互斥锁内部就实现了优先级的提升、恢复

运行时序图如下图所示

-   A：HPTask执行 xSemaphoreTake(xLock, portMAX_DELAY); ，它的优先级被LPTask继承 
-   B：LPTask抢占MPTask，运行 
-   C：LPTask执行 xSemaphoreGive(xLock); ，它的优先级恢复为原来值 
-   D：HPTask得到互斥锁，开始运行 
-   互斥锁的"优先级继承"，可以减小"优先级反转"的影响

<img src="img/优先级继承时序图.png" alt="优先级继承时序图" style="zoom: 67%;" />



### 2.4 递归锁

特点：

-   任务A获得了递归锁M后，它还可以多次去获得这个锁
-   “take”了N次，要“give”N次，这个锁才会被释放



递归锁函数

|      | 递归锁                         | 一般互斥量            |
| ---- | ------------------------------ | --------------------- |
| 创建 | xSemaphoreCreateRecursiveMutex | xSemaphoreCreateMutex |
| 获得 | xSemaphoreTakeRecursive        | xSemaphoreTake        |
| 释放 | xSemaphoreGiveRecursive        | xSemaphoreGive        |

```C
SemaphoreHandle_t xSemaphoreCreateRecursiveMutex(void);
BaseType_t xSemaphoreGiveRecursive(SemaphoreHandle_t xSemaphore);
BaseType_t xSemaphoreTakeRecursive(SemaphoreHandle_t xSemaphore, TickType_t xTicksToWait);
```



示例：递归锁`20_mutex_recursive`

运行流程如下所示： 

-   A：任务1优先级最高，先运行，获得递归锁 
-   B：任务1阻塞，让任务2得以运行 
-   C：任务2运行，看看能否获得别人持有的递归锁：不能 
-   D：任务2故意执行"give"操作，看看能否释放别人持有的递归锁：不能 
-   E：任务2等待递归锁 
-   F：任务1阻塞时间到后继续运行，使用循环多次获得、释放递归锁 
-   **递归锁在代码上实现了**：谁持有递归锁，必须由谁释放。



### 2.5 事件组

**或与**的关系

事件组可以简单地认为就是一个整数：

-   每一位表示一个事件 
-   每一位事件的含义由程序员决定，比如：Bit0表示用来串口是否就绪，Bit1表示按键是否被按下 
-   这些位，值为1表示事件发生了，值为0表示事件没发生 
-   一个或多个任务、ISR都可以去写这些位；一个或多个任务、ISR都可以去读这些位 
-   可以等待某一位、某些位中的任意一个，也可以等待多位

事件组用一个整数来表示，其中的高8位留给内核使用，只能用其他的位来表示事件。那么这个整数是多少位的？

-   如果configUSE_16_BIT_TICKS是1，那么这个整数就是16位的，低8位用来表示事件 
-   如果configUSE_16_BIT_TICKS是0，那么这个整数就是32位的，低24位用来表示事件 
-   configUSE_16_BIT_TICKS是用来表示Tick Count的，怎么会影响事件组？这只是基于效率来考虑 
    -   如果configUSE_16_BIT_TICKS是1，就表示该处理器使用16位更高效，所以事件组也使用16 位 
    -   如果configUSE_16_BIT_TICKS是0，就表示该处理器使用32位更高效，所以事件组也使用32 位

事件组和队列、信号量不同的地方：

-   唤醒谁？
    -   队列、信号量：事件发生时，只会唤醒一个任务
    -   事件组：事件发生时，会唤醒所有符号条件的任务，即多播
-   是否清除事件？
    -   队列、信号量：是消耗型的资源，队列的数据被读走后就没了，信号量被获取后就少了
    -   事件组：被唤醒的任务有两个选择，可以让事件保持不动，也可以清除事件



#### **事件组函数**

```C
/*创建*/
EventGroupHandle_t xEventGroupCreate(void);
EventGroupHandle_t xEventGroupCreateStatic(StaticEventGroup_t *pxEventGroupBuffer);

/*删除*/
void vEventGroupDelete(EventGroupHandle_t xEventGroup);

/* 设置事件组中的位
 * uxBitsToSet：可以用来设置多个位，比如0x15就表示设置bit4, bit2, bit0
 * 返回值：返回原来的事件值（没什么意义，因为很可能已经被其他任务修改了）
 */
EventBits_t xEventGroupSetBits(EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToSet);

BaseType_t xEventGroupSetBitsFromISR(EventGroup_t xEventGroup, const EventBits_t uxBitsToSet, BaseType_t * pxHigherPriorityTaskWoken);

/* 等待事件 */
EventBits_t xEventGroupWaitBits(EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToWaitFor, const BaseType_t xClearOnExit, const BaseType_t xWaitForAllBits, TickType_t xTicksToWait);
```

**设置事件组注意事项**

>   值得注意的是，ISR中的函数，比如队列函数 `xQueueSendToBackFromISR`、信号量函数 `xSemaphoreGiveFromISR` ，它们会唤醒某个任务，最多只会唤醒1个任务。
>
>   但是设置事件组时，有可能导致多个任务被唤醒，这会带来很大的不确定性。所以 `xEventGroupSetBitsFromISR` 函数不是直接去设置事件组，而是给一个FreeRTOS后台任务(daemon task)发送队列数据，由这个任务来设置事件组。 
>
>   如果后台任务的优先级比当前被中断的任务优先级高， `xEventGroupSetBitsFromISR` 会设置 `*pxHigherPriorityTaskWoken `为pdTRUE。 如果daemon task成功地把队列数据发送给了后台任务，那么 `xEventGroupSetBitsFromISR` 的返回值就是pdPASS(成功)。

**等待事件注意事项**

>   `unblock condition`：一个任务在等待事件发生时，它处于阻塞状态；当期望的时间发生时，这个状态就叫“unblock condition”，称为“非阻塞条件成立”，当该条件成立后，任务即可变为就绪态。
>   可以使用`xEventGroupWaitBits()`等待期望的事件，它发生后再使用`xEventGroupClearBits()`来清除，但是这两个函数之间，有可能会被其他任务/中断抢占，他们可能会修改事件组。
>   可以通过将`xClearOnExit`设置为pdTRUE, 使得对事件组的测试、清零都在`xEventGroupWaitBits()`中完成，这是一个**原子操作**。

| 参数            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| xEventGroup     | 等待哪个事件组？                                             |
| uxBitsToWaitFor | 等待哪些位？哪些位要被测试？                                 |
| xWaitForAllBits | 怎么测试？是“AND”还是“OR”？<br />pdTRUE：等待的位，全部为1；<br />pdFALSE：等待的位，某一个为1即可 |
| xClearOnExit    | 函数退出前是否要清除事件？<br />pdTRUE：清除uxBitsToWaitFor指定的位<br />pdFALSE：不清除 |
| xTicksToWait    | 如果期待的事件未发生，阻塞多久？<br />0：判断后即刻返回<br />portMAX_DELAY：一直等到成功后才返回<br />期望的Tick Count |
| 返回值          | 返回的是事件值<br />如果期待的事件发生了，返回的是“非阻塞条件成立”时的事件值<br />如果是超时退出，返回的是超时时刻的事件值 |

**举例**

| 事件组的值 | uxBitsToWaitFor | xWaitForAllBits | 说明                                                         |
| ---------- | --------------- | --------------- | ------------------------------------------------------------ |
| 0100       | 0101            | pdTRUE          | 任务期望bit0、bit2都为1<br />当前只有bit2满足，任务进入阻塞态<br />当bit0、bit2都为1时退出阻塞态 |
| 0100       | 0110            | pdFALSE         | 任务期望bit1、bit2某一个为1<br />当前值满足，所以任务成功退出 |
| 0100       | 0110            | pdTRUE          | 任务期望bit1、bit2都为1<br />当前只有bit2满足，任务进入阻塞态<br />当bit1、bit2都为1时退出阻塞态 |

**任务协同同步点**

`xEventGroupSync()`可以同步多个任务

-   可以设置某位、某些位，表示自己做了什么事
-   可以等待某位、某些位，表示要等待其他任务
-   期望的事件发生后，`xEventGroupSync()`才会成功返回
-   `xEventGroupSync`成功返回后，会清除事件

```C
EventBits_t xEventGroupSync( EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToSet, const EventBits_t uxBitsToWaitFor, TickType_t xTicksToWait );
```





示例：**等待多个事件** 见程序`21_event_group_wait_multi_events`

执行流程如下：

-   A："炒菜任务"优先级最高，先执行。它要等待的2个事件未发生：洗菜、生火，进入阻塞状态
-   B："生火任务"接着执行，它要等待的1个事件未发生：洗菜，进入阻塞状态
-   C："洗菜任务"接着执行，它洗好菜，发出事件：洗菜，然后调用F等待"炒菜"事件
-   D："生火任务"等待的事件满足了，从B处继续执行，开始生火、发出"生火"事件 
-   E："炒菜任务"等待的事件满足了，从A出继续执行，开始炒菜、发出"炒菜"事件 
-   F："洗菜任务"等待的事件满足了，退出F、继续执行C



示例：**任务同步**，见程序`22_event_group_task_sync`

要点在于`xEventGroupSync()`

-   设置事件：表示自己完成了某个、某些事件
-   等待事件：跟别的任务同步
-   成功返回后，清除“等待的事件”



### 2.6 任务通知

使用队列、信号量、事件组等方法时，并不知道对方是谁。使用任务通知时，可以明确指定：通知哪个任务。

使用队列、信号量、事件组时，需要事先创建对应的结构体，双方通过中间的结构体通信：

<img src="img/通常任务通信流程.png" alt="通常任务通信流程" style="zoom:50%;" />

使用任务通知时，任务结构体TCB中就包含了内部对象，可以直接接收别人发过来的“通知”：

<img src="img/任务通知流程.png" alt="任务通知流程" style="zoom:50%;" />

任务通知的**优势**：

-   效率更高：使用任务通知来发送事件、数据给某个任务时，效率更高。比队列、信号量、事件组都有更大的优势。
-   更节省内存：使用其他方法时都要创建对应的结构体，使用任务通知时无需额外创建结构体。

任务通知的**限制**：

-   不能发送数据给ISR：

	ISR并没有任务结构体，所以无法使用任务通知的功能给ISR发送数据。但是ISR可以使用任务通知的功能，发数据给任务。
	
- 数据只能该任务独享

​		使用队列、信号量、事件组时，数据保存在这些结构体中，其他任务、ISR都可以访问这些数		据。使用任务通知时，数据存放在目标任务中，只有它可以访问这些数据。

>   在日常工作中，这个限制影响不大。因为很多场合是从多个数据源把数据发送给某个任务，而不是一个数据源的数据发送给多个任务。

-   无法缓冲数据

​		使用队列时，假设队列深度为N，那么它可以保持N个数据。

​		使用任务通知时，任务结构体中只有一个任务通知值，只能保持一个数据。

-   无法多播给多个任务

​		使用事件组可以同时给多个任务发送事件

​		使用任务通知，只能发送一个任务

-   如果发送受阻，**发送方无法进入阻塞状态等待**

​		假设队列已满，使用`xQueueSendToBack()`给队列发送数据时，任务可以进入阻塞状态等待发		送完成。

​		使用任务通知时，即使对方无法接收数据，发送方也无法阻塞等待，只能即刻返回错误。



#### 通知状态和通知值

```C
typedef struct tskTaskControlBlock
{
    ......
    /*configTASK_NOTIFICATION_ARRAY_ENTRIES = 1*/
    volatile uint32_t ulNotifiedValue[configTASK_NOTIFICATION_ARRAY_ENTRIES];
    volatile uint8_t  ulNotifyState[configTASK_NOTIFICATION_ARRAY_ENTRIES];
	......
}tskTCB;

#define taskNOT_WAITING_NOTIFICATION		((uint8_t) 0)  /*initial state*/
#define taskWAITING_NOTIFICATION			((uint8_t) 1)
#define taskNOTIFICATION_RECEIVED			((uint8_t) 2)
```

通知值可以有**很多种**类型：

-   计数值
-   位（类似事件组）
-   任意数值





#### 任务通知的使用

使用任务通知，可以实现轻量级的队列（长度为1）、邮箱（覆盖的队列）、计数型信号量、二进制信号量、事件组



**两类函数**

任务通知有两套函数：**简化版**与**专业版**

-   简化版函数的使用比较简单，它实际上也是使用专业版函数实现的
-   专业版支持更多参数，可以实现很多功能

|          | 简化版                                      | 专业版                              |
| -------- | ------------------------------------------- | ----------------------------------- |
| 发出通知 | xTaskNotifyGive<br />vTaskNotifyGiveFromISR | xTaskNotify<br />xTaskNotifyFromISR |
| 取出通知 | ulTaskNotifyTake                            | xTaskNotifyWait                     |

**简化版发出通知**：

-   直接给其他任务发出通知，使得通知值加1
-   并使得通知状态变为“pending”，也就是`taskNOTIFICATION_RECEIVED`, 表示有数据了、待处理，相当于直接唤醒对应的取出函数。

**简化版取出通知**：

-   如果通知值等于0，则阻塞（可以指定超时时间）
-   当通知值大于0时，任务从阻塞态进入就绪状态
-   当ulTaskNotifyTake返回之前，可以进行一些清理工作：把通知值减一，或者把通知值清0

```C
/*xTaskToNotify: 给哪个任务发通知； 返回值：返回pdPASS*/
BaseType_t xTaskNotifyGive(TaskHandle_t xTaskToNotify);

/*被通知的任务可能正处于阻塞态，此函数发出通知后，会把它从阻塞态切换为就绪态*/
/*
 *如果被唤醒的任务的优先级高于当前任务的优先级，则*pxHigherPriorityTaskWoken会被设置为
 *pdTRUE,表示在中断返回前要进行任务切换
 */
void vTaskNotifyGiveFromISR(TaskHandle_t xTaskHandle, BaseType_t *pxHigherPriorityTaskWoken);

/*xClearCountOnExit：返回前是否清0 pdTRUE:清零 pdFALSE：若通知值大于0则减1
 *xTicksTowait:任务进入阻塞态的超时时间，它在等待通知值大于0
  0：不等待，即刻返回； portMAX_DELAY:一直等待，直到通知值大于0
 *返回值：返回在清零或减一之前的通知值 
  若xTickToWait非0，则有两种情况：
  1. 大于0：在超时前，通知值被增加了
  2. 等于0：一直没有其他任务增加通知值，最后超时返回0
 */
uint32_t ulTaskNotifyTake(BaseType_t xClearCountOnExit, TickType_t xTicksToWait);
```





**专业版发出通知**：

-   让接收人物的通知值加1：这时`xTaskNotify()`等同于`xTaskNotifyGive()`
-   设置接收任务的通知值的某一位、某些位，这就是一个**轻量级的、更高效**的事件组
-   把一个新值写入接收任务的通知值：上一次的通知值被读走后，写入才成功。这就是轻量级的、长度为1的队列
-   用一个新值覆盖接受任务的通知值：无论上一次的通知值是否被读走，覆盖都成功。类似`xQueueOverwrite()`函数，这就是轻量级的邮箱。

**专业版取出通知：**

`xTaskNotifyWait()`

-   可以让任务等待（可以加上超时时间），等待状态为“pending”（也就是有数据）
-   还可以在函数进入、退出时，清除通知值的指定位

```C
/*怎么使用ulValue，由eAction参数决定
返回值：pdPASS：成功。pdFAIL：只有一种情况会失败，当eAction为eSetValueWithoutOverwrite，并且通知状态为“pending”（表示有新数据未读），这时就会失败*/
BaseType_t xTaskNotify(TaskHandle_t xTaskToNotify, uint32_t ulvalue, eNotifyAction eAction);

BaseType_t xTaskNotifyFromISR(TaskHandle_t xTaskToNotify, uint32_t ulvalue, eNotifyAction eAction, BaseType_t *pxHigherPriorityTaskWoken);

BaseType_t xTaskNotifywait(uint32_t ulBitsToClearOnEntry, uint32_t ulBitsToClearOnExit, uint32_t *pulNotificationValue, TickType_t xTicksToWait);
```

**eNotifyAction参数说明**

| 取值                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| eNoAction                 | 仅仅是更新通知状态为”pending“，未使用ulValue<br />这个选项相当于轻量级、更高效的二进制信号量 |
| eSetBits                  | 通知值 = 原来的通知值 \| ulValue，按位或<br />相当于轻量级的、更高效的事件组 |
| eIncrement                | 通知值 = 原来的通知值 + 1，未使用ulValue<br />相当于轻量级的、更高效的二进制信号量、计数型信号量<br />相当于`xTaskNotifyGive()`函数 |
| eSetValueWithoutOverwrite | 不覆盖。如果通知状态为”Pending“（表示有数据未读）<br />则此次调用xTaskNotify不做任何事，返回pdFAIL<br />如果通知状态不是”Pending“(表示没有新数据)<br />则通知值 = ulValue |
| eSetValueWithOverwrite    | 覆盖。无论通知状态是否为”Pending“，通知值 = ulValue          |

`xTaskNotifywait`**参数说明**

| 参数                 | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| ulBitsToClearOnEntry | 在xTaskNotifyWait入口处，要清除通知值的哪些位？ <br />通知状态不是"pending"的情况下，才会清除。 <br />它的本意是：我想等待某些事件发生，所以先把"旧数据"的某些位清零。 <br />能清零的话：通知值 = 通知值 & ~(ulBitsToClearOnEntry)。 <br />比如传入0x01，表示清除通知值的bit0； <br />传入0xffffffff即ULONG_MAX，表示清除所有位，即把值设置为0 |
| ulBitsToClearOnExit  | 在xTaskNotifyWait出口处，如果不是因为超时退出，而是因为得到了数据而退出时： <br />通知值 = 通知值 & ~(ulBitsToClearOnExit)。 <br />在清除某些位之前，通知值先被赋给"*pulNotificationValue"。 <br />比如入0x03，表示清除通知值的bit0、bit1； <br />传入0xffffffff即ULONG_MAX，表示清除所有位，即把值设置为0 |
| pulNotificationValue | 用来取出通知值。 <br />在函数退出时，使用ulBitsToClearOnExit清除之前，把通知值赋 给"*pulNotificationValue"。 <br />如果不需要取出通知值，可以设为NULL。 |
| xTicksToWait         | 任务进入阻塞态的超时时间，它在等待通知状态变为"pending"。    |
| 返回值               | 1. pdPASS：成功 这表示xTaskNotifyWait成功获得了通知： <br />可能是调用函数之前，通知状态就是"pending"； <br />也可能是在阻塞期间，通知状态变为了"pending"。 <br />2. pdFAIL：没有得到通知。 |



示例：**传输计数值**（轻量级信号量），见程序`23_tasknotify_transfer_count`

本程序创建2个任务： 

-   发送任务：把数据写入唤醒缓冲区，使用 `xTaskNotifyGive()` 让通知值加一 
-   接收任务：使用 `ulTaskNotifyTake()` 取出通知值，这表示字符数，一次性打印字符

|      | 信号量                                                       | 使用任务通知实现信号量                                       |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 创建 | SemaphoreHandle_t xSemaphoreCreateCounting( <br />                UBaseType_t uxMaxCount, <br />                UBaseType_t uxInitialCount ); | 无                                                           |
| Give | xSemaphoreGive( SemaphoreHandle_t xSemaphore );              | BaseType_t xTaskNotifyGive( TaskHandle_t xTaskToNotify );    |
| Take | xSemaphoreTake(<br/>                   SemaphoreHandle_t xSemaphore,<br/>                   TickType_t xBlockTime<br/>               ); | uint32_t ulTaskNotifyTake( <br />                                   BaseType_t xClearCountOnExit, <br />                                   TickType_t xTicksToWait <br />                                  ); |

示例：**传输任一值**（轻量级队列），见程序`24_tasknotify_transfer_value`

程序执行流程如下：

-   A：发送任务优先级最高，先执行。连续给对方任务发送3个字符，只成功了1次 
-   B：发送任务阻塞，让接收任务能执行 
-   C：接收任务读取通知值 
-   D：把读到的通知值作为字符打印出来 
-   E：再次读取任务通知，阻塞

**注：**任务通知值只有一个，**数据可能丢失**，设计程序要考虑这一点

|      | 队列                                                         | 使用任务通知实现队列                                         |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 创建 | QueueHandle_t xQueueCreate(<br/>                           UBaseType_t uxQueueLength,<br/>                           UBaseType_t uxItemSize<br/>                       ); | 无                                                           |
| 发送 | BaseType_t xQueueSend(<br/>                           QueueHandle_t xQueue,<br/>                           const void * pvItemToQueue,<br/>                           TickType_t xTicksToWait<br/>                      ); | BaseType_t xTaskNotify( <br/>                     TaskHandle_t xTaskToNotify, <br/>					 uint32_t ulValue, <br/>					 eNotifyAction eAction <br />               ); |
| 接收 | BaseType_t xQueueReceive( QueueHandle_t xQueue,<br/>                          void * const pvBuffer,<br/>                          TickType_t xTicksToWait <br />                    ); | BaseType_t xTaskNotifyWait( <br/>                      uint32_t ulBitsToClearOnEntry, <br/>					  uint32_t ulBitsToClearOnExit, <br/>					  uint32_t *pulNotificationValue, <br/>					  TickType_t xTicksToWait<br />                  ); |

示例：轻量**事件组** 见程序`25_tasknotify_event_group`

|          | 事件组                                                       | 使用任务通知实现事件组                                       |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 创建     | EventGroupHandle_t xEventGroupCreate( void )                 | 无                                                           |
| 设置事件 | EventBits_t xEventGroupSetBits( <br />                 EventGroupHandle_t xEventGroup,<br />                 const EventBits_t uxBitsToSet <br />                 ); | BaseType_t xTaskNotify( <br/>                     TaskHandle_t xTaskToNotify, <br/>					 uint32_t ulValue, <br/>					 eNotifyAction eAction <br />               ); |
| 等待事件 | EventBits_t xEventGroupWaitBits( <br/>                EventGroupHandle_t xEventGroup,<br/>                const EventBits_t uxBitsToWaitFor,<br/>                const BaseType_t xClearOnExit,<br/>                const BaseType_t xWaitForAllBits,<br/>                TickType_t xTicksToWait <br/>				); | BaseType_t xTaskNotifyWait( <br/>                      uint32_t ulBitsToClearOnEntry, <br/>					  uint32_t ulBitsToClearOnExit, <br/>					  uint32_t *pulNotificationValue, <br/>					  TickType_t xTicksToWait<br />                  ); |

## 3. 软件定时器(software timer)

无数的软件定时器要基于一个真实的定时器

在FreeRTOS里，软件定时器都是基于**系统滴答中断**（Tick Interrupt）

-   指定时间：启动定时器和运行回调函数两者之间的间隔被称为定时器的周期(period)
-   指定类型：
    -   一次性（One-shot timers）: 这类定时器启动后，它的中断回调函数只会被调用一次可以手动再次启动它，但它不会自启
    -   自动加载定时器（Auto-reload timers）：这类定时器启动后，时间到将会自启，使得回调函数被周期性地调用
-   指定要做什么事：即指定回调函数

软件定时器的两种状态：

-   运行（Running、Active）：运行态的定时器，当指定时间到达后，它的回调函数会被调用
-   休眠（Dormant）：休眠态的定时器还可以通过句柄来访问它，但它未运行，它的回调函数不会被调用



### 软件定时器的上下文

FreeRTOS是实时操作系统，它不允许在内核、在中断中执行不确定的代码。如果定时器函数很耗时，将会影响整个系统。所以在FreeRTOS中，不在Tick中断里执行定时器函数

定时器函数在一个任务中执行：RTOS Daemon Task，RTOS守护任务，之前被称为“Timer Server”，但它不仅仅是定时器相关，所以就有了现在的名字

当FreeRTOS的配置项`configUSE_TIMERS`被设置为1，启动调度器时会自动创建RTOS Daemon Task

用户程序要使用定时器时，是通过“定时器命令队列(timer command queue)”和守护任务交互，如下图所示

<img src="img/守护任务交互过程.png" alt="守护任务交互过程" style="zoom: 50%;" />

守护任务的优先级：`configTIMER_TASK_PRIORITY`，定时器命令队列的长度`configTIMER_QUEUE_LENGTH`



### 守护任务的调度

它的工作有两类：

-   处理命令：从命令队列里取出命令、处理
-   执行定时器的回调函数

能否及时处理定时器的命令、能否及时执行定时器的回调函数，**严重依赖于守护任务的优先级**。下面使用两个例子来演示

例子1：**守护任务的优先性级较低**

-   t1：Task1处于运行态，守护任务处于阻塞态。 守护任务在这两种情况下会退出阻塞态切换为就绪态：1.命令队列中有数据 2.某个定时器超时了。 至于守护任务能否马上执行，取决于它的优先级

-   t2：Task1调用 xTimerStart() 

    要注意的是， xTimerStart() 只是把"start timer"的命令发给"定时器命令队列"，使得守护任务 退出阻塞态。 在本例中，Task1的优先级高于守护任务，所以守护任务无法抢占Task1

-   t3：Task1执行完 xTimerStart() 

    但是定时器的启动工作由守护任务来实现，所以 xTimerStart() 返回并不表示定时器已经被启动了

-   t4：Task1由于某些原因进入阻塞态，现在轮到守护任务运行

    守护任务从队列中取出"start timer"命令，启动定时器

-   t5：守护任务处理完队列中所有的命令，再次进入阻塞态。Idle任务作为优先级最高的就绪态任务执行

-   注意：假设定时器在后续某个时刻tX超时了，超时时间是"tX-t2"，而非"tX-t4"，**从 xTimerStart() 函数被调用时算起**

<img src="img/守护任务优先级较低.png" alt="守护任务优先级较低" style="zoom:50%;" />

例子2：**守护任务的优先性级较高**

-   t1：Task1处于运行态，守护任务处于阻塞态。 

-    t2：Task1调用 xTimerStart() 

    要注意的是， xTimerStart() 只是把"start timer"的命令发给"定时器命令队列"，使得守护任务 退出阻塞态。 在本例中，守护任务的优先级高于Task1，所以守护任务抢占Task1，守护任务开始处理命令队列。 Task1在执行 xTimerStart() 的过程中被抢占，这时它无法完成此函数

- t3：守护任务处理完命令队列中所有的命令，再次进入阻塞态。 此时Task1是优先级最高的就绪态任务，它开始执行

- t4：Task1之前被守护任务抢占，对 xTimerStart() 的调用尚未返回。现在开始继续运行次函 数、返回 

- t5：Task1由于某些原因进入阻塞态。Idle任务作为优先级最高的就绪态任务执行

- 注：定时器的超时时间是基于调用`xTimerStart()`的时刻tX，而不是基于守护任务处理命令的时刻tY，假设超时时间是10个Tick，整个超时时间是“tx+10”，而非“tY+10”

<img src="img/守护任务优先级较高.png" alt="守护任务优先级较高" style="zoom:50%;" />



### 定时器回调函数

```C
void ATimerCallBack(TimerHandle_t xTimer);
typedef void (* TimerCallbackFunction_t)(TimerHandle_t xTimer);
```

定时器的回调函数是在守护任务中被调用的，守护任务**不是专为某个定时器服务**的，它还要处理其他定时器

所以，定时器的回调函数不能影响其他人：

-   回调函数要尽快执行，不能进入阻塞态
-   不能调用会导致阻塞的API函数，例如`vTaskDelay()`
-   可能调用`xQueueReceive()`**之类的**函数，但是超时时间要设为0，即刻返回，不可阻塞



### 软件定时器的函数

**自动加载定时器的状态转换图**

<img src="img/自动重装载.png" alt="自动重装载" style="zoom: 67%;" />

**一次性定时器的状态转换图**

<img src="img/一次性定时器.png" alt="一次性定时器" style="zoom:67%;" />

```C
/* 使用动态分配内存的方法创建定时器
* pcTimerName:定时器名字, 用处不大, 仅在调试时用到
* xTimerPeriodInTicks: 周期, 以Tick为单位
* uxAutoReload: 类型, pdTRUE表示自动加载, pdFALSE表示一次性
* pvTimerID: 回调函数可以使用此参数, 分辨是哪个定时器
* pxCallbackFunction: 回调函数
* 返回值: 成功则返回TimerHandle_t, 否则返回NULL
*/
TimerHandle_t xTimerCreate( const char * const pcTimerName, const TickType_t xTimerPeriodInTicks, const UBaseType_t uxAutoReload, void * const pvTimerID,
TimerCallbackFunction_t pxCallbackFunction );

/* 使用静态分配内存的方法创建定时器
 * pxTimerBuffer: 传入一个StaticTimer_t结构体, 将在上面构造定时器
 */
TimerHandle_t xTimerCreateStatic(const char * const pcTimerName, TickType_t xTimerPeriodInTicks, UBaseType_t uxAutoReload, void * pvTimerID, TimerCallbackFunction_t pxCallbackFunction, StaticTimer_t *pxTimerBuffer );

/* 删除定时器
 * 返回值：pdFAIL表示“删除命令”在xTicksToWait个Tick内无法写入队列
 		  pdPASS表示成功
 */
BaseType_t xTimerDelete(TimerHandle_t xTimer, TickType_t xTicksToWait);
```

定时器的很多API函数，都是通过发送“命令”到命令队列，由守护任务来实现

如果队列满，“命令”就无法即刻写入队列，可以指定一个超时时间`xTicksToWait`，等待一会儿

```C
BaseType_t xTimerStart( TimerHandle_t xTimer, TickType_t xTicksToWait );

/*ISR版本*/
BaseType_t xTimerStartFromISR( TimerHandle_t xTimer, BaseType_t *pxHigherPriorityTaskWoken );

BaseType_t xTimerStop( TimerHandle_t xTimer, TickType_t xTicksToWait );
BaseType_t xTimerStopFromISR( TimerHandle_t xTimer, BaseType_t *pxHigherPriorityTaskWoken );
```

注：`xTicksToWait`是把命令写入命令队列的超时时间，**不是定时器本身的超时时间**，**不是定时器本身的周期**

创建定时器时，设置了它的周期(period)。 `xTimerStart()` 函数是用来启动定时器。假设调用 `xTimerStart()` 的时刻是tX，定时器的周期是n，那么在 tX+n 时刻定时器的回调函数被调用。 

如果定时器已经被启动，但是它的函数尚未被执行，再次执行` xTimerStart()` 函数**相当于执行** `xTimerReset() `，重新设定它的启动时间。



**复位**

从定时器的状态转换图可以知道，使用 `xTimerReset()` 函数可以让定时器的状态从冬眠态转换为运行 态，相当于使用 `xTimerStart()` 函数。 如果定时器已经处于运行态，使用` xTimerReset()` 函数就相当于重新确定超时时间。假设调用 `xTimerReset()` 的时刻是tX，定时器的周期是n，那么 tX+n 就是重新确定的超时时间。

```C
BaseType_t xTimerReset( TimerHandle_t xTimer, TickType_t xTicksToWait );

BaseType_t xTimerResetFromISR( TimerHandle_t xTimer, BaseType_t *pxHigherPriorityTaskWoken );
```



**修改周期**

使用` xTimerChangePeriod() `函数，除了能修改它的周期外，还可以让定时器的状态从休眠态转换为运行态

修改定时器的周期时，会使用新的周期重新计算它的超时时间。假设调用 xTimerChangePeriod() 函数的时间tX，新的周期是n，则 tX+n 就是新的超时时间

```C
BaseType_t xTimerChangePeriod( TimerHandle_t xTimer, TickType_t xNewPeriod, TickType_t xTicksToWait );
BaseType_t xTimerChangePeriodFromISR( TimerHandle_t xTimer, TickType_t xNewPeriod, BaseType_t *pxHigherPriorityTaskWoken );
```



**定时器ID**

<img src="img/定时器结构体.png" alt="定时器结构体" style="zoom: 67%;" />

怎么使用定时器ID，完全由程序来决定： 

-   可以用来标记定时器，表示自己是什么定时器 

-   可以用来保存参数，给回调函数使用 

它的初始值在创建定时器时由 `xTimerCreate()` 这类函数传入，后续可以使用这些函数来操作：

-   更新ID：使用 `vTimerSetTimerID()` 函数 
-   查询ID：查询 `pvTimerGetTimerID()` 函数 

这两个函数不涉及命令队列，**它们直接操作定时器结构体**

```C
void *pvTimerGetTimerID(TimerHandle_t xTimer);
/* 参数*pvNewID：新ID*/
void vTimerSetTimerID(TimeHandle_t xTimer, void *pvNewID);
```



示例：**按键防抖动** 见程序`27_software_timer_readkey`

解决措施：

-   连续读很多次，直到数值稳定 ----> **浪费CPU资源**
-   使用定时器：**要结合中断使用**

第二种方法具体处理措施：

按下按键后

-   在t1产生中断，这时不马上确定按键，而是复位定时器，假设周期时20ms，超时时间 为"t1+20ms"
-   由于抖动，在t2再次产生中断，再次复位定时器，超时时间变为"t2+20ms"
-   由于抖动，在t3再次产生中断，再次复位定时器，超时时间变为"t3+20ms"
-   在"t3+20ms"处，按键已经稳定，读取按键值

<img src="img/按键抖动原理.png" alt="按键抖动原理" style="zoom:50%;" />



## 4. 中断管理

假设当前系统正在运行Task1时，用户按下了按键，触发了按键中断。这个中断的处理流程如下：

-   CPU跳到固定地址去执行代码，这个固定地址通常被称为**中断向量**。这个跳转是由硬件实现的
-   执行代码做什么？
    -   保存现场：Task1被打断，需要先保存Task1的运行环境，比如各类寄存器的值
    -   分辨中断、调用处理函数（这个函数被称为ISR，Interrupt service routine）
    -   恢复现场：继续运行Task1，或者运行其他优先级更高的任务

**ISR是在内核中被调用的**，ISR执行过程中，用户的任务无法执行。ISR要尽量快，否则：

-   其他低优先级的中断无法被处理：实时性无法保证
-   用户任务无法被执行：系统显得很卡顿
-   如果运行**中断嵌套**，情况将会更复杂，ISR越快越有助于中断嵌套

如果这个**硬件中断的处理**就是**很耗时间**呢？对于这类中断的处理就要分为两部分（即**中断延迟处理**）：

-   ISR：尽快做些**清理、记录工作**，然后触发某个任务
-   任务：更复杂的事情放在任务中处理
-   所以：需要在**ISR与任务之间进行通信**

在FreeRTOS中使用中断的几个原则：

-   FreeRTOS认为**任务与硬件无关**，任务的优先级与程序员决定，任务何时运行由调度器决定
-   ISR虽然也是使用软件实现的，但它被认为是**硬件特性的一部分**，因为它跟硬件密切相关
    -   何时执行？由硬件决定
    -   哪个ISR被执行？由硬件决定
-   **ISR的优先级高于任务**：即使是优先级最低的中断，它的优先级也高于任务。任务只有在没有中断的情况下才能执行。

### 两套API函数

为什么需要两套API函数？

-   很多API函数会导致**运行这个API的任务进入阻塞状态**，比如写队列时，如果队列已满，可以进入阻塞状态等待一会
-   ISR调用API函数时，ISR不是”任务“，ISR不能进入阻塞状态
-   所以，在任务与ISR中这些函数的功能是有差别的

为什么不使用同一套函数，比如在函数里分辨当前调用者是任务还是ISR呢？示例代码如下：

```C
BaseType_t xQueueSend(...)
{
    if (is_in_isr())
    {
        /* 把数据放入队列 */
        /* 不管是否成功都直接返回 */
    }
    else /* 在任务中 */
    {
        /* 把数据放入队列 */
        /* 不成功就等待一会再重试 */
    }
}

```

FreeRTOS使用两套函数而不是一套，有如下好处：

-   使用同一套函数的话，需要增加额外的判断代码、增加额外分支，使得函数更长、更复杂、难以测试
-   任务、ISR所需参数不一致
    -   在任务中调用：需要指定超时时间，表示如果不成功就阻塞一会儿
    -   在ISR中调用：不需要指定超时时间，无论是否成功都要即刻返回
    -   如果强行把两套函数揉在一起，将会导致参数臃肿，无效
-   移植FreeRTOS时，还需要提供监测上下文的函数，比如`is_in_isr()`
-   有些**处理器架构没有办法轻易分辨当前是处于任务中还是ISR中**，就需要额外添加更多、更复杂的代码

使用两套函数可以让程序更高效，但是也有一些缺点，比如你要使用第三方库函数时，即会在任务中 用它，也会在ISR中调用它。这个第三方库函数用到了FreeRTOS的API函数，你无法修改库函数。这个问题可以通过下列手段解决：

-   把中断的处理推迟到任务中进行（Defer Interrupt processing），在任务中调用库函数
-   尝试在库函数中使用”FromISR“函数
    -   在任务、ISR中都可以调用”FromISR“函数
    -   反过来不可以，非FromISR函数无法在ISR中使用
-   第三方库函数也许会提供OS抽象层，自行判断当前环境是在任务还是ISR中，分别调用不同的函数



### 两套函数API列表

| 类型                        | 在任务中           | 在ISR中                   |
| --------------------------- | ------------------ | ------------------------- |
| 队列(queue)                 | xQueueSendToBack   | xQueueSendToBackFromISR   |
|                             | xQueueSendToFront  | xQueueSendToFrontFromISR  |
|                             | xQueueReceive      | xQueueReceiveFromISR      |
|                             | xQueueOverwrite    | xQueueOverwriteFromISR    |
|                             | xQueuePeek         | xQueuePeekFromISR         |
| 信号量(semaphore)           | xSemaphoreGive     | xSemaphoreGiveFromISR     |
|                             | xSemaphoreTake     | xSemaphoreTakeFromISR     |
| 事件组(event group)         | xEventGroupSetBits | xEventGroupSetBitsFromISR |
|                             | xEventGroupGetBits | xEventGroupGetBitsFromISR |
| 任务通知(task notification) | xTaskNotifyGive    | vTaskNotifyGiveFromISR    |
|                             | xTaskNotify        | xTaskNotifyFromISR        |
| 软件定时器(software timer)  | xTimerStart        | xTimerStartFromISR        |
|                             | xTimerStop         | xTimerStopFromISR         |
|                             | xTimerReset        | xTimerResetFromISR        |
|                             | xTimerChangePeriod | xTimerChangePeriodFromISR |



### xHigherPriorityTaskWoken参数

`xHigherPriorityTaskWoken`的含义是：是否有更高优先级的任务被唤醒了。如果为pdTRUE，则意味着后面要进行任务切换。 

以写队列为例。 

任务A调用 `xQueueSendToBack()` 写队列，有几种情况发生：

-   队列满了，任务A阻塞等待，另一个任务B运行
-   队列没满，任务A成功写入队列，但是它导致另一个任务B被唤醒，任务B的优先级更高：任务B先运行 
-   队列没满，任务A成功写入队列，即刻返回

可以看到，在任务中调用API函数可能导致任务阻塞、任务切换，这叫做"context switch"，**上下文切 换**。这个函数可能很长时间才返回，**在函数的内部实现了任务切换**。



`xQueueSendToBackFromISR()` 函数也可能导致任务切换，但是不会在函数内部进行切换，而是返回一个参数：表示是否需要切换，函数原型与用法如下：

```C
/* 往队列尾部写入数据，此函数可以在中断函数中使用，不可阻塞 */
BaseType_t xQueueSendToBackFromISR(QueueHandle_t xQueue, const void *pvItemToQueue, BaseType_t *pxHigherPriorityTaskWoken);
/* 用法示例 */
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
xQueueSendToBackFromISR(xQueue, pvItemToQueue, &xHigherPriorityTaskWoken);
if (xHigherPriorityTaskWoken == pdTRUE)
{
	/* 任务切换 */
}
```

`pxHigherPriorityTaskWoken`参数，就是用来保存函数的结果：是否需要切换

-   `*pxHigherPriorityTaskWoken`等于pdTRUE：函数的操作导致更高优先级的任务就绪了，ISR应该进行任务切换 
-   `*pxHigherPriorityTaskWoken`等于pdFALSE：没有进行任务切换的必要

为什么不在"FromISR"函数内部进行**任务切换，而只是标记一下而已呢？**

**为了效率**

```C
void XXX_ISR()
{
    int i;
    for (i = 0; i < N; i++)
    {
    	xQueueSendToBackFromISR(...); /* 被多次调用 */
        /* 任务打断不了中断的处理，所以在ISR中调度任务是无用功*/
    }
}
```

ISR中可能多次调用`FromISR`函数，如果在`FromISR`内部进行任务切换，会浪费时间。

解决方法是：

-   在`FromISR`中标记是否需要切换
-   在ISR返回之前再进行任务切换

示例代码如下：

```C
void XXX_ISR()
{
    int i;
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    for (i = 0; i < N; i++)
    {
    	xQueueSendToBackFromISR(...,&xHigherPriorityTaskWoken); /* 被多次调用 */
    }
    /* 最后再决定是否进行任务切换 */
    if (xHigherPriorityTaskWoken == pdTRUE)
    {
    	/* 触发任务切换 */
    }
}
```



上述例子很常见：例如在UART的ISR中读取多个字符，发现收到回车符后才进行任务切换

在ISR中调用API时不进行任务切换，而只是在`xHigherPriorityTaskWoken`中标记一下，除了效率，还有多种好处：

-   让ISR更可控：中断随机产生，在API中进行任务切换的话，可能导致问题更复杂
-   可移植性
-   在Tick中断中，调用 `vApplicationTickHook()` ：它运行于ISR，只能使用"FromISR"的函数

使用"FromISR"函数时，如果不想使用`xHigherPriorityTaskWoken`参数，可以设置为NULL



### 切换任务

FreeRTOS中使用两个宏进行任务切换：

```C
portEND_SWITCHING_ISR(xHigherPriorityTaskWoken);
portYIELD_FROM_ISR(xHigherPriorityTaskWoken)
```

这两个宏做的事情完全一样：

-   `portEND_SWITHING_ISR`使用汇编实现
-   `portYIELD_FROM_ISR`使用C语言实现

新版本FreeRTOS统一使用`portYIELD_FROM_ISR`

使用示例：

```C
void XXX_ISR()
{
    int i;
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    for (i = 0; i < N; i++)
    {
    	xQueueSendToBackFromISR(...,&xHigherPriorityTaskWoken); /* 被多次调用 */
    }
    /* 最后再决定是否进行任务切换
    * xHigherPriorityTaskWoken为pdTRUE时才切换
    */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```



### 中断延迟处理（Deferring Interrupt Processing）

示例：

-   t1：任务1运行，任务2阻塞 
-   t2：发生中断， 
    -   该中断的ISR函数被执行，任务1被打断 
    -   ISR函数要尽快快速地运行，它做一些必要的操作(比如清除中断)，然后唤醒任务2 - 
-   t3：在创建任务时设置任务2的优先级比任务1高(这取决于设计者)，所以ISR返回后，运行的是任务2，它要完成中断的处理。**任务2即"deferred processing task"，中断的延迟处理任务**
-   t4：任务2处理完中断后，进入阻塞态以等待下一个中断，任务1重新运行

<img src="img/中断延迟处理.png" alt="中断延迟处理" style="zoom:67%;" />



## 5. 资源管理

要独占式地访问临界资源，有3种方法：

-   公平竞争：使用互斥量，谁先获得互斥量谁就访问临界资源
-   谁要跟我抢，我就灭掉谁
    -   中断要抢？屏蔽中断
    -   其他任务要抢？禁止调度器，不运行任务切换

### 屏蔽中断

屏蔽一些优先级比较低的中断，而非**关闭所有中断**

![cortex-m3中断](img/cortex-m3中断.png)

`FreeRTOSconfig.h`中的两个中断宏：

`configMAX_SYSCALL_INTERRUPT_PRIORITY`与`configKERNEL_INTERRUPT_PRIORITY`

两者之间的中断叫做**用到SYSCALL的中断**，从0到宏`configMAX_SYSCALL_INTERRUPT_PRIORITY`的叫做非~

实际应用中可直接屏蔽某一类中断（例如使用SYSCALL的中断）



-   任务中使用：`taskENTER_CRITICAL()/taskEXIT_CRITICAL()`
-   ISR中使用：`taskENTER_CRITICAL_FROM_ISR()/taskEXIT_CRITICAL_FROM_ISR()`

示例代码：

```C
/* 在任务中，当前时刻中断是使能的
* 执行这句代码后，屏蔽中断
*/
taskENTER_CRITICAL();

/* 访问临界资源 */

/* 重新使能中断 */
taskEXIT_CRITICAL();
```

在`taskENTER_CRITICAL()/taskEXIT_CRITICAL()`之间：（ISR中以下三点同理）

-   低优先级的中断被屏蔽：优先级低于、等于 `configMAX_SYSCALL_INTERRUPT_PRIORITY`
-   高优先级的中断可以产生：优先级高于 `configMAX_SYSCALL_INTERRUPT_PRIORITY`
    -   但这些中断ISR里，**不允许使用FreeRTOS的API函数**
-   任务调度依赖于**中断**、依赖于**API函数**，所以：这两段代码之间，**不会有任务调度产生**

这套 `taskENTER_CRITICAL()/taskEXIT_CRITICAL() `宏，是可以递归使用的，它的内部会**记录嵌套的深度**，只有嵌套深度变为0时，调用` taskEXIT_CRITICAL() `才会重新使能中断

使用`taskENTER_CRITICAL()/taskEXIT_CRITICAL() `来访问临界资源是很**粗鲁**的方法： 

-   中断无法正常运行 
-   任务调度无法进行 

所以，这之间的代码要尽可能快速地执行



**在ISR中屏蔽中断示例代码**

```C
void vAnInterruptServiceRoutine(void)
{
    /* 用来记录当前中断是否使能 */
    UBaseType_t uxSavedInterruptStatus;
    
    /* 在ISR中，当前时刻中断可能是使能的，也可能是禁止的
	 * 所以要记录当前状态, 后面要恢复为原先的状态 */

    /*执行这句代码后，屏蔽中断*/
    uxSavedInterruptStatus = taskENTER_CRITICAL_FROM_ISR();
    
    /* 访问临界资源 */
    
    /* 恢复中断状态 */
	taskEXIT_CRITICAL_FROM_ISR( uxSavedInterruptStatus );
	/* 现在，当前ISR可以被更高优先级的中断打断了 */
}
```

重点注意`恢复`这两个字的意思

### 暂停调度

如果有别的任务竞争临界资源，可以把中断关掉：这当然可以禁止别的任务运行，但代价太大，会影响到**中断的处理**

如果只是禁止别的任务竞争，事实上不需要关中断，只需要暂停调度器即可，这期间中断还是可以发生、处理

```C
/*暂停调度器*/
void vTaskSuspendAll(void);

/*恢复调度器
 *返回值：pdTRUE表示在暂停期间有更高的任务就绪，可以不理会
 */
BaseType_t xTaskResumeAll(void);


/*示例代码*/
vTaskSuspendAll();

/*访问临界资源*/

xTaskResumeAll();
```

`vTaskSuspendAll()/xTaskResumeAll() `宏可以递归使用，它的内部会记录嵌套的深度，只有嵌套深度变为0时，调用`xTaskResumeAll()`才会重新使能调度器



## 6. 调试与优化

### 6.1 调试

FreeRTOS提供了很多调试手段：

* 打印
* 断言：`configASSERT  `
* Trace
* Hook函数(回调函数)

#### 6.1.1 打印

printf：FreeRTOS工程里使用了microlib，里面实现了printf函数。只需实现以下函数即可使用printf（一般使用串口打印）：

```C
int fputc( int ch, FILE *f );
```



#### 6.1.2 断言

一般的C库里面，断言就是一个函数：

```C
void assert(scalar expression);
```

它的作用是：确认expression必须为真，如果expression为假的话就中止程序。

在FreeRTOS里，使用`configASSERT()`，比如：

```C
#define configASSERT(x)  if (!x) while(1);
```

可以让它提供更多信息，比如如果程序出错，打印该程序位于哪个文件的哪个函数以及行号：

```C
#define configASSERT(x)  \
	if (!x) \
	{
		printf("%s %s %d\r\n", __FILE__, __FUNCTION__, __LINE__); \
        while(1); \
 	}
```

configASSERT(x)中，如果x为假，表示发生了很严重的错误，**必须停止系统的运行**

它用在很多场合，比如：

**队列操作**

```C
BaseType_t xQueueGenericSend( QueueHandle_t xQueue, const void * const pvItemToQueue, TickType_t xTicksToWait, const BaseType_t xCopyPosition )
{
    BaseType_t xEntryTimeSet = pdFALSE, xYieldRequired;
    TimeOut_t xTimeOut;
    Queue_t * const pxQueue = xQueue;

    configASSERT( pxQueue );
    configASSERT(!((pvItemToQueue == NULL) && (pxQueue->uxItemSize != (UBaseType_t)0U)));
    configASSERT( !((xCopyPosition == queueOVERWRITE) && (pxQueue->uxLength != 1 )));
	......
}
```

**中断级别的判断**

```C
void vPortValidateInterruptPriority( void )
{
	uint32_t ulCurrentInterrupt;
	uint8_t ucCurrentPriority;

	/* Obtain the number of the currently executing interrupt. */
	ulCurrentInterrupt = vPortGetIPSR();

	/* Is the interrupt number a user defined interrupt? */
	if( ulCurrentInterrupt >= portFIRST_USER_INTERRUPT_NUMBER )
	{
		/* Look up the interrupt's priority. */
		ucCurrentPriority = pcInterruptPriorityRegisters[ ulCurrentInterrupt ];

		configASSERT( ucCurrentPriority >= ucMaxSysCallPriority );
	}
    
    .......
}
```



#### 6.1.3 Trace

FreeRTOS中定义了很多trace开头的宏，这些宏被放在系统个关键位置。

它们一般都是空的宏，这不会影响代码：不影响编程处理的程序大小、不影响运行时间。

我们要调试某些功能时，可以**修改宏**：修改某些标记变量、打印信息等待。

| trace宏                                     | 描述                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| traceTASK_INCREMENT_TICK(xTickCount)        | 当tick计数自增之前此宏函数被调用。参数xTickCount当前的Tick值，它还没有增加。 |
| traceTASK_SWITCHED_OUT()                    | vTaskSwitchContext中，把当前任务切换出去之前调用此宏函数。   |
| traceTASK_SWITCHED_IN()                     | vTaskSwitchContext中，新的任务已经被切换进来了，就调用此函数。 |
| traceBLOCKING_ON_QUEUE_RECEIVE(pxQueue)     | 当正在执行的当前任务因为试图去读取一个空的队列、信号或者互斥量而进入阻塞状态时，此函数会被立即调用。参数pxQueue保存的是试图读取的目标队列、信号或者互斥量的句柄，传递给此宏函数。 |
| traceBLOCKING_ON_QUEUE_SEND(pxQueue)        | 当正在执行的当前任务因为试图往一个已经写满的队列或者信号或者互斥量而进入了阻塞状态时，此函数会被立即调用。参数pxQueue保存的是试图写入的目标队列、信号或者互斥量的句柄，传递给此宏函数。 |
| traceQUEUE_SEND(pxQueue)                    | 当一个队列或者信号发送成功时，此宏函数会在内核函数xQueueSend(),xQueueSendToFront(),xQueueSendToBack(),以及所有的信号give函数中被调用，参数pxQueue是要发送的目标队列或信号的句柄，传递给此宏函数。 |
| traceQUEUE_SEND_FAILED(pxQueue)             | 当一个队列或者信号发送失败时，此宏函数会在内核函数xQueueSend(),xQueueSendToFront(),xQueueSendToBack(),以及所有的信号give函数中被调用，参数pxQueue是要发送的目标队列或信号的句柄，传递给此宏函数。 |
| traceQUEUE_RECEIVE(pxQueue)                 | 当读取一个队列或者接收信号成功时，此宏函数会在内核函数xQueueReceive()以及所有的信号take函数中被调用，参数pxQueue是要接收的目标队列或信号的句柄，传递给此宏函数。 |
| traceQUEUE_RECEIVE_FAILED(pxQueue)          | 当读取一个队列或者接收信号失败时，此宏函数会在内核函数xQueueReceive()以及所有的信号take函数中被调用，参数pxQueue是要接收的目标队列或信号的句柄，传递给此宏函数。 |
| traceQUEUE_SEND_FROM_ISR(pxQueue)           | 当在中断中发送一个队列成功时，此函数会在xQueueSendFromISR()中被调用。参数pxQueue是要发送的目标队列的句柄。 |
| traceQUEUE_SEND_FROM_ISR_FAILED(pxQueue)    | 当在中断中发送一个队列失败时，此函数会在xQueueSendFromISR()中被调用。参数pxQueue是要发送的目标队列的句柄。 |
| traceQUEUE_RECEIVE_FROM_ISR(pxQueue)        | 当在中断中读取一个队列成功时，此函数会在xQueueReceiveFromISR()中被调用。参数pxQueue是要发送的目标队列的句柄。 |
| traceQUEUE_RECEIVE_FROM_ISR_FAILED(pxQueue) | 当在中断中读取一个队列失败时，此函数会在xQueueReceiveFromISR()中被调用。参数pxQueue是要发送的目标队列的句柄。 |
| traceTASK_DELAY_UNTIL()                     | 当一个任务因为调用了vTaskDelayUntil()进入了阻塞状态的前一刻此宏函数会在vTaskDelayUntil()中被立即调用。 |
| traceTASK_DELAY()                           | 当一个任务因为调用了vTaskDelay()进入了阻塞状态的前一刻此宏函数会在vTaskDelay中被立即调用。 |



编程时，一般的逻辑错误都容易解决。难以处理的是内存越界、栈溢出等。

#### 6.1.4 Malloc Hook函数

内存越界经常发生在堆的使用过程中。堆就是使用malloc得到的内存

并没有很好的方法检测内存越界，但可以提供一些回调函数：

使用pvPortMalloc失败时，如果在FreeRTOSConfig.h里配置`configUSE_MALLOC_FAILED_HOOK`为1，会调用`void vApplicationMallocFailedHook(void);`



#### 6.1.5 栈溢出Hook函数

在切换任务（`vTaskSwitchContext`）时调用`taskCHECK_FOR_STACK_OVERFLOW`来检测栈是否溢出，若溢出则会调用`void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)`

判断栈溢出有两种方法：

-   方法一：
    -   当前任务被切换出去之前，它的整个运行现场都被保存在栈里，这时**很可能**就是它对栈的使用到达了峰值。
    -   这方法很高效，但是并不精确
    -   比如：任务在运行过程中调用了函数A大量地使用了栈，调用完函数A后才被调度。

![栈信息1](img/栈信息1.png)

-   方法二：
    -   创建任务时，它的栈被填入固定的值，比如：0xa5
    -   检测栈里最后16字节的数据，如果不是0xa5的话表示栈即将、或者已经被用完了
    -   没有方法1快速，但是也足够快，能捕获**几乎所有**的栈溢出
    -   为什么是几乎所有？可能有些函数使用栈时，非常凑巧地把栈设置为0xa5

![栈信息2](img/栈信息2.png)





### 6.2 优化

FreeRTOS中可以查看任务使用CPU的情况、使用栈的情况，然后针对性地进行优化。

#### 6.2.1 栈使用情况

在创建任务时分配了栈，可以填入固定的数值比如0xa5，以后可以使用以下函数查看"栈的高水位"，也就是还有多少空余的栈空间（一般保持40+字节即可）：

```c
UBaseType_t uxTaskGetStackHighWaterMark( TaskHandle_t xTask );
```

原理是：从栈底往栈顶逐个字节地判断，它们的值持续是0xa5就表示它是空闲的。

| 参数/返回值 | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| xTask       | 哪个任务                                                     |
| 返回值      | 任务运行时、任务被切换时，都会用到栈。栈里原来值(0xa5)就会被覆盖。<br />逐个函数从栈的尾部判断栈的值连续为0xa5的个数，<br />它就是任务运行过程中空闲内存容量的最小值。<br />注意：假设从栈尾开始连续为0xa5的栈空间是N字节，返回值是N/4。 |

#### 6.2.2 任务运行时间统计

对于同优先级的任务，它们按照时间片轮流运行：你执行一个Tick，我执行一个Tick。

是否可以在Tick中断函数中，统计当前任务的累计运行时间？

不行！很不精确，因为有更高优先级的任务就绪时，当前任务还没运行一个完整的Tick就被抢占了。

我们需要比Tick更快的时钟，比如Tick周期时1ms，我们可以使用另一个定时器，让它发生中断的周期时0.1ms甚至更短。

使用这个定时器来衡量一个任务的运行时间，原理如下图所示：

<img src="img/03_task_statistics.png" alt="image-20211201150333865" style="zoom:50%;" />



* 切换到Task1时，使用更快的定时器记录当前时间T1
* Task1被切换出去时，使用更快的定时器记录当前时间T4
* (T4-T1)就是它运行的时间，累加起来
* 关键点：在`vTaskSwitchContext`函数中，使用**更快的定时器**统计运行时间



-   配置

```c
#define configGENERATE_RUN_TIME_STATS 1
#define configUSE_TRACE_FACILITY    1
#define configUSE_STATS_FORMATTING_FUNCTIONS  1
```

* 实现宏`portCONFIGURE_TIMER_FOR_RUN_TIME_STATS()`，它用来初始化更快的定时器

* 实现这两个宏之一，它们用来返回当前时钟值(更快的定时器)
    * portGET_RUN_TIME_COUNTER_VALUE()：直接返回时钟值
    * portALT_GET_RUN_TIME_COUNTER_VALUE(Time)：设置Time变量等于时钟值

代码执行流程：

* 初始化更快的定时器：启动调度器时

    ![image-20211201152037316](img/04_init_timer.png)

    

* 在任务切换时统计运行时间
    ![image-20211201152339799](img/05_cal_runtime.png)



获得统计信息，可以使用下列函数
* uxTaskGetSystemState：对于每个任务它的统计信息都放在一个TaskStatus_t结构体里
* vTaskList：得到的信息是可读的字符串，比如
* vTaskGetRunTimeStats：  得到的信息是可读的字符串，比如

**函数说明**

* uxTaskGetSystemState：获得任务的统计信息

```c
UBaseType_t uxTaskGetSystemState( TaskStatus_t * const pxTaskStatusArray, const UBaseType_t uxArraySize, uint32_t * const pulTotalRunTime );
```

| 参数              | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| pxTaskStatusArray | 指向一个TaskStatus_t结构体数组，用来保存任务的统计信息。<br />有多少个任务？可以用`uxTaskGetNumberOfTasks()`来获得。 |
| uxArraySize       | 数组大小、数组项个数，必须大于或等于`uxTaskGetNumberOfTasks()` |
| pulTotalRunTime   | 用来保存当前总的运行时间(更快的定时器)，可以传入NULL         |
| 返回值            | 传入的pxTaskStatusArray数组，被设置了几个数组项。<br />注意：如果传入的uxArraySize小于`uxTaskGetNumberOfTasks()`，返回值就是0 |



* vTaskList ：获得任务的统计信息，形式为可读的字符串。注意，pcWriteBuffer必须足够大。

```c
  void vTaskList( signed char *pcWriteBuffer );
```

可读信息格式如下：

  ![image-20211201152918891](img/06_task_list.png)

* vTaskGetRunTimeStats：获得任务的运行信息，形式为可读的字符串。注意，pcWriteBuffer必须足够大。

```c
void vTaskGetRunTimeStats( signed char *pcWriteBuffer );
```

  可读信息格式如下：

![image-20211201153040395](img/07_task_runtimestats.png)

## X.Tips

-   在包含FreeRTOS.h之后，再去包含其他头文件，比如： task.h、queue.h、semphr.h...
-   回收内存（TCB、栈）的操作在idle中完成，倘若内核无法执行到idle任务，而是一直徘徊在其他高优先级的任务时，最终将会因为内存耗尽而无法创建新任务
-   任务切换是需要时间的，因此每个时间片并不是完全被某个函数所利用的，其包含了任务切换的时间
-   获取当前的Tick Count：`xTaskGetTickCount()`
-   最后创建的最高优先级任务**先执行**，为什么？

