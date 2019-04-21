# xPendingReadyList 链表

## 前言

本章介绍了链表 **xPendingReadyList** 相关的内容。

## 特殊性

FreeRTOS 内部有很多链表，但是这些链表都是使用状态项 **xStateListItem** 进行挂载。只有 **xPendingReadyList** 使用消息项 **xEventListItem** 进行挂载。

因此链表 **xPendingReadyList** 更像一个消息链表，用于接收 **ISR** 发送给任务的消息。

## 为什么需要该链表

详情可以参看[FreeRTOS kernel code problem][1]，简单来说原因如下：

 1. FreeRTOS 使用了两种方式来访问临界区：
    - **taskENTER_CRITICAL / taskEXIT_CRITICAL**：关闭中断的方式，此时系统无法接收任何中断。
    - **vTaskSuspendAll / xTaskResumeAll**：关闭调度器的方式，此时系统不产生任务调度，但是可以接收并处理中断。
 2. FreeRTOS 为了提高系统可靠性，减少丢失中断的可能，任务中经常使用 **vTaskSuspendAll / xTaskResumeAll** 的方式来访问临界区。
 3. 而对于 FreeRTOS 来说，在中断中使用的某些接口可能涉及任务的状态变化，比如 **xTaskResumeFromISR** 会使任务由 **Suspended** 状态切换到 **ready** 状态。如果在中断中关闭调度器时直接操作链表，很有可能会与任务中操作该链表的接口产生竞争冲突，严重时甚至导致系统崩溃。
 4. 因此我们将链表 **xPendingReadyList** 作为消息链表，将中断中需要进行的操作记录下来，在调度器恢复时再进行操作。

 [1]: https://sourceforge.net/p/freertos/discussion/382005/thread/90f6213f/?limit=25#7c9e
