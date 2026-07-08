# 1 任务状态

![[Pasted image 20260701102237.png|320]]

| 状态  | 含义               | 什么时候进入                    |
| --- | ---------------- | ------------------------- |
| 就绪态 | 已准备好运行，等CPU分配    | 创建后默认；阻塞结束后回这里            |
| 运行态 | 正被CPU执行（单核只有一个）  | 调度器选中                     |
| 阻塞态 | 等待明确条件（时间到/资源可用） | `vTaskDelay()`、等信号量/队列没等到 |
| 挂起态 | 被强制冻结，无自动唤醒条件    | `vTaskSuspend()`          |

## 1.1 阻塞态 vs 挂起态——核心区别在"谁来唤醒"

判断标准：唤醒机制：
- **阻塞态**：任务自己设定了明确的唤醒条件（时间到/资源到），条件满足后**系统自动**唤醒，不需要外部插手。类比"定闹钟睡觉"。

- **挂起态**：没有任何自动唤醒条件，必须靠**外部代码**主动调用`vTaskResume()`才能恢复，不管过多久、外部资源怎么变，自己不会醒。类比"被打了麻药，必须有人打解药"。


---

# 2 任务创建

## 2.1 xTaskCreate（动态）

```c
BaseType_t xTaskCreate(
    TaskFunction_t pxTaskCode,          // 任务函数（函数指针）
    const char * const pcName,          // 任务名字（调试用）
    uint16_t usStackDepth,              // 栈大小（单位是"字"）
    void * const pvParameters,          // 传给任务函数的参数
    UBaseType_t uxPriority,             // 优先级
    TaskHandle_t * const pxCreatedTask  // 任务句柄存这里
);
```

TCB由内部`pvPortMalloc`自动从`ucHeap`分配，不需要传。

## 2.2 xTaskCreateStatic（静态）

```c
TaskHandle_t xTaskCreateStatic(
    TaskFunction_t pxTaskCode,
    const char * const pcName,
    uint32_t ulStackDepth,
    void * const pvParameters,
    UBaseType_t uxPriority,
    StackType_t * const puxStackBuffer,   // 栈内存
    StaticTask_t * const pxTaskBuffer     // TCB内存
);
```

比动态多两个参数：栈内存地址、TCB内存地址——因为静态创建不经过`ucHeap`，需要开发者提供固定内存。

> [!warning] 在32位MCU平台，1字 = 32位 = 4字节，在FreeRTOS里面创建任务用字

---
# 3 调度机制
## 3.1 抢占式调度+时间片轮转

**不同优先级 → 抢占式**：高优先级任务只要就绪，立刻打断正在跑的低优先级任务。

**相同优先级 → 时间片轮转**：每个任务跑1tick周期，然后切换任务。

> [!warning] 时间片不是写死的"1ms" 时间片长度 = 1个tick周期，由 `configTICK_RATE_HZ` 决定（常见默认1000Hz对应1ms，但可配置成任意值）。

## 3.2 抢占式 vs 协作式——两个不同维度，不是互斥选项

**协作式（Cooperative Scheduling）**：任务一旦开始运行，会一直占着CPU，直到它自己主动调用`taskYIELD()`让出，或自己进入阻塞（等信号量等），调度器才有机会切换。**即使有更高优先级的任务已就绪，只要当前任务不主动让，调度器也无法打断它。**

|         | 抢占式                | 协作式                     |
| ------- | ------------------ | ----------------------- |
| 能否强制打断  | 能，高优先级随时抢占         | 不能，必须任务自己让出             |
| 实时性     | 好                  | 差（高优先级可能被迫等很久）          |
| 风险      | 无（调度器有绝对控制权）       | 某个任务不自觉（忘了让出）会拖死全局      |
| 临界区保护   | 需要考虑随时被打断          | 更简单（反正不会被抢占）            |
| 上下文切换开销 | 较大                 | 较小                      |
| 适用场景    | 实时系统（FreeRTOS默认定位） | 任务少、都写得规范、实时性要求不苛刻的简单系统 |

---

# 4 TCB（任务控制块）

TCB 全称是 **Task Control Block**，即任务控制块。

