# 任务介绍

## 前言





## 任务的定义

FreeRTOS 的核心是任务的管理，那么什么是任务？

在维基百科中，对 `Task` 的定义如下：

> In computing, a task is a unit of execution or a unit of work.
> ...
> In the sense of "unit of execution", in some operating systems, a task is synonymous with a process, and in others with a thread.
> ...

原来 `Task` 就是我们平时在程序中使用的 **process** 或 **thread**。当然，由于嵌入式系统与我们平时使用的 PC 系统相比，硬件结构要简单的多，编程模型也相对来说比较简单。因此，FreeRTOS 中没有对 **process** 和 **thread** 进行区分，而是统一使用 `Task` 作为基本的执行单元。

在此，我们引用 RTOS 官网上的话来描述一下任务：

> A real time application that uses an RTOS can be structured as a set of independent tasks. Each task executes within its own context with no coincidental dependency on other tasks within the system or the RTOS scheduler itself. Only one task within the application can be executing at any point in time and the real time RTOS scheduler is responsible for deciding which task this should be. The RTOS scheduler may therefore repeatedly start and stop each task (swap each task in and out) as the application executes. 

![Task][1]

## 全局数据

## 数据结构 TCB_t

FreeRTOS 的核心是任务管理，每个任务都有一些数据需要存储，包括任务状态，堆栈指针，统计数据等。FreeRTOS 把这些数据集合到一个数据结构中进行管理，这个数据结构就是结构体 `TCB_t`，被称作任务控制块。

每个任务都有各自独立的 `TCB_t` ，一旦任务建立了，任务控制块将被赋值。当任务的 CPU 使用权被剥夺时，FreeRTOS 用该结构体来保存该任务的状态。当任务重新得到 CPU 使用权时，任务控制块能确保任务从被中断的位置继续执行。

`TCB_t` 的成员如下：

>	volatile StackType_t	*pxTopOfStack;

指向栈顶（最后一次压栈的位置）。在汇编中会利用该指针进行压栈和出栈操作。比如 `xPortPendSVHandler` 的实现。 *TBD：增加链接*

>	xMPU_SETTINGS	xMPUSettings;

MPU 的配置项，记录了内存段的基地址和访问属性。 *TBD：关于MPU的链接*

>	ListItem_t			xStateListItem;

状态链表的节点项，FreeRTOS 根据任务所处的不同状态，将任务插入到对应的全局任务链表中（Ready, Blocked, Suspended）。 *TBD: 全局链表图的链接*

>	ListItem_t			xEventListItem;

事件链表的节点项，FreeRTOS 根据任务所依赖的不同事件，将其插入到对应的事件链表中（Queue）。 *TBD：事件链表的链接*

>	UBaseType_t			uxPriority;

任务的优先级， **0** 为最低优先级。

>	StackType_t			*pxStack;

指向栈空间的起始地址。

>	char				pcTaskName[ configMAX_TASK_NAME_LEN ];

任务的名称，只用于调试。

>	StackType_t		*pxEndOfStack，;

指向栈底。下图展示了 `pxTopOfStack` ， `pxEndOfStack` ， `pxStack` 之间的关系。

![stack][2]

另外我们可以使用宏 `taskCHECK_FOR_STACK_OVERFLOW` 去检测堆栈是否越界。

>	UBaseType_t		uxCriticalNesting;

用于临界区嵌套计数，Cortex-M3 不使用该变量，取而代之的是一个全局的临界区嵌套计数。

下面的链接简单说了下什么场景下可能需要在 `TCB_t` 中使用该嵌套计数：

[FreeRTOS Forum][3]

> A task can, if it wishes (and care is taken) yield while it is within a critical section.  Therefore each task needs to maintain its own critical nesting state.  If a task yields from within a critical section interrupts must again be disabled when the task starts to run again.

>	UBaseType_t		uxTCBNumber;

调试追踪时使用， 对该任务提供一个可以追溯的数值。

>	UBaseType_t		uxTaskNumber;

调试追踪时使用，用户可以设置自定义数值。

>	UBaseType_t		uxBasePriority;
>	UBaseType_t		uxMutexesHeld;

在互斥锁中提供优先级继承机制，解决优先级反转的问题。 *TBD：优先级反转的链接*

>	TaskHookFunction_t pxTaskTag;

向用户提供一个可以设置回调函数的指针。 FreeRTOS 本身没有使用该指针。

>	void	*pvThreadLocalStoragePointers[ configNUM_THREAD_LOCAL_STORAGE_POINTERS ];

存储一些本地数据的指针， FreeRTOS 本身不会使用该组指针。

>	uint32_t		ulRunTimeCounter;

记录任务运行状态下的总时间。 *TBD：使用的例子的链接*

>	struct	_reent xNewLib_reent;

提供对 newlib 的支持，FreeRTOS 本身不需要关心该变量。

>	volatile uint32_t ulNotifiedValue;
>	volatile uint8_t ucNotifyState;

提供任务通知机制。 *TBD：任务通知机制链接*

>	uint8_t	ucStaticallyAllocated;

标识 Stack 和 TCB 是静态定义的还是动态申请的，在 `vTaskDelete` 中根据该状态字进行相应的注销操作。

>	uint8_t ucDelayAborted;

标识该任务是否需要立即唤醒，可以使用 `xTaskAbortDelay` 停止延时等待。 *TBD：延时链接*

>	int iTaskErrno;

记录该任务的errno。

## 状态迁移





## 状态与链表的关系


  [1]: ./images/task.jpg
  [2]: ./images/stack_growth.jpg
  [3]: https://sourceforge.net/p/freertos/discussion/382005/thread/4b56fac4/