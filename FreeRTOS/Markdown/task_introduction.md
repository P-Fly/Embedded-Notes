# 任务介绍

## 前言

FreeRTOS 是一个轻量级实时操作系统。它由一个个独立的用户任务组成，而每个用户任务都有自己独立的数据结构和代码。本章从全局的角度出发对任务进行描述。

## 任务的定义

FreeRTOS 的核心是任务管理，那么什么是任务？

在维基百科中，对 `Task` 的定义如下：

> In computing, a task is a unit of execution or a unit of work.
> ...
> In the sense of "unit of execution", in some operating systems, a task is synonymous with a process, and in others with a thread.
> ...

原来 `Task` 就是我们平时在程序中使用的 **Process** 或 **Thread**。当然，由于嵌入式系统与我们平时使用的 **PC** 系统相比，硬件结构要简单的多，编程模型也相对来说比较简单。因此，FreeRTOS 中没有对 **Process** 和 **Thread** 进行区分，而是统一使用 `Task` 作为基本的执行单元。

在此，我们引用 FreeRTOS 官网上的话来描述任务：

> A real time application that uses an RTOS can be structured as a set of independent tasks. Each task executes within its own context with no coincidental dependency on other tasks within the system or the RTOS scheduler itself. Only one task within the application can be executing at any point in time and the real time RTOS scheduler is responsible for deciding which task this should be. The RTOS scheduler may therefore repeatedly start and stop each task (swap each task in and out) as the application executes.

![Task][1]

## 全局数据

### TCB_t

FreeRTOS 的核心是任务管理，每个任务都有一些数据需要存储，包括任务状态，堆栈指针，统计数据等。FreeRTOS 把这些数据集合到一个数据结构中进行管理，这个数据结构就是结构体 `TCB_t`，被称作**任务控制块**。

每个任务都有各自独立的 `TCB_t`，一旦任务建立了，任务控制块将被赋值。当任务的 **CPU** 使用权被剥夺时，FreeRTOS 会用该结构体来保存该任务的状态。当任务重新得到 **CPU** 使用权时，任务控制块能确保任务从被中断的位置继续执行。

`TCB_t` 的成员如下：

> volatile StackType_t *pxTopOfStack;

指向栈顶（最后一次压栈的位置）。在汇编中会利用该指针进行压栈和出栈操作。比如 `xPortPendSVHandler` 的实现。

> xMPU_SETTINGS xMPUSettings;

`MPU` 的配置项，记录了内存段的基地址和访问属性。

> ListItem_t xStateListItem;

状态链表的节点项，FreeRTOS 根据任务所处的不同状态，将任务插入到对应的全局任务链表中（Ready, Delay, Suspended）。

> ListItem_t xEventListItem;

事件链表的节点项，FreeRTOS 根据任务所依赖的不同事件，将其插入到对应的事件链表中（Queue）。

> UBaseType_t uxPriority;

任务的优先级，**0** 为最低优先级。

> StackType_t *pxStack;

指向栈空间的起始地址。

> char pcTaskName[ configMAX_TASK_NAME_LEN ];

任务的名称。

> StackType_t *pxEndOfStack;

指向栈底。下图展示了 `pxTopOfStack` ， `pxEndOfStack` ， `pxStack` 之间的关系。

![stack][2]

另外我们可以使用宏 `taskCHECK_FOR_STACK_OVERFLOW` 去检测堆栈是否越界。

> UBaseType_t uxCriticalNesting;

用于临界区嵌套计数，**Cortex-M** 不使用该变量，取而代之的是一个全局临界区嵌套计数。

下面的链接简单说了下什么场景下可能需要在 `TCB_t` 中使用该嵌套计数：

[FreeRTOS Forum][3]

> A task can, if it wishes (and care is taken) yield while it is within a critical section.  Therefore each task needs to maintain its own critical nesting state.  If a task yields from within a critical section interrupts must again be disabled when the task starts to run again.

> UBaseType_t uxTCBNumber;

调试追踪时使用，对该任务提供一个可以追溯的数值。

> UBaseType_t uxTaskNumber;

调试追踪时使用，用户可以设置自定义数值。

> UBaseType_t uxBasePriority;

> UBaseType_t uxMutexesHeld;

在互斥锁中提供优先级继承机制，解决优先级反转的问题。

> TaskHookFunction_t pxTaskTag;

向用户提供一个可以设置回调函数的指针。FreeRTOS 本身不会使用该指针。

> void *pvThreadLocalStoragePointers[ configNUM_THREAD_LOCAL_STORAGE_POINTERS ];

存储一些本地数据的指针，FreeRTOS 本身不会使用该指针。

> uint32_t ulRunTimeCounter;

记录任务**运行状态**的总时间。

> struct _reent xNewLib_reent;

提供对 newlib 的支持，FreeRTOS 本身不需要关心该变量。

> volatile uint32_t ulNotifiedValue;

> volatile uint8_t ucNotifyState;

提供任务通知机制。

> uint8_t ucStaticallyAllocated;

标识 **Stack** 和 **TCB** 是静态定义的还是动态申请的，在 `vTaskDelete` 中根据该状态字进行相应的注销操作。

> uint8_t ucDelayAborted;

标识该任务是否需要立即唤醒，可以使用 `xTaskAbortDelay` 停止延时等待。

> int iTaskErrno;

记录该任务的errno。

### pxCurrentTCB

指向当前正在运行的任务。在任何时刻，FreeRTOS 都只有一个任务处于 **eRunning** 状态。

