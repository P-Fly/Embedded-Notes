# 系统时钟与任务延时

## 前言

FreeROTS 是由系统时钟驱动的。在每个系统时钟到来时，FreeRTOS 会检测是否有阻塞任务已经进入等待状态，或者是否需要切换到其它等待任务继续执行。

本章详细介绍了系统时钟的处理接口 **xTaskIncrementTick**，并分析了 FreeROTS 提供的任务延时接口。

## 时钟配置

FreeRTOS 提供了一些宏来配置系统时钟的频率：

**configCPU_CLOCK_HZ**： **CPU**的硬件时钟频率，该频率一般是晶振直接提供或通过锁相环锁定后的值。
**configTICK_RATE_HZ**： 系统的时钟频率，该频率代表系统1秒钟产生多少次时钟中断。

另外，当我们使用 FreeRTOS 提供的延时接口时，需要用 **pdMS_TO_TICKS** 将毫秒值转换为系统时钟的节拍数。

## xTaskIncrementTick

### 原型

``` C
BaseType_t xTaskIncrementTick( void );
```

需要注意，该函数是在临界状态下执行的：

 1. 中断环境下，需要使用 **vPortRaiseBASEPRI / vPortClearBASEPRIFromISR** 进行临界区保护。
 2. 线程环境下，需要使用 **taskENTER_CRITICAL / taskEXIT_CRITICAL** 进行临界区保护。

### 返回值

如果该函数返回真，说明有就绪态任务的优先级高于当前运行态的任务，需要手动发出一次 **PendSV** 中断，进行上下文切换。

### 流程

![xTaskIncrementTick][1]

该函数的流程图如上，该函数有两个运行场景，时钟中断和 **xTaskResumeAll**。

### 时钟中断



### xTaskResumeAll


### 常见问题

1. 滴答计数器溢出时为什么要交换链表

## 任务延时

### vTaskDelay

### vTaskDelayUntil

### xTaskAbortDelay

 [1]: ./images/xTaskIncrementTick.jpg