在 FreeRTOS 中，每个任务都会对应一个 TCB。任务函数只是任务的执行代码，而 TCB 用来保存任务的管理信息。

可以理解为：

```
任务函数 = 任务要执行什么代码
TCB = FreeRTOS 如何管理这个任务
```
## 4.1 TCB 核心源码结构

FreeRTOS 中 TCB 的核心结构可以简化为：

```c
typedef struct tskTaskControlBlock
{
    volatile StackType_t *pxTopOfStack;      // 当前任务栈顶指针
    ListItem_t xStateListItem;              // 挂入任务状态链表：就绪/挂起等
    ListItem_t xEventListItem;              // 挂入事件等待链表：如等待队列/信号量
    UBaseType_t uxPriority;                 // 当前生效优先级
    StackType_t *pxStack;                   // 任务栈起始地址
    char pcTaskName[configMAX_TASK_NAME_LEN]; // 任务名，主要用于调试
#if ( configUSE_MUTEXES == 1 )
    UBaseType_t uxBasePriority;             // 原始优先级，优先级继承后用于恢复
    UBaseType_t uxMutexesHeld;              // 当前任务持有的互斥量数量
#endif
#if ( configUSE_TASK_NOTIFICATIONS == 1 )
    volatile uint32_t ulNotifiedValue[configTASK_NOTIFICATION_ARRAY_ENTRIES]; // 任务通知值，直接内嵌在 TCB 中
    volatile uint8_t ucNotifyState[configTASK_NOTIFICATION_ARRAY_ENTRIES];    // 通知状态：未等待/正在等待/已收到
#endif
} TCB_t;
```

任务切换时，最核心的字段是任务栈顶指针：

```c
volatile StackType_t *pxTopOfStack;
```

---
# 5 队列

## 5.1 队列的存储方式：值拷贝，不是存地址

```c
BaseType_t xQueueSend(QueueHandle_t xQueue, const void *pvItemToQueue, TickType_t xTicksToWait);
```

> [!important] `pvItemToQueue`只是告诉函数去哪读数据，不是把地址存进队列 函数内部会把这个地址指向的数据**内容**完整拷贝一份，存进队列自己的缓冲区——之后即使原始变量被销毁或改变，队列里的拷贝不受影响。这是为了避免"发送方局部变量生命周期结束后，队列里存的地址变成悬空指针"这类问题。

如果数据本身很大（比如几百字节的结构体），常见优化是发送"指向数据的指针"而不是数据本身——但此时被拷贝的"值"变成了这个指针（4字节），数据本身的生命周期需要发送/接收双方自己协调（堆分配、接收方用完释放），队列机制本身不负责保证这块内存的有效性。

## 5.2 队列内部结构：环形缓冲区

```c
typedef struct QueueDefinition
{
    int8_t *pcHead;          // 队列存储区起始地址
    int8_t *pcWriteTo;       // 下一个要写入的位置
    union
    {
        struct
        {
            int8_t *pcTail;      // 队列存储区结束地址
            int8_t *pcReadFrom;  // 上一次读取的位置
        } xQueue;//如果是队列，环形缓冲区
        struct
        {
            TaskHandle_t xMutexHolder;
            UBaseType_t uxRecursiveCallCount;
        } xSemaphore;//信号量/互斥量，记录互斥量携带者，递归互斥量数
    } u;
    List_t xTasksWaitingToSend;     // 等待写入/send/give 的任务链表
    List_t xTasksWaitingToReceive;  // 等待读取/receive/take 的任务链表
    volatile UBaseType_t uxMessagesWaiting; // 当前队列中有效 item/token 个数
    UBaseType_t uxLength;                   // 队列容量
    UBaseType_t uxItemSize;                 // 每个 item 大小
    volatile int8_t cRxLock;
    volatile int8_t cTxLock;
} Queue_t;
```

> [!important] 为什么需要`uxMessagesWaiting`这个独立计数器 队列写满后绕一圈，`pcWriteTo`和`pcReadFrom`会重新指向同一个位置——这和"队列刚创建、空的"这个状态，两个指针的值完全一样，无法区分。内存本身不会自动标记"有效/无效"（读走的数据残留在原地，不会被清零）。所以必须单独维护`uxMessagesWaiting`：每次写入成功+1，读取成功-1，判空看是否为0，判满看是否等于`uxLength`，不依赖比较两个指针的位置。

