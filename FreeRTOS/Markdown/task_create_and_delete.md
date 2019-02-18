# 任务的创建与删除

## 前言

FreeRTOS 的基本组成单位是任务，因此任务的管理尤为重要。本章重点描述任务的创建和删除接口。

对于 FreeRTOS ，创建和删除任务的 API 如下：

- **xTaskCreate**：创建任务的基本接口。
- **vTaskDelete**：删除任务的基本接口。
- **xTaskCreateStatic**：`TCB_t` 和 stack 由用户直接提供，不从 OS heap 中分配。
- **xTaskCreateRestricted**：输入参数中增加了对 **MPU** 的配置。
- **xTaskCreateRestrictedStatic**：静态版的 `xTaskCreateRestricted` 。

其中，`xTaskCreate` 和 `vTaskDelete` 是最基本的 API，其它3个API只是对基本API做了一些参数上的扩展。因此我们需要重点分析 `xTaskCreate` 和 `vTaskDelete` 的实现。

## xTaskCreate

### 原型

``` C
BaseType_t xTaskCreate( TaskFunction_t pxTaskCode, 
		const char * const pcName,
		const configSTACK_DEPTH_TYPE usStackDepth,
		void * const pvParameters,
		UBaseType_t uxPriority,
		TaskHandle_t * const pxCreatedTask )
```

### 参数

 - **pxTaskCode**：提供函数指针，指向任务入口。
 - **pcName**：指定任务的名称。
 - **usStackDepth**：指定栈的深度，实际使用的字节空间为 **usStackDepth * sizeof(StackType_t)**。
 - **pvParameters**：提供任务参数。
 - **uxPriority**：提供任务优先级。
 - **pxCreatedTask**：返回任务句柄，供其它接口使用。

### 流程

![xTaskCreate Sequence][1]

 1. 检查该任务是否需要运行在特权级

    可以通过将输入参数 `uxPriority` 高位置位的方式通知 FreeRTOS，新创建的该任务应该运行在特权级。此时 FreeRTOS 通过使用与普通任务不同的 **RAM** 段的方式来对这一任务提供特殊的保护，特权位由宏 `portPRIVILEGE_BIT` 定义。

 2. 计算栈顶地址

    对于 **Cortex-M3**，需要按照 **8Byte** 对齐，由 `portBYTE_ALIGNMENT_MASK` 定义。

 3. 初始化栈空间

	`pxPortInitialiseStack` 负责初始化任务堆栈，并将最新的栈顶指针赋值给任务 `TCB_t` 的 `pxTopOfStack`。	

	调用该函数后，相当于执行了一次 **ISR** 中断。虽然此时任务并没有开始执行，但我们模拟了一次寄存器入栈的流程。之后在我们调用 `vTaskStartScheduler` 启动 FreeRTOS 时，会通过 **SVC** 指令产生一次 **SVC exception**，最终我们会在 **SVC handler** 中通过出栈操作使CPU进入任务的上下文环境。

    对于 **Cortex-M3**，首先模拟硬件压栈，然后调整堆栈指针模拟 **R11 - R4** 寄存器压栈。

    详情可以参考 *ARM®v7-M ArchitectureReference Manual*：

    ![][2]

	对于 **Cortex-M4**，这一部分实现上有一点差异，在硬件压栈后会额外压入值 `portINITIAL_EXC_RETURN`。在 **SVC handler** 中会将该值赋予 **R14** 寄存器，并调用 **bx r14** 指令退出 **SVC handler**。

	`portINITIAL_EXC_RETURN` 的定义可以参考 *ARM®v7-M ArchitectureReference Manual*：

    ![][3]

	相关代码如下：

``` C
StackType_t *pxPortInitialiseStack( StackType_t *pxTopOfStack, TaskFunction_t pxCode, void *pvParameters )
{
	...
	/* A save method is being used that requires each task to maintain its
	own exec return value. */
	pxTopOfStack--;
	*pxTopOfStack = portINITIAL_EXC_RETURN;
	...
}

__asm void vPortSVCHandler( void )
{
	...
	/* Pop the core registers. */
	ldmia r0!, {r4-r11, r14}
	...
	bx r14
}
```

 4. 将新创建的 `TCB_t` 挂载到 `pxReadyTasksLists`
 
	`prvAddNewTaskToReadyList` 主要负责将新创建的任务挂载到等待链表中。同时它还会根据 FreeRTOS 的运行环境（FreeRTOS 是否启动、当前任务的优先级）和新建任务优先级的对比结果，进行适当的任务切换（`taskYIELD_IF_USING_PREEMPTION`），以保证当前运行的任务一定是优先级最高的任务。

## vTaskDelete

### 原型

``` C
void vTaskDelete( TaskHandle_t xTaskToDelete )
```

### 参数

 - xTaskToDelete：待删除任务的句柄。

### 流程

![vTaskDelete Sequence][4]

 [1]: ./images/xTaskCreate.jpg
 [2]: ./images/exception_entry_behavior.jpg
 [3]: ./images/exc_return_definition.jpg
 [4]: ./images/vTaskDelete.jpg