### 全局链表

 - **pxReadyTasksLists**：由多个链表头组成的链表数组，按照优先级跟踪就绪态任务，每个链表头代表一组相同优先级的任务。
 - **pxDelayedTaskList**：延时列表指针，如果唤醒时间（**xTickCount + xTicksToDelay**）未溢出，就放入到该链表中。
 - **pxOverflowDelayedTaskList**：溢出延时列表指针，如果唤醒时间（**xTickCount + xTicksToDelay**）溢出，就放入到该链表中。
 - **xPendingReadyList**：某些任务进入就绪态时，无法放入 **pxReadyTasksLists** 中。比如：调度器已经被挂起，此时放入该链表，等待调度器恢复。注意：放入到该链表中的是 **pxTCB->xEventListItem**，因此，我们可以认为这个链表是一个消息链表。关于链表 **xPendingReadyList** 更详细的讨论，可以参看文档：[xPendingReadyList链表][5]。
 - **xTasksWaitingTermination**：**eRunning** 状态的任务被删除时，无法释放资源，需要使用该链表进行跟踪，以便后续在 **IDLE** 任务中释放。
 - **xSuspendedTaskList**：跟踪被挂起的任务（无限期等待的任务）。

### 全局变量

 - **uxCurrentNumberOfTasks**：当前管理的任务数，包括所有就绪、阻塞和挂起的任务。已删除但空闲任务尚未释放的任务也将包含在计数中。
 - **xTickCount**：内核的滴答计数。
 - **uxTopReadyPriority**：在 **Cortex-M** 中以Bit位记录当前哪些优先级被使用，能通过该变量快速找出最高优先级。
 - **xSchedulerRunning**：记录调度器是否处于运行状态。
 - **uxPendedTicks**：当使用`vTaskSuspendAll`挂起调度器时，该变量跟踪这期间未被处理的Tick数量。
 - **xYieldPending**：记录是否需要重新调度。
 - **xNumOfOverflows**：记录定时器溢出次数。
 - **uxTaskNumber**：记录全局任务数，每个任务会被分配一个唯一的 **Number** 作为 **ID**。
 - **xNextTaskUnblockTime**：记录延时队列下一次需要唤醒的时间点。
 - **xIdleTaskHandle**：空闲任务的句柄。
 - **uxSchedulerSuspended**：调度器挂起标志。
 - **ulTaskSwitchedInTime**：记录内核上次切换时的时间值。
 - **ulTotalRunTime**：记录内核总运行时间。

## 任务的状态

### 状态描述

 - **Running**：
    - **pxCurrentTCB** 所指向的任务。
 - **Ready**：
    - 属于 **pxReadyTasksLists** 链表数组的任务。
    - 被挂载到消息链表 **xPendingReadyList** 上的任务，此时任务的状态项（**xStateListItem**）可能依然处于 **Delay** 或 **Suspended** 状态。
 - **Blocked**：
    - 属于 **pxDelayedTaskList** 链表的任务，该组任务可能被超时唤醒，也可能被事件唤醒。
    - 属于 **pxOverflowDelayedTaskList** 链表的任务，该组任务可能被超时唤醒，也可能被事件唤醒。
    - 属于 **xSuspendedTaskList** 链表，但正在等待唤醒消息的任务。
    - 属于 **xSuspendedTaskList** 链表，但正在等待任务通知的任务。
 - **Suspended**：
    - 属于 **xSuspendedTaskList** 链表，并且没有等待唤醒消息和任务通知。
 - **Deleted**：
    - 属于 **xTasksWaitingTermination** 链表的任务。
    - 不属于任何链表的任务。

### 状态迁移

![Task States][4]

### 任务的状态和全局链表的关系

需要注意 [任务的状态](./#任务的状态)  和 [全局链表](./#全局链表) 是程序的两个维度，是不能直接对应的：

 - **任务的状态** 是从用户的维度来看待任务：
    - **Blocked** 是指会在一定条件下被唤醒的任务，包括：Ticks、Event、Notify 等。
    - **Suspended** 是指不会被唤醒的任务（只能由用户使用 `vTaskSuspend`/`vTaskResume` 手动休眠/唤醒）。

 - **全局链表** 是从内核的维度来看待任务（只与等待周期相关，而与事件消息无关）：
    - 如果任务会在一定的周期后被唤醒，就会挂载到链表 **pxDelayedTaskList** 或 **pxOverflowDelayedTaskList** 上。比如：
        - 使用 `xQueueReceive` 等待消息接收，超时时间不为 **portMAX_DELAY**。
        - 使用 `vTaskDelay` 等待一段时间被唤醒（该函数不支持 **portMAX_DELAY**）。

    - 如果任务将无限期的等待，就会挂载到链表 **xSuspendedTaskList** 上，比如：
        - 使用 `xQueueReceive` 等待消息接收，超时时间为 **portMAX_DELAY**。
        - 使用 `vTaskSuspend` 挂起任务。

### 查询接口

FreeRTOS 提供了多个任务状态和任务信息的查询接口：

 - **eTaskGetState**：获取某个任务的状态。
 - **vTaskGetInfo**：获取某个任务的信息，可以通过标志位跳过某些需要花费较长时间的信息。
 - **uxTaskGetSystemState**：获取系统中所有任务的信息。
 - **vTaskList**：将 **uxTaskGetSystemState** 获取的任务信息以字符串形式输出。

 [1]: ./images/task.jpg
 [2]: ./images/stack_growth.jpg
 [3]: https://sourceforge.net/p/freertos/discussion/382005/thread/4b56fac4/
 [4]: ./images/task_states.jpg
 [5]: misc_xpendingreadylist.md