---
# 6 信号量：

|                       | 二值信号量  | 计数信号量         |
| --------------------- | ------ | ------------- |
| 队列长度                  | 1      | max_count     |
| `uxMessagesWaiting`范围 | 只能是0或1 | 0 ~ max_count |
## 6.1 二值信号量

队列长度为1，队列内元素大小为0的特殊队列

```c
xSemaphoreCreateBinary()
   ↓ 内部调用
xQueueGenericCreate(),其中uxLength=1，uxItemSize=0（信号量不携带任何数据内容）
```

> 信号量要表达的信息只有"有/无"，完全由`uxMessagesWaiting`这一个计数器承载，不需要额外的数据存储空间。`xSemaphoreGive`本质是`xQueueSend(sem, NULL, 0, ...)`（不拷贝数据，只让计数器+1）；`xSemaphoreTake`本质是`xQueueReceive`（计数器-1）。

## 6.2 计数信号量

队列长度为max_count，队列内元素大小为0的特殊队列

---

# 7 互斥量

底层队列参数和二值信号量完全一样（`uxLength=1`，`uxItemSize=0`），区别在于`Queue_t`结构体里被复用出两处额外信息：

```c
// 1. pcHead 被宏定义复用成 uxQueueType（同一个字段）
	#define uxQueueType pcHead
	#define queueQUEUE_IS_MUTEX NULL
// 作为mutex时，pcHead被设成NULL——仅用来标记"这是mutex"，和持有者无关
// 2. union 复用，同一块内存按用途解读成不同结构体
typedef struct SemaphoreData {
    TaskHandle_t xMutexHolder;        // 真正记录"当前持有者"的字段
    UBaseType_t uxRecursiveCallCount;
} SemaphoreData_t;

typedef struct QueueDefinition {
    ...
    union {
        QueuePointers_t xQueue;       // 普通队列：解读成队列指针信息
        SemaphoreData_t xSemaphore;   // 互斥量：解读成持有者信息
    } u;
    ...
} Queue_t;
```

## 7.1 底层复用:mutex 本质是特殊配置的队列

互斥量的队列参数和二值信号量完全一样(`uxLength=1`, `uxItemSize=0`),区别在于 `Queue_t` 里两处字段被"重新解读":

**① `pcHead` 被宏复用成类型标记**

```c
#define uxQueueType             pcHead
#define queueQUEUE_IS_MUTEX     NULL
```

创建互斥量时 `pcHead` 被设为 `NULL`,仅用作"这是 mutex,不是普通队列"的类型标记,和"谁持有锁"无关——**别把这个字段和持有者信息搞混**,持有者信息在下面的 union 里。

**② union 里的 `xSemaphore` 才是真正记录持有者的地方**

```c
typedef struct SemaphoreData
{
    TaskHandle_t xMutexHolder;         // 当前持有者的任务句柄
    UBaseType_t  uxRecursiveCallCount; // 递归互斥量的嵌套计数
} SemaphoreData_t;

typedef struct QueueDefinition
{
    ...
    union
    {
        QueuePointers_t xQueue;      // 普通队列:读写指针
        SemaphoreData_t xSemaphore;  // 互斥量:持有者信息
    } u;
    ...
} Queue_t;
```

普通队列和互斥量**互斥使用**同一块内存,靠 `uxQueueType` 决定按哪种结构体解读。

## 7.2 优先级继承完整触发链路

### 7.2.1 场景

- 低优先级任务 **L** 持有 mutex
- 高优先级任务 **H** 尝试获取,失败阻塞
- 需要把 L 的优先级临时提到 H 的水平,防止中优先级任务插队
### 7.2.2 第一步:H 获取失败 → `xQueueSemaphoreTake`

