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

    通过设置 **BASEPRI** 寄存器的方式来开关中断，该寄存器属于特殊寄存器，使用 **msr / mrs** 指令进行操作。该寄存器的用途如下：

    - 当它被设成某个值后，所有优先级号大于等于此值的中断都被关（优先级号越大，优先级越低）。
    - 当 **BASEPRI = 0** 时，不屏蔽任何 **exception**。

    详情可以参考 *ARM®v7-M ArchitectureReference Manual*：

    ![basepri][2]

    关于硬件优先级和优先级号的关系可以参看下图：

    ![priority value][8]

    后文中所有的优先级都是指硬件优先级。

 3. 校验中断优先级

    FreeRTOS 的中断管理如下：

    ![interrupt priority][3]

    - 中断优先级值越小，优先级越高。
    - 小于值 **configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY** 的中断由用户自行管理，不应该调用 FreeRTOS 的系统函数。
    - FreeRTOS 只关心 **configLIBRARY_LOWEST_INTERRUPT_PRIORITY** 和 **configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY** 之间的中断，这部分中断可以调用中断安全函数(以 FromISR 后缀的系统函数）。
    - **SVC** 的中断优先级为 **configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY - 1**，它比所有 FreeRTOS 管理的中断的优先级都要高。
    - **PendSV** 和 **SysTick** 的优先级最低。

    对于 FreeRTOS 中断的更多详情和一些讨论的链接可以参看文档：[中断优先级][4]。

 4. 配置 **PendSV** 和 **SysTick** 的中断优先级

    ``` C
    portNVIC_SYSPRI2_REG |= portNVIC_PENDSV_PRI;
    portNVIC_SYSPRI2_REG |= portNVIC_SYSTICK_PRI;
    ```

    详情可以参考 *ARM®v7-M ArchitectureReference Manual*：

    ![interrupt priority][5]

 5. 配置 **Systick**

    初始化 **SysTick**，包括配置基本时钟值，使能 **SysTick exception**。

 6. 执行 **prvStartFirstTask**

    ```armasm
    PRESERVE8
    ````

    声明当前函数的栈需要 8Byte 对齐。
    > The PRESERVE8 directive specifies that the current file preserves eight-byte alignment of the stack.

    ```armasm
    ldr r0, =0xE000ED08
    ldr r0, [r0]
    ldr r0, [r0]
    msr msp, r0
    ```

    根据 **VTOR** 寄存器获取向量表的地址，并将第一组 32Bit 数据赋予 **msp** 寄存器。

    详情可以参考 *ARM®v7-M ArchitectureReference Manual*：

    ![vtor register][6]

    - 根据 **Cortex-M** 的定义，向量表的第一个 32Bit 数据为系统上电时的默认栈地址。这组操作相当于我们从代码启动到当前位置时，所产生的栈都不需要了。
    - 这段栈的大小为 Startup.S 文件中定义的 **Stack_Size**。
    - 根据后面任务切换的分析，我们可以发现：**Stack_Size**定义的堆栈用于存储 **interrupt handler** 中的数据，使用 **msp** 进行访问；任务栈由各个任务的 TCB_t 去追踪，使用 **psp** 进行访问。

    ```armasm
    cpsie i
    cpsie f
    dsb
    isb
    ```

    - 使能全局中断，并清理缓存。
    - 虽然全局中断已经打开，但 **basepri** 寄存器依然被设为 **configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY**，此时只有 **用户自管理的中断** 和 **SVC中断** 可以被执行。（SVC中断优先级为 **configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY - 1**）

    ```armasm
    svc 0
    ```

    手动产生 **SVC exception**。

 7. 执行 **vPortSVCHandler**

    - 产生中断前使用的是 **msp**，因此硬件自动将 **8** 个寄存器压入到系统栈。
    - 进入 **SVC exception**，此时默认使用的是 **msp**。

    ```armasm
    ldr r3, =pxCurrentTCB
    ldr r1, [r3]
    ldr r0, [r1]
    ldmia r0!, {r4-r11, r14}
    ```

    - 通过 **pxCurrentTCB** 找到下一个需要执行的任务的栈。
    - 将 **r4 - r11** 手动出栈 （其它寄存器由硬件自动退栈）。
    - **r14** 寄存器恢复的数据在入栈时被设置为了 0xFFFFFFFD： 表示退出 **SVC exception** 时自动切换为 **Thread mode**，栈指针也切换为 **psp** （硬件出栈也使用该栈寄存器）。

    详情可以参考 *ARM®v7-M ArchitectureReference Manual*：

    ![exception return behavior][7]

    ```armasm
    msr psp, r0
    isb
    ```

    在退出前手动调整**psp**，设为 **pxCurrentTCB->pxTopOfStack**。

    ```armasm
    mov r0, #0
    msr basepri, r0
    ```

    取消优先级屏蔽，此时所有中断均可以被响应。

    ```armasm
    bx r14
    ```

    从 **SVC exception** 返回，进入 **Thread mode**。硬件使用 **psp** 恢复其它寄存器。后续的代码也继续使用 **psp** 寄存器。

 [1]: ./images/vTaskStartScheduler.jpg
 [2]: ./images/basepri.jpg
 [3]: ./images/interrupt_priority.jpg
 [4]: misc_interrupt_priority.md
 [5]: ./images/pendsv_and_systick_priority_register.jpg
 [6]: ./images/vtor_register.jpg
 [7]: ./images/exception_return_behavior.jpg
 [8]: ./images/priority_value.jpg
