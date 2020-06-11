# 空闲任务

## 功能介绍

当调度器启动时，将自动创建空闲任务，它是在尽可能低的优先级创建的，以确保始终至少有一个任务能够运行。主要功能如下：
 - 释放不能立即删除的任务资源(延时释放)。
 - 提供低功耗支持。

[IDLE Task][1]

![prvIdleTask][2]

## 低功耗tickless模式

 [1]: https://www.freertos.org/RTOS-idle-task.html
 [2]: ./images/prvIdleTask.jpg
 [3]: ./images/low_power_support.jpg