```c
if ( uxQueueType == queueQUEUE_IS_MUTEX && uxMessagesWaiting == 0 )
{
    pxTCBOwner = pxQueue->u.xSemaphore.xMutexHolder;  // 从union里取出持有者
    xTaskPriorityInherit( pxTCBOwner );                // 触发提升
}
// H 挂入 xTasksWaitingToSend,进入阻塞
```
### 7.2.3 第二步:`xTaskPriorityInherit` 执行提升

```c
if ( pxTCB->uxPriority < pxCurrentTCB->uxPriority )   // L 确实比 H 低,才需要提升
{
    if ( pxTCB->uxBasePriority == 0 )                  // 尚未提升过,先保存原值
    {
        pxTCB->uxBasePriority = pxTCB->uxPriority;
    }
    pxTCB->uxMutexesHeld++;
    pxTCB->uxPriority = pxCurrentTCB->uxPriority;       // 直接改L的优先级字段
}
```
### 7.2.4 第三步:L 释放锁 → `xQueueGenericSend`

```c
if ( uxQueueType == queueQUEUE_IS_MUTEX )
{
    xTaskPriorityDisinherit( pxQueue->u.xSemaphore.xMutexHolder ); // 先用旧持有者句柄触发降级判断
}
pxQueue->u.xSemaphore.xMutexHolder = NULL;  // 再清空持有者字段
```
### 7.2.5 第四步:`xTaskPriorityDisinherit` 判断是否真正降级

```c
pxTCB->uxMutexesHeld--;
if ( pxTCB->uxMutexesHeld == 0 && pxTCB->uxPriority != pxTCB->uxBasePriority )
{
    pxTCB->uxPriority = pxTCB->uxBasePriority;   // 恢复原始优先级
    pxTCB->uxBasePriority = 0;
    // 从当前(被提升的)就绪列表摘出,重新插入原优先级列表
}
```

---
## 7.3 递归互斥量(Recursive Mutex)

普通互斥量 `xSemaphoreTake` **不允许同一任务重入**——同一个任务对同一把锁 take 两次会自己把自己卡死。

典型触发场景:**驱动层函数自带加锁**,而**业务层又想把多次驱动调用包成一个原子操作**,导致同一个任务对同一把锁嵌套 take。

- 普通互斥量的语义是:**"锁是否空闲"决定能不能拿**,不关心请求者是谁。
- 递归互斥量的语义是:**"锁是否空闲,或者持有者是不是我自己"决定能不能拿**——多了一层"身份识别"。

## 7.4 自旋锁(Spinlock)vs 互斥量

| 互斥量|自旋锁|
|---|---|---|
|拿不到锁时|阻塞,让出CPU给别的任务|原地忙等(busy-wait),不让出CPU|
|依赖调度器|是|否|
|适用场景|单核任务间,临界区较长/时间不确定|多核(SMP)之间,临界区极短|
|单核FreeRTOS|常用|一般不需要|

单核 FreeRTOS 靠 `taskENTER_CRITICAL()` 关中断就能实现互斥,不需要自旋锁——同一时刻只有一条指令流,关中断就没人能抢。**自旋锁只在SMP多核场景下才必要**(如 FreeRTOS SMP、ESP-IDF 双核):核A关自己的中断拦不住核B并发访问共享数据,必须靠一个多核可见的原子标志位"转圈"等待。

### 7.4.1 为什么原地等待,而不是阻塞让出CPU

- **临界区极短**:自旋锁保护的操作通常只有几条指令(改个计数器、改个指针)。若改用阻塞式(互斥量),一次Take/Give要经历"任务挂起→触发调度→上下文切换→恢复"的完整流程,这个切换开销可能比临界区本身还长,反而更慢。
- **多核场景下"阻塞"没有意义或不安全**:自旋锁常用来保护调度器自身的核心数据结构(如就绪列表)。如果在这种地方用互斥量去阻塞等待,等于让调度器去调度"调度器自己需要用的资源",容易产生逻辑死锁;而忙等不涉及调度器,不会有这个问题。
- **代价换算**:忙等是纯粹浪费CPU(空转不产生有效工作),但只要临界区足够短,这段浪费远小于一次上下文切换的开销——这也是为什么自旋锁的铁律是"临界区必须尽可能短",长临界区绝不能用自旋锁。

---
# 8 任务通知(Task Notification)

