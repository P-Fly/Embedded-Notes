# 系统时钟与任务延时

## 前言

FreeROTS 是由系统时钟驱动的。在每个系统时钟到来时，FreeRTOS 会检测是否有阻塞任务已经进入等待状态，或者是否需要切换到其它等待任务继续执行。

本章详细介绍了系统时钟的处理接口和调度器挂起 / 恢复接口，并分析了 FreeROTS 提供的任务延时接口。

## 时钟配置

FreeRTOS 提供了一些宏来配置系统时钟的频率：

 - **configCPU_CLOCK_HZ**： **CPU** 的硬件时钟频率，该频率一般是晶振直接提供或通过锁相环锁定后的值。
 - **configTICK_RATE_HZ**： 系统的时钟频率，该频率代表系统1秒钟产生多少次时钟中断。

## xTaskIncrementTick

### 功能

由实时内核调用，增加滴答计数，并检查是否有任务需要由阻塞状态切换到等待状态。

需要注意，该函数是在临界状态下执行的。在时钟中断和调度器恢复这两个场景中会被使用：

 1. 时钟中断场景：需要使用 **vPortRaiseBASEPRI / vPortClearBASEPRIFromISR** 进行临界区保护。
 2. 调度器恢复场景：需要使用 **taskENTER_CRITICAL / taskEXIT_CRITICAL** 进行临界区保护。

### 原型

``` C
BaseType_t xTaskIncrementTick( void );
```

### 返回值

如果该函数返回非0，表明应该发起一次上下文切换（有高优先级任务或同优先级任务需要切换）。

### 流程

![xTaskIncrementTick][1]

 1. 调度器正常运行的场景

    -  调度器正常情况下，首先将计数器 **xTickCount** 加1。
    -  如果计数器 **xTickCount** 为0，则说明计数器溢出，需要交换链表 **pxDelayedTaskList** 和 **pxOverflowDelayedTaskList**，并重新计算下一次唤醒的时间。
    - 遍历链表 **pxDelayedTaskList**，将所有到达等待时间的任务挂载到等待链表中。
    - 如果有必要（有更高优先级的任务进入等待状态或有其它同优先级的任务），则设置任务切换标志位作为函数的返回值返回。

 2. 调度器挂起的场景

    - 当调用 **vTaskSuspendAll** 挂起调度器后，在时钟中断中就不对阻塞任务进行处理，只会递增计数器 **uxPendedTicks**。
    - 当调用 **xTaskResumeAll** 恢复调度器时，会在该接口中反复调用 **uxPendedTicks** 次 **xTaskIncrementTick**接口。以模拟 FreeRTOS 挂起的这段时间内阻塞任务的状态变化。

## vTaskSuspendAll

### 功能

在不禁用中断的情况下挂起调度器。需要注意：

 - 不能在挂起状态下调用可能导致上下文切换的函数，比如 **vTaskDelay**，**vTaskDelayUntil**，**xQueueSend** 等。
 - 该函数与 **xTaskResumeAll** 配对使用，通过变量 **uxSchedulerSuspended** 记录调度器的状态，每次挂起时变量加1，恢复时变量减1。

### 原型

``` C
void vTaskSuspendAll( void );
```

## xTaskResumeAll

### 功能

恢复由 **vTaskSuspendAll** 挂起的调度器。该函数不会取消使用 **vTaskSuspend** 挂起的任务。

### 原型

``` C
BaseType_t xTaskResumeAll( void );
```

### 返回值

如果该函数返回真，表明已经发生过一次上下文切换（有高优先级任务或同优先级任务需要切换）。

### 流程

![xTaskResumeAll][2]

对于 **xTaskResumeAll**，主要功能有两点：

 1. 将消息链表 **xPendingReadyList** 中的任务挂载到等待链表中。
 2. 通过调用 **xTaskIncrementTick**，恢复处理因为调度器挂起而丢失的时钟节拍。

## vTaskDelay

### 功能

将任务阻塞一定时间。

### 原型

``` C
void vTaskDelay( const TickType_t xTicksToDelay );
```

### 参数

 - xTicksToDelay：需要阻塞的系统时钟节拍数。可以用 **pdMS_TO_TICKS** 将需要延时的毫秒值转换为系统时钟的节拍数。

### 其它

在 **vTaskDelay** 中，会调用 **prvAddCurrentTaskToDelayedList** 处理延时队列的挂载流程，在该接口中对唤醒时间的处理比较有趣。

![prvAddCurrentTaskToDelayedList][3]

如上图，假设当前时钟位置为 **xTickCount**，需要唤醒的时间为 **xTimeToWake**（**xTickCount** + **xTicksToWait**），此时唤醒时间有三个可能的区域：
 - 区域1：将任务挂载到 **pxDelayedTaskList** 链表上，并更新下一次唤醒时间 **xNextTaskUnblockTime**。
 - 区域2：将任务挂载到 **pxDelayedTaskList** 链表上。
 - 区域3：此时虽然唤醒时间已经溢出，但是它必定小于 **xTickCount**，并且大于延时链表所能代表的最大时间，可以理解为此部分区域被映射到了溢出延时链表上。因此需要将任务挂载到 **pxOverflowDelayedTaskList** 链表上。

## vTaskDelayUntil

### 功能

将任务阻塞到指定的时间。该函数可以被周期性的任务调用，以确保恒定的执行频率。

该函数与 **vTaskDelay** 最大的区别在于：**vTaskDelayUntil** 不会因为任务本身运行时间的长短影响到任务周期调度的准确性。

### 原型

``` C
void vTaskDelayUntil( TickType_t * const pxPreviousWakeTime, const TickType_t xTimeIncrement );
```

### 参数

 - pxPreviousWakeTime：指向一个变量，该变量保存任务最后一次解除阻塞的时间。第一次使用前，该变量必须初始化为当前时间。
 - xTimeIncrement：周期循环的时钟节拍数。可以用 **pdMS_TO_TICKS** 将需要延时的毫秒值转换为系统时钟的节拍数。

### 其它

在 **vTaskDelayUntil** 中，有三个需要休眠的场景：

![vTaskDelayUntil][4]

 - 1_1 当前系统时钟已经溢出，且唤醒时间处于 **xTickCount** 和 **pxPreviousWakeTime** 之间。
 - 2_1 当前系统时钟没有溢出，并且唤醒时间也没有溢出。
 - 2_2 当前系统时钟没有溢出，但唤醒时间已经溢出，但唤醒时间小于 **pxPreviousWakeTime**。

## xTaskAbortDelay

### 功能

强制任务离开阻塞状态，进入就绪状态。

### 原型

``` C
BaseType_t xTaskAbortDelay( TaskHandle_t xTask );
```

### 参数

 - xTask：需要操作的任务句柄。

### 返回值

如果任务没有处于阻塞状态，则返回假。否则返回真。

 [1]: ./images/xTaskIncrementTick.jpg
 [2]: ./images/xTaskResumeAll.jpg
 [3]: ./images/prvAddCurrentTaskToDelayedList.jpg
 [4]: ./images/vTaskDelayUntil.jpg
