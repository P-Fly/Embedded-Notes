# List

## 背景

在 FreeRTOS 中，大量使用了 `List` 作为最基本的一种数据结构，比如：

 - **调度器** 使用链表跟踪任务的不同状态。
 - **消息队列** 使用链表跟踪消息的发送和接收。

因此，我们需要分析下 FreeRTOS 中的链表是如何组织的。

## 总览

![overview][1]

FreeRTOS 中的链表使用了 **List_t** 和 **ListItem_t** 这两组数据结构。其中：

 - **List_t** 用于管理该组链表，相当于一个链表头，包含了 **列表项的个数**（uxNumberOfItems），**迭代器**（pxIndex）等相关元素。
 - **ListItem_t** 用于记录链表的每个数据单元。

即 FreeRTOS 使用双向链表将每一个 **ListItem_t** 链接在一起，由 **List_t** 负责管理该组链表。

## 数据结构

### List_t

``` C
/*
 * Definition of the type of queue used by the scheduler.
 */
typedef struct xLIST
{
	...
	/* 记录链表项个数 */
	volatile UBaseType_t uxNumberOfItems;

	/* 遍历链表项使用的迭代器 */
	ListItem_t * configLIST_VOLATILE pxIndex;

	/* 链表尾指针，该链表项的值为该组链表的最大可能值 */
	MiniListItem_t xListEnd;
	...
} List_t;
```

### ListItem_t

``` C
/*
 * Definition of the only type of object that a list can contain.
 */
struct xLIST_ITEM
{
	...
	/* 该列表项的值，可以保存任务优先级、阻塞时间等信息，大部分链表会按照该值递减排序 */
	configLIST_VOLATILE TickType_t xItemValue;

	/* 双向指针 */
	struct xLIST_ITEM * configLIST_VOLATILE pxNext;
	struct xLIST_ITEM * configLIST_VOLATILE pxPrevious;

	/* 指向包含该列表项的对象，相当于指向了自己的派生类的实体 */
	void * pvOwner;

	/* 指向所属的 List_t */
	struct xLIST * configLIST_VOLATILE pxContainer;
	...
};
typedef struct xLIST_ITEM ListItem_t;
```

在 **List_t** 中，有一个 **MiniListItem_t** 的数据结构。其实它的作用与 **ListItem_t** 一样，只是为了节省内存。在链表头有部分数据不需要，因此新增了一个数据结构。

## 常用函数接口

### vListInitialise

初始化链表头 `List_t`。

### vListInitialiseItem

初始化链表项 `ListItem_t`。

### vListInsertEnd

在当前的迭代器 `pxIndex` 指向的位置前方插入 `ListItem_t`，插入位置如下图蓝色部分：

![vListInsertEnd][2]

该接口插入操作的链表一般是不需要排序，而是通过轮询进行访问的链表，其中 `pxIndex` 的作用就是对链表中的链表项进行遍历。例如 `pxReadyTasksLists[0]` 上挂载的是任务优先级相同的任务， `pxIndex` 负责遍历链表，新加入的任务按照 `pxIndex` 的位置挂载即可，而不需要根据新的链表项的值进行排序。

### vListInsert

以**降序**方式插入 `ListItem_t`，比如已有链表项 `xItemValue = 5` 和 `xItemValue = 3` ，新插入链表项 `xItemValue = 4`，此时插入位置如下图蓝色部分：

![vListInsert][3]

该接口会按照链表项的值以降序方式将 `ListItem_t` 插入进链表。例如 `pxDelayedTaskList` 上挂载的是需要一定延时的任务，因此新加入此链表的链表项需要按照延迟的时间长短进行排序。

### uxListRemove

删除链表项。

## 其它

我们可以使用 `configUSE_LIST_DATA_INTEGRITY_CHECK_BYTES` 宏，在数据结构中放置一些预设的数据，并在需要时通过检测这些数据来确定这部分内存是否被破坏。

  [1]: ./images/overview.jpg
  [2]: ./images/vListInsertEnd.jpg
  [3]: ./images/vListInsert.jpg