每个任务的 TCB(任务控制块)里自带了一块"通知专用"存储,不需要额外创建队列/信号量对象就能实现任务间的轻量级同步或传值。

```c
typedef struct tskTaskControlBlock
{
    ...
#if ( configUSE_TASK_NOTIFICATIONS == 1 )
    volatile uint32_t ulNotifiedValue[ configTASK_NOTIFICATION_ARRAY_ENTRIES ];
    volatile uint8_t  ucNotifyState[ configTASK_NOTIFICATION_ARRAY_ENTRIES ];
#endif
    ...
} TCB_t;
```

- `ulNotifiedValue`:一个(或多个,新版支持多组通知)32位整数,存"通知的内容"
- `ucNotifyState`:三态标记,`taskNOT_WAITING_NOTIFICATION` / `taskWAITING_NOTIFICATION` / `taskNOTIFICATION_RECEIVED

通知是"点对点"的,只能唤醒一个确定的任务,不能像队列那样被多个任务共同等待/竞争。

---

# 9 SVC —— 从裸机到第一个任务的唯一一次切换

`vTaskStartScheduler()`执行到最后，会执行一条`SVC`指令，触发一次异常，跳到`vPortSVCHandler`，完成"从main()裸机执行流，切换到第一个任务开始用PSP运行"这个只发生一次的关键动作。

## 9.1 为什么要走"异常"这条路，不是普通函数调用

启动任务不是普通函数调用，而是要恢复一个任务上下文。FreeRTOS 在创建任务时，已经在任务栈里构造好了初始现场，包括任务入口地址、参数、xPSR、LR 等信息。

触发 SVC 后，CPU 进入异常处理流程，FreeRTOS 在 SVC_Handler 中取出第一个任务的栈顶，并通过“异常返回”机制，让 CPU 自动切换到 Thread mode + PSP，开始执行第一个任务。

所以，SVC 的核心作用不是“提权”，而是借助 Cortex-M 的异常返回机制，规范地启动第一个任务。
## 9.2 触发时机：vTaskStartScheduler() 的最后一步

```c
int main(void) {
    xTaskCreate(...);            // 创建任务，TCB和栈已在ucHeap里备好
    xTaskCreate(...);
    vTaskStartScheduler();       // 启动调度器，这一去不复返
    for(;;);                     // 正常情况下永远不会执行到这里
}
```


```c
BaseType_t xPortStartScheduler(void) {
    // 配置SysTick按configTICK_RATE_HZ跳动
    // 配置PendSV和SysTick为最低优先级
    __asm volatile ("svc 0");    // 触发SVC异常，跳到vPortSVCHandler
    // 理论上不会执行到这里——SVC_Handler处理完直接交给第一个任务，不会"返回"
}
```

## 9.3 vPortSVCHandler 内部（简化版汇编）

```asm
ldr r3, =pxCurrentTCB    ; r3 = &pxCurrentTCB
ldr r1, [r3]              ; r1 = pxCurrentTCB（已指向第一个任务的TCB）
ldr r0, [r1]              ; r0 = TCB->pxTopOfStack
                           ; （第一个任务创建时预填的"假装暂停过"的现场）
ldmia r0!, {r4-r11}       ; 从任务栈恢复R4-R11
msr psp, r0                ; 把恢复后的栈顶写入PSP
mov r0, #0
msr basepri, r0            ; 清零BASEPRI，确保启动后中断不被意外屏蔽
orr r14, r14, #0xd        ; 构造正确的EXC_RETURN（bit2=1用PSP，bit0=1 Thumb状态）
bx r14                     ; 异常返回，硬件自动恢复剩余R0-R3,R12,LR,PC,xPSR
                           ; PC恢复成第一个任务入口，CPU开始运行第一个任务
