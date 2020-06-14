# 软件定时器

## 简介

 [Software Timers][1]

软件定时器允许在未来设定的时间到来时，执行用户的回调函数。定时器从启动到回调函数被执行之间的时间被称为定时器的 **period**。

**FreeRTOS** 通过 **Timer Service Daemon** 管理软件定时器。该任务是一个单独的线程在运行，用户和软件定时器通过命令队列 `xTimerQueue` 进行通讯。当用户对软件定时器进行设置时，会向该队列发送命令，同时 **Timer Service Daemon** 会从队列中提取命令，作出相应的操作。

 ![Timer Command Queue][2]

因为软件定时器被任务化，因此不会在中断上下文中消耗任何处理时间，也不会在该状态下执行用户回调函数。

用户回调函数在定时器服务任务的上下文中执行。因此，用户回调函数中不应该调用会阻塞任务的函数。

## 定时器配置

 [Configuring Software Timers][3]

## One-Shot模式和Auto-Reload模式

 [One Shot VS Auto Reload][4]

 [1]: https://www.freertos.org/RTOS-software-timer.html
 [2]: ./images/timer_command_queue.jpg
 [3]: https://www.freertos.org/Configuring-a-real-time-RTOS-application-to-use-software-timers.html
 [4]: https://www.freertos.org/One-shot-Vs-auto-reload-real-time-software-timers.html
