# 空闲任务

## 功能介绍

[IDLE Task][1]

当调度器启动时，将自动创建空闲任务，它是在尽可能低的优先级创建的，以确保始终至少有一个任务能够运行。主要功能如下：
 - 释放不能立即删除的任务资源(延时释放)。
 - 提供低功耗支持。

![prvIdleTask][2]

## 低功耗tickless模式

 ![Low Power Support][3]

**FreeRTOS** 实际使用 `portSUPPRESS_TICKS_AND_SLEEP` 来实现 **低功耗tickless模式**，其实现方式根据不同的硬件有所不同，但大致的流程如下：

 - 根据任务的状态，计算可以停止的系统节拍计数器的周期数。
 - 配置外部低功耗时钟（如 **RTC** 或 **Low Power Timer**），并停止系统节拍。
 - 系统进入低功耗状态（如 **WFI** 指令），等待唤醒。
 - 系统被唤醒。
 - 处理唤醒中断。
 - 根据外部低功耗时钟，恢复系统节拍计数器的配置。
 - 退出低功耗状态继续执行

 [1]: https://www.freertos.org/RTOS-idle-task.html
 [2]: ./images/prvIdleTask.jpg
 [3]: ./images/low_power_support.jpg
