# 调度器的启动

## 前言

调度器是 FreeRTOS 操作系统的核心，其主要负责任务切换，本章重点描述调度器的启动流程。

## 流程

![vTaskStartScheduler][1]

 1. 创建 IDLE 和 Tmr Svc 任务

这两个任务是系统提供的默认服务。

 2. 关闭中断

此处关闭中断是为了在调度器启动的过程中不会被中断打断，在调度器启动成功后 FreeRTOS 会自动打开中断。

对于 **Cortex-M**，开关中断的代码实现如下：

``` C
static portFORCE_INLINE void vPortRaiseBASEPRI( void )
{
uint32_t ulNewBASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY;

	__asm
	{
		/* Set BASEPRI to the max syscall priority to effect a critical
		section. */
		msr basepri, ulNewBASEPRI
		dsb
		isb
	}
}

static portFORCE_INLINE void vPortSetBASEPRI( uint32_t ulBASEPRI )
{
	__asm
	{
		/* Barrier instructions are not used as this function is only used to
		lower the BASEPRI value. */
		msr basepri, ulBASEPRI
	}
}
```

通过使用设置 **BASEPRI** 寄存器的方式来开关中断，该寄存器属于特殊寄存器，使用 **msr / mrs** 指令进行操作。该寄存器的用途如下：

 - 当优先级值大于或等于 **BASEPRI** 的异常会被屏蔽。
 - 当 **BASEPRI = 0** 时，不屏蔽任何异常。

详情可以参考 *ARM®v7-M ArchitectureReference Manual*：

![basepri][2]

 3. 校验中断优先级

FreeRTOS 的中断管理如下：

![interrupt priority][3]

系统只关心 **configLIBRARY_LOWEST_INTERRUPT_PRIORITY** ~ **configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY** 之间的中断。高于的部分由用户自己去管理，在这部分中断中，用户不应该调用 FreeRTOS 的系统调用。

 4. 配置 PendSV 和 SysTick 的中断优先级

``` C
	portNVIC_SYSPRI2_REG |= portNVIC_PENDSV_PRI;
	portNVIC_SYSPRI2_REG |= portNVIC_SYSTICK_PRI;
```

 详情可以参考 *ARM®v7-M ArchitectureReference Manual*：

![interrupt priority][4]

 5. 配置 Systick

```C
	void vPortSetupTimerInterrupt( void )
	{
		/* Calculate the constants required to configure the tick interrupt. */
		#if configUSE_TICKLESS_IDLE == 1
		{
			ulTimerCountsForOneTick = ( configSYSTICK_CLOCK_HZ / configTICK_RATE_HZ );
			xMaximumPossibleSuppressedTicks = portMAX_24_BIT_NUMBER / ulTimerCountsForOneTick;
			ulStoppedTimerCompensation = portMISSED_COUNTS_FACTOR / ( configCPU_CLOCK_HZ / configSYSTICK_CLOCK_HZ );
		}
		#endif /* configUSE_TICKLESS_IDLE */

		/* Configure SysTick to interrupt at the requested rate. */
		portNVIC_SYSTICK_LOAD_REG = ( configSYSTICK_CLOCK_HZ / configTICK_RATE_HZ ) - 1UL;
		portNVIC_SYSTICK_CTRL_REG = ( portNVIC_SYSTICK_CLK_BIT | portNVIC_SYSTICK_INT_BIT | portNVIC_SYSTICK_ENABLE_BIT );
	}
```

 6. prvStartFirstTask

```armasm
	ldr r0, =0xE000ED08
	ldr r0, [r0]
	ldr r0, [r0]
	msr msp, r0
```

根据 **VTOR** 寄存器获取向量表的地址，并将第一组 32Bit 数据赋予 **msp** 寄存器。

详情可以参考 *ARM®v7-M ArchitectureReference Manual*：

![interrupt priority][5]

 - 根据 **Cortex-M** 的定义，向量表的第一个 32Bit 数据为系统栈上电时的默认地址。（相当于我们从代码启动到运行到当前位置时所产生的栈都不需要了）
 -  这段栈的大小为 Startup.S 文件中定义的 **Stack_Size**。
 - 根据后面任务切换的分析，我们可以发现该系统栈存储 **interrupt handler** 中的栈数据，使用 **msp** 进行访问。任务的堆栈由各个任务的 TCB_t 去追踪，使用 **psp** 进行访问。

```armasm
	cpsie i
	cpsie f
	dsb
	isb
```

使能全局中断并清理缓存。

```armasm
	svc 0
```

手动调用 **SVC exception**。

 7. vPortSVCHandler

```armasm
	ldr	r3, =pxCurrentTCB
	ldr r1, [r3]
	ldr r0, [r1]
	ldmia r0!, {r4-r11, r14}
```

将 **r4 - r11** 手动出栈，**r14** 寄存器恢复的数据在入栈时被设置为了 0xFFFFFFFD。

```armasm
	msr psp, r0
	isb
```

当前是在 **handler** 模式，使用的是 **msp**，需要在退出 **handler** 模式前手动调整 **psp**。

```armasm
	mov r0, #0
	msr	basepri, r0
```

关闭优先级屏蔽。

```armasm
	bx r14
```

从 SVC exception 返回，**r14** 的 0xFFFFFFFD 表示返回 **Thread** 模式，并且使用 **PSP**。

  [1]: ./images/vTaskStartScheduler.jpg
  [2]: ./images/basepri.jpg
  [3]: ./images/interrupt_priority.jpg
  [4]: ./images/pendsv_and_systick_priority_register.jpg
  [5]: ./images/vtor_register.jpg