```


---

# 10 PendSV（可挂起的系统调用）

## 10.1 PendSV 是什么？

PendSV 全称 **Pendable Service Call**（可挂起的系统调用），是 Cortex-M 内核提供的一个**系统异常**（不是外设中断），编号是 14。

- **可挂起（Pendable）**：可以通过软件设置"挂起位"来触发，不像外设中断需要硬件信号
- **优先级可配置**：可以设成任意优先级（FreeRTOS 设成最低）
- **专门用于上下文切换**：设计初衷就是为了操作系统做任务切换

## 10.2 为什么不在 SysTick 里直接切换？

SysTick 是系统时基中断，主要职责是：

- 更新 tick 计数
- 检查延时任务是否到期
- 判断是否需要调度

它不适合直接做完整上下文切换，原因有两个：

1. **SysTick 要短**：如果在里面保存寄存器、改 TCB、选任务、恢复寄存器，会拉长中断执行时间。
2. **切换来源不止 SysTick**：`taskYIELD()`、信号量/队列操作、ISR 唤醒高优先级任务，都可能触发调度。

所以 FreeRTOS 把逻辑拆开：

```
SysTick_Handler()
    → 更新 tick
    → 判断是否需要切换
    → 挂起 PendSV

PendSV_Handler()
    → 保存当前任务现场
    → 调用 vTaskSwitchContext()
    → 恢复新任务现场
```

核心记忆：

> **SysTick 负责判断“该不该切”，PendSV 负责真正“怎么切”。**


## 10.3 PendSV 为什么是最低优先级？

任务切换本身不如外设中断紧急。FreeRTOS 会把 PendSV 配置为**最低优先级**，让高优先级中断先处理，等系统没有更紧急的异常时，再执行任务切换。

FreeRTOS Cortex-M 常见端口中，`SysTick` 和 `PendSV` 通常都会被设置为内核最低优先级，但职责不同：

- **SysTick**：管 tick 和调度请求
- **PendSV**：管任务现场切换

如果 PendSV 优先级设得太高，任务切换可能压住外设中断，影响实时性。

## 10.4 PendSV 切换流程

以任务 A 切到任务 B 为例。

### 10.4.1 关键概念

- **TCB**：任务控制块，每个任务一个，保存任务管理信息
- **pxCurrentTCB**：全局指针，指向当前正在运行任务的 TCB
- **pxTopOfStack**：TCB 里的字段，保存该任务当前栈顶地址
- **PSP**：任务运行时使用的栈指针，切换任务本质就是切换 PSP
- **MSP**：中断/异常 Handler 自己运行时使用的栈指针

> [!important] PSP vs MSP
> 任务 A 在线程模式下运行，使用 PSP。PendSV 异常入口时，硬件自动保存的 r0-r3、r12、lr、pc、xPSR 会压入任务 A 的 PSP 栈；进入 PendSV_Handler 后，Handler 自己使用 MSP 运行。任务现场保存在 PSP 栈里，不会额外压到 MSP。

### 10.4.2 完整流程

```
1. SysTick / taskYIELD / ISR 请求调度
   → 设置 PendSV 挂起位，只是”请求切换”，还没有真正切换

2. CPU 进入 PendSV_Handler
   → 异常之前，硬件把 r0-r3、r12、lr、pc、xPSR 压入任务 A 的 PSP 栈
   → PendSV_Handler 本身使用 MSP 运行

3. 读取当前 PSP
   → MRS r0, PSP（此时 PSP 指向任务 A 已经压入硬件栈帧后的栈顶）

4. 获取任务 A 的 TCB
   → 通过全局指针 pxCurrentTCB（此时指向任务 A 的 TCB）
   → LDR r2, =pxCurrentTCB
   → LDR r2, [r2]（r2 = TCB_A 地址）

5. 手动保存 r4-r11 到任务 A 的栈
   → STMDB r0!, {r4-r11}（r0 自动更新为新栈顶）

6. 把更新后的栈顶写入 TCB_A->pxTopOfStack
   → STR r0, [r2]
   → 任务 A 的现场保存完成

7. 调用 vTaskSwitchContext()
   → 调度器从就绪列表中选择下一个任务
   → pxCurrentTCB 从指向 TCB_A 改为指向 TCB_B

8. 获取任务 B 的 TCB
   → 通过全局指针 pxCurrentTCB（此时已指向任务 B 的 TCB）
   → LDR r2, =pxCurrentTCB
   → LDR r2, [r2]（r2 = TCB_B 地址）

