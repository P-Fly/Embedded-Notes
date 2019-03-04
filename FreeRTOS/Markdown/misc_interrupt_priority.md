# 中断优先级

## 前言

中断优先级是 FreeRTOS 中的一个重点内容，本章重点描述该部分内容。

## 总览

FreeRTOS 的中断管理如下：

![interrupt priority][1]

## 用户中断优先级

 - 中断优先级值越小，优先级越高。
 - 小于值 **configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY** 的中断由用户自行管理，不应该调用 FreeRTOS 的系统函数。
 - FreeRTOS 只关心 **configLIBRARY_LOWEST_INTERRUPT_PRIORITY** 和 **configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY** 之间的中断，这部分中断可以调用中断安全函数(以 FromISR 后缀的系统函数）。

## SVC Priority

 - 对于带 **MPU** 的 **Cortex-M**，**SVC** 的优先级由 **portNVIC_SVC_PRI** 定义：

    ```C
    #define portNVIC_SVC_PRI    ( ( ( uint32_t ) configMAX_SYSCALL_INTERRUPT_PRIORITY - 1UL ) << 24UL )
    ```

 - 对于不带 **MPU** 的 **Cortex-M**，**SVC** 的中断优先级为0。

    因为此时只有在启动调度器时需要 **SVC**，平时不需要，所以 FreeRTOS 没有初始化 **SVC**。

    详情可以参看[Priority initialization][2]

## PendSV Priority

**PendSV** 必须为最低优先级，因为需要考虑到中断嵌套。

如果在多个嵌套中断中会请求上下文切换，那么应该只让上下文切换程序执行一次，并且只有当所有嵌套中断完成时才执行切换。

详情可以参看[FreeRTOS sets the systick and pendsv priority to the lowest one][3]

## Systick Priority

FreeRTOS 只是建议以最低优先级运行 **Systick**，因为需要为更实时的应用程序预留更高的优先级，但这不是必须的。

还有一种观点认为 **Systick** 本身应该有更高的优先级，因为只有这样才能保证操作系统的 **real-time**。

详情可以参看[SysTick interrupt priority][4]

 [1]: ./images/interrupt_priority.jpg
 [2]: https://sourceforge.net/p/freertos/discussion/382005/thread/1a42e593/
 [3]: https://sourceforge.net/p/freertos/discussion/382005/thread/0dfa0de2/?limit=25#3265
 [4]: https://sourceforge.net/p/freertos/discussion/382005/thread/99f02a87/