9. 从 TCB_B->pxTopOfStack 读取任务 B 的栈顶地址
   → LDR r0, [r2]（r0 现在指向任务 B 上次保存的上下文现场）

10. 从任务 B 的栈恢复 r4-r11
    → LDMIA r0!, {r4-r11}（r0 自动递增，指向硬件栈帧起始位置）

11. 把更新后的栈顶写回 PSP
    → MSR PSP, r0（现在 PSP 指向任务 B 的硬件栈帧）

12. 异常返回
    → BX lr
    → 硬件自动从 PSP 指向的位置恢复 r0-r3、r12、lr、pc、xPSR
    → PC 被恢复后，任务 B 从上次被切走的位置继续运行
```

### 10.4.3 切换本质

**保存旧任务的 PSP 到旧 TCB，从新 TCB 恢复新 PSP。**

最关键的字段是 `TCB_t->pxTopOfStack`，它保存每个任务自己的栈顶位置。

## 10.5 上下文保存哪些寄存器？

任务现场分两部分保存：

| 保存者    | 寄存器                       | 时机                 |
| ------ | ------------------------- | ------------------ |
| 硬件自动保存 | r0-r3、r12、lr、pc、xPSR       | 进入异常时自动压栈          |
| 软件手动保存 | r4-r11                    | PendSV_Handler 中保存 |

**为什么这样分？**

- `r0-r3、r12` 是 **caller-saved**（调用者保存寄存器），进入异常时硬件自动保存
- `r4-r11` 是 **callee-saved**（被调用者保存寄存器），需要 PendSV 用汇编手动保存
- `sp/psp` 不单独当普通寄存器保存，而是写入当前任务 TCB 的 `pxTopOfStack`

## 10.6 PendSV 中的临界保护

PendSV 切换过程中会调用 `vTaskSwitchContext()`，这个函数会操作就绪链表、延时链表、`pxCurrentTCB` 等调度器核心数据结构，因此需要临界保护。

**重要**：在 Cortex-M3/M4/M7 常见端口中，FreeRTOS 通常使用 **BASEPRI** 屏蔽**可以调用 FreeRTOS API 的中断**。

### 10.6.1 BASEPRI 机制

设置一个优先级阈值（如 `configMAX_SYSCALL_INTERRUPT_PRIORITY`）：
- 优先级数值 **≥ 阈值** 的中断被屏蔽（可调用 FreeRTOS API 的中断）
- 优先级数值 **< 阈值** 的高优先级中断**不受影响**，可以打断 PendSV

**好处**：既能保护调度器数据结构，又不影响更高优先级的紧急中断（如电机失速保护、紧急制动等）。

---

# 11 FromISR API

## 11.1 为什么中断里不能调用普通API

> [!important] 阻塞的本质是"任务切换"，中断处理程序不是任务，使用普通API（如`xQueueSend`）如果资源不可用，会尝试让当前"任务"进入阻塞态、切换给别的任务运行。但中断处理程序根本不是一个任务——它没有自己的TCB、没有PSP栈，用的是MSP，只是"借用"被打断的那个任务的执行流。

FromISR版本的核心区别：**绝不阻塞**，操作不能立刻完成就直接返回失败状态码，把处理权交还调用者。

```c
// 普通版本（任务里用，可能阻塞）
xQueueSend(myQueue, &data, portMAX_DELAY);

// FromISR版本（中断里用，绝不阻塞）
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
xQueueSendFromISR(myQueue, &data, &xHigherPriorityTaskWoken);
```

## 11.2 xHigherPriorityTaskWoken 与 portYIELD_FROM_ISR

`xHigherPriorityTaskWoken`是中断处理函数里**自己声明的局部变量**，传给FromISR函数——如果这次操作唤醒了一个优先级更高的任务，函数内部会把它改成`pdTRUE`。

```c
void UART_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    xQueueSendFromISR(myQueue, &data, &xHigherPriorityTaskWoken);

    // 中断处理最后必须手动调用，FreeRTOS不会自动做
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

> [!warning] 忘了调用这一句，不会真正触发切换——高优先级任务要等到下一次SysTick才被调度到，白白多等一个时间片

