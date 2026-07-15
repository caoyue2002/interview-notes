
前言：我发现这玩意写得异常不友好，没看过源码是很难看懂的。我建议这玩意要搭配源码食用。B站有很多RTOS源码详解，我觉得最好先过一遍源码详解再看这下面这些东西。

列表是RTOS最顶层的玩意，所有调度，队列都在列表的结构下。我希望复习的时候是一个从顶向下复习的过程。

# 1 列表概述：调度器如何从就绪任务找到下一个任务

FreeRTOS 不会把所有任务放在一个大数组里逐个扫描，而是用多个 `List_t` 把不同状态、不同优先级的任务组织起来。

- 每个优先级对应一个就绪列表：`pxReadyTasksLists[priority]`
- 一个任务要挂到某个列表中，靠的是 TCB 内嵌的 `ListItem_t`
- 调度器选中任务后，最终得到的是该任务的 `TCB_t *`，再由 PendSV 根据 TCB 中保存的栈顶恢复任务现场

先记住最核心的链路：

```text
就绪列表 List_t
    ↓ pxIndex->pxNext
当前要选中的 ListItem_t（通常就是任务的 xStateListItem）
    ↓ pvOwner
该 ListItem_t 所属任务的 TCB_t
    ↓ 更新 pxCurrentTCB
PendSV 根据 TCB->pxTopOfStack 恢复任务现场
    ↓
硬件恢复保存的 PC，任务从上次位置继续；首次启动时进入任务函数
```

## 1.1 FreeRTOS 中有哪些 `List_t`

`List_t` 是 FreeRTOS 复用的双向循环链表容器。任务状态变化时，内核会把 TCB 内嵌的 `ListItem_t` 从一条链表移到另一条链表。

任务相关的主要 `List_t` 实例如下：

| 列表                                               | 保存的任务                      | 用途                                                                       |
| ------------------------------------------------ | -------------------------- | ------------------------------------------------------------------------ |
| `pxReadyTasksLists[priority]`                    | 已就绪任务                      | 每个优先级一条就绪链表；调度器从最高优先级的非空链表中选取下一个任务。                                      |
| `pxDelayedTaskList`                              | 未到唤醒时间的延时任务                | 按 `xItemValue`（唤醒 tick）升序排列；当前 tick 到达时，任务转入就绪列表。                        |
| `pxOverflowDelayedTaskList`                      | tick 即将溢出后才会到期的延时任务        | 与普通延时列表配对，避免 tick 计数器回绕破坏排序。tick 溢出时，两条延时列表交换角色。                         |
| `xPendingReadyList`                              | 已被事件唤醒、但暂不能立刻移入就绪列表的任务     | 调度器挂起期间，ISR/其他操作先将任务放到这里；恢复调度时再统一移入就绪列表。                                 |
| `xSuspendedTaskList`                             | 被 `vTaskSuspend()` 显式挂起的任务 | 不会因 tick 到期或资源就绪自动恢复，必须显式 `vTaskResume()`。                               |
| `xTasksWaitingToSend` / `xTasksWaitingToReceive` | 等待队列、信号量等对象的任务             | 这两条 `List_t` 属于具体 `Queue_t` 对象，而非全局任务列表；任务在等待资源时用 `xEventListItem` 挂入其中。 |
| `xTasksWaitingTermination`                       | 等待释放资源的已删除任务               | 仅在 `INCLUDE_vTaskDelete == 1` 时使用，由 Idle 任务回收其 TCB 和栈。                   |

任务的典型移动路径：

```text
创建任务
    → 对应优先级的 pxReadyTasksLists[priority]

调用 vTaskDelay()
    → pxDelayedTaskList 或 pxOverflowDelayedTaskList
    → 到期后回到 pxReadyTasksLists[priority]

等待队列/信号量且资源不可用
    → 延时列表（若设置了超时） + Queue_t 的 xTasksWaitingToSend/Receive
    → 资源到来或超时后回到 pxReadyTasksLists[priority]

调用 vTaskSuspend()
    → xSuspendedTaskList
    → vTaskResume() 后回到 pxReadyTasksLists[priority]
```

真正执行任务选择时，调度器只访问 `pxReadyTasksLists[]`：先找到最高优先级的非空就绪列表，再在该 `List_t` 内轮转选择任务。
![[Pasted image 20260713015341.png]]

---

## 1.2 `List_t` 的结构与链接方式

每一条就绪、延时、挂起或事件等待链表，底层都是一个 `List_t`。它记录真实节点数量、当前轮转位置，并提供一个不属于任务的哨兵节点：

```c
typedef struct xLIST
{
    UBaseType_t uxNumberOfItems;  // 当前链表中的真实节点数
    ListItem_t *pxIndex;          // 轮转遍历时的当前位置
    MiniListItem_t xListEnd;      // 哨兵尾节点，负责把链表首尾连成环
} List_t;
```

### 1.2.1 `vListInitialise()`：把 `List_t` 初始化为空的循环链表

每个 `List_t` 在第一次插入任务节点前，都要调用：

```c
vListInitialise( List_t * const pxList );
```

它不分配内存，也不创建任务；它只把传入的 `List_t` 配置成一个可插入节点的空链表。核心初始化逻辑可以简化为：

```c
void vListInitialise( List_t * const pxList )
{
    pxList->xListEnd.xItemValue = portMAX_DELAY; // 哨兵排序值最大
    pxList->xListEnd.pxNext = ( ListItem_t * ) &( pxList->xListEnd );
    pxList->xListEnd.pxPrevious = ( ListItem_t * ) &( pxList->xListEnd );

    pxList->pxIndex = ( ListItem_t * ) &( pxList->xListEnd );
    pxList->uxNumberOfItems = 0;
}
```

初始化完成后的字段含义：

| 字段 | 初始化结果 | 作用 |
| --- | --- | --- |
| `xListEnd.xItemValue` | `portMAX_DELAY` | 作为最大排序值，保证按 `xItemValue` 升序插入时，真实节点排在哨兵之前。 |
| `xListEnd.pxNext` | 指向 `xListEnd` 自己 | 空链表的“下一个节点”仍是哨兵。 |
| `xListEnd.pxPrevious` | 指向 `xListEnd` 自己 | 空链表的“上一个节点”仍是哨兵。 |
| `pxIndex` | 指向 `xListEnd` | 第一次执行 `pxIndex = pxIndex->pxNext` 后，才能选到第一个真实节点。 |
| `uxNumberOfItems` | `0` | 表示当前没有任何真实 `ListItem_t`；哨兵不计入此数量。 |

因此，刚初始化完成时的结构是：

```text
pxIndex
   │
   ▼
xListEnd
   ├── pxNext ─────→ xListEnd
   └── pxPrevious ─→ xListEnd

uxNumberOfItems = 0
```

依次插入两个真实任务节点 A、B 后，链接变为：

```text
                 pxNext
xListEnd ─────────────────→ Task A 的 ListItem_t
   ↑                              │
   │                              │ pxNext
   └──────────────── Task B 的 ListItem_t
```

此时 `xListEnd → A → B → xListEnd` 形成循环。遍历到最后一个任务后，继续沿 `pxNext` 会到达 `xListEnd`；调度器跳过哨兵后回到第一个真实任务，因此不需要使用 `NULL` 表示链表尾部。

---

## 1.3 `ListItem_t`：任务挂入链表的节点

真正被串起来的是 `ListItem_t`。TCB 中内嵌了两个这种节点：

```c
typedef struct xLIST_ITEM
{
    TickType_t xItemValue;        // 排序值：如延时链表中的唤醒 tick
    struct xLIST_ITEM *pxNext;
    struct xLIST_ITEM *pxPrevious;
    void *pvOwner;                // 该节点属于谁
    struct xLIST *pxContainer;    // 当前挂在哪个 List_t
} ListItem_t;

typedef struct tskTaskControlBlock
{
    volatile StackType_t *pxTopOfStack;
    ListItem_t xStateListItem;    // 挂入任务状态列表：就绪/延时/挂起等
    ListItem_t xEventListItem;    // 挂入资源的事件等待列表
    // ...
} TCB_t;
```

创建任务时，内核会建立反向关联：

```c
listSET_LIST_ITEM_OWNER(&pxNewTCB->xStateListItem, pxNewTCB);
listSET_LIST_ITEM_OWNER(&pxNewTCB->xEventListItem, pxNewTCB);
```

因此：

```text
TCB_A.xStateListItem.pvOwner == TCB_A
TCB_A.xEventListItem.pvOwner == TCB_A
```

两个节点用途不同：

- `xStateListItem`：一个任务当前处于什么调度状态，就挂到对应状态列表。**就绪列表中挂的是它**。
- `xEventListItem`：任务阻塞等待队列、信号量等对象时，挂到该对象的等待列表。

同一个任务能同时处于“延时状态”与“等待某事件”的语义中，所以需要两个独立节点；一个 `ListItem_t` 不能同时挂入两条链表。

---

## 1.4 调度器如何沿 `pxNext` 选中一个 TCB

以当前最高就绪优先级为 `uxTopPriority` 为例，调度器先定位对应的就绪列表：

```c
List_t *pxReadyList = &pxReadyTasksLists[ uxTopPriority ];
```

然后在该列表中轮转。逻辑可简化为：

```c
pxReadyList->pxIndex = pxReadyList->pxIndex->pxNext;

if ( pxReadyList->pxIndex == ( ListItem_t * ) &pxReadyList->xListEnd )
{
    pxReadyList->pxIndex = pxReadyList->pxIndex->pxNext; // 跳过哨兵
}

pxCurrentTCB = ( TCB_t * ) pxReadyList->pxIndex->pvOwner;
```

也就是：

```text
1. 找最高优先级的非空就绪列表
2. pxIndex = pxIndex->pxNext
3. 若碰到 xListEnd 哨兵，则再走一次 pxNext 跳过它
4. 当前 ListItem_t 的 pvOwner 转回 TCB_t *
5. pxCurrentTCB 指向这个 TCB
```

同优先级任务都在同一条就绪链表上，`pxIndex` 每次向后移动一个真实节点，所以自然形成时间片轮转：

```text
就绪列表：xListEnd → A → B → C → xListEnd

第1次选择：pxIndex 指向 A
第2次选择：pxIndex 指向 B
第3次选择：pxIndex 指向 C
第4次选择：经过 xListEnd 后回到 A
```

> [!warning] `pxIndex` 是“本次轮转选到哪里”的游标；`uxNumberOfItems` 才是实际任务数。`xListEnd` 仅用于链接和遍历，不计入 `uxNumberOfItems`。

---

## 1.5 一次完整的任务切换顺序

下面以任务 A 切换到任务 B 为例，把“触发调度”到“B 开始运行”的顺序连起来。

```text
1. 任务 A 正在运行

2. SysTick、taskYIELD()，或 ISR 唤醒了更高优先级任务
   → 请求一次调度，挂起 PendSV

3. CPU 进入 PendSV_Handler
   → 硬件自动把 A 的 r0-r3、r12、lr、pc、xPSR 压入 A 的 PSP 栈
   → PendSV 再保存 A 的 r4-r11
   → 将更新后的 PSP 写入 TCB_A->pxTopOfStack

4. PendSV 调用 vTaskSwitchContext()
   → 找到最高优先级的非空就绪列表：pxReadyTasksLists[uxTopPriority]
   → pxIndex 沿 pxNext 移动到下一个 ListItem_t
   → 若到达 xListEnd 哨兵，则跳过哨兵，继续取第一个真实 ListItem_t
   → 通过 ListItem_t->pvOwner 得到对应的 TCB_B
   → pxCurrentTCB = TCB_B

5. vTaskSwitchContext() 返回 PendSV_Handler
   → 从 pxCurrentTCB->pxTopOfStack 读取 B 上次保存的 PSP
   → 恢复 B 的 r4-r11
   → 将恢复后的栈顶写回 PSP

6. PendSV 异常返回
   → 硬件从 B 的 PSP 栈自动恢复 r0-r3、r12、lr、pc、xPSR
   → PC 恢复为 B 上次被切走的位置，任务 B 继续运行
```

第一次启动某个任务时，FreeRTOS 在创建任务阶段已经提前在该任务栈中构造好了初始寄存器现场，其中 `PC` 预设为任务入口 `pxTaskCode`。所以第一次从该任务的 `pxTopOfStack` 恢复现场后，CPU 才会从任务函数入口开始执行；后续切换则恢复上次保存的 `PC`，继续原来的执行位置。

因此，`pvOwner` 这一步的作用只是把链表节点还原为任务的 TCB，供内核更新 `pxCurrentTCB`；任务代码并不是由调度器通过函数指针直接调用，而是由 PendSV 恢复任务栈中的 `PC` 后继续执行。

一句话总结：

> **`List_t` 决定“哪个任务有资格被选中”，`pxNext` 决定“同优先级下轮到谁”，`pvOwner` 把链表节点还原为 TCB，`pxTopOfStack` 决定“如何回到这个任务的执行现场”。**

---

# 2 任务切换

任务切换分成两件事：

```text
调度：决定下一个应该运行哪个 TCB
上下文切换：保存旧任务现场，恢复新任务现场
```

`xTaskIncrementTick()`、`vTaskSwitchContext()` 主要处理第一件事；Cortex-M 的 `PendSV_Handler` 主要处理第二件事。

## 2.1 宏观看一次任务从阻塞到运行

以任务 B 调用 `vTaskDelay()` 后到期、任务 A 当前正在运行的场景为例：

```text
任务 B 调用 vTaskDelay()
    → B 从 pxReadyTasksLists[B优先级] 移出
    → B 的 xStateListItem 按唤醒 tick 插入 pxDelayedTaskList

SysTick 周期到来
    → xTaskIncrementTick() 递增 xTickCount
    → 检查 pxDelayedTaskList 表头
    → 若表头任务 B 的唤醒 tick 已到：B 从延时列表移出
    → B 插入 pxReadyTasksLists[B优先级]

若 B 的优先级高于当前任务 A
    → 请求任务切换
    → PendSV 保存 A 的现场，恢复 B 的现场
    → B 开始运行
```

延时列表按唤醒 tick 升序排列，因此只需要检查表头：表头尚未到期时，后面的任务也一定尚未到期；表头到期时，内核连续取出所有已到期任务并移入对应的就绪列表。

> [!important] 任务到期后不是直接运行，而是先回到就绪列表。最终是否立刻执行，由它的优先级、抢占配置和当前任务状态共同决定。

## 2.2 就绪列表如何决定候选任务

`pxReadyTasksLists[]` 不是一条链表，而是一个按优先级索引的 `List_t` 数组：

```text
pxReadyTasksLists[0]       → 优先级 0 的就绪任务
pxReadyTasksLists[1]       → 优先级 1 的就绪任务
...
pxReadyTasksLists[N]       → 优先级 N 的就绪任务
```

优先级数值越大，优先级越高。`vTaskSwitchContext()` 从最高优先级的非空列表开始选择：

```text
1. 找到 uxTopReadyPriority 对应的 pxReadyTasksLists[uxTopReadyPriority]
2. 该列表的 pxIndex 沿 pxNext 移到下一个真实 ListItem_t
3. ListItem_t->pvOwner 得到对应 TCB
4. pxCurrentTCB 指向该 TCB
```

这里选择的是“**当前 `pxIndex` 的下一个节点**”，不是固定选择“最后注册的任务”。任务插入就绪列表后会参与该列表的轮转，实际轮到谁由当前 `pxIndex` 决定。

### 2.2.1 抢占与时间片轮转

任务进入就绪列表后，是否触发一次切换取决于配置与优先级：

| 条件 | 调度结果 |
| --- | --- |
| 新就绪任务优先级高于当前任务，且 `configUSE_PREEMPTION == 1` | 请求切换；高优先级任务抢占当前任务。 |
| 新就绪任务优先级低于当前任务 | 当前任务继续运行。 |
| 两个任务优先级相同 | 不会因为“新任务到期”自动抢占；是否轮转由 tick 时间片机制决定。 |

当 `configUSE_PREEMPTION == 1` 且 `configUSE_TIME_SLICING == 1` 时，若当前优先级的就绪列表中有多个任务，SysTick 会请求一次切换：

```text
同优先级就绪列表：A → B → C

第 1 个 tick：A 运行，SysTick 请求切换，pxIndex 移到 B
第 2 个 tick：B 运行，SysTick 请求切换，pxIndex 移到 C
第 3 个 tick：C 运行，SysTick 请求切换，pxIndex 移回 A
```

因此，时间片长度是 **一个 tick 周期**，由 `configTICK_RATE_HZ` 决定。例如 `configTICK_RATE_HZ = 1000` 时，一个 tick 为 1 ms；它不是 FreeRTOS 固定写死的 1 ms。

以上只描述“谁应当运行”。接下来才进入 Cortex-M 上真正的寄存器保存与恢复；`List_t` 的插入、删除、优先级位图和临界区细节在后续章节再展开。

---

## 2.3 `xPortSysTickHandler()`：更新 tick 并请求 PendSV

SysTick 是 Cortex-M 的系统定时器异常。常见 Cortex-M FreeRTOS port 中，SysTick 会周期性进入 `xPortSysTickHandler()`；该函数的主线可以简化为：

```c
void xPortSysTickHandler( void )
{
    // 进入内核临界保护：保护 tick 和任务列表等调度器数据

    if( xTaskIncrementTick() != pdFALSE )
    {
        portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
        // 设置 PendSV 挂起位：请求稍后执行真正的任务切换
    }

    // 退出内核临界保护
}
```

这段代码中先不展开 `xPortRaiseBASEPRI()`：它的作用是暂时屏蔽一部分会调用 FreeRTOS API 的中断，避免它们同时修改调度器链表。此处只需要记住，`xTaskIncrementTick()` 操作 tick、延时列表和就绪列表时必须受保护。

`xTaskIncrementTick()` 主要完成：

```text
1. xTickCount 加 1
2. 处理 tick 溢出时两条延时列表的交换
3. 把已经到期的任务从延时列表移到对应优先级的就绪列表
4. 判断是否需要切换：
   - 有更高优先级任务就绪；或
   - 开启时间片且当前优先级有多个就绪任务
5. 返回是否需要切换的结果
```

`xPortSysTickHandler()` 不保存任务寄存器，也不直接调用 `vTaskSwitchContext()`。它只在 `xTaskIncrementTick()` 返回“需要切换”时设置 PendSV 挂起位。

```text
SysTick_Handler
    → xPortSysTickHandler()
    → xTaskIncrementTick()
    → 若需要切换：设置 PendSVSET
    → SysTick 退出

PendSV 在没有更高优先级异常需要处理时运行
    → 保存旧任务现场
    → vTaskSwitchContext() 选出新任务
    → 恢复新任务现场
```

PendSV 通常设为最低异常优先级：高优先级外设中断可以先完成，避免任务切换过程抢占更紧急的中断处理。

---

## 2.4 任务切换时涉及的寄存器

Cortex-M 有 `R0` 到 `R15`，另有 `xPSR` 状态寄存器。任务被切走后，要让它能像从未被打断一样继续执行，就必须恢复该任务当时依赖的寄存器状态。

| 寄存器 | 主要作用 | PendSV 切换中的处理 |
| --- | --- | --- |
| `R0-R3` | 临时数据、函数前 4 个参数、返回值 | 异常进入时由硬件自动压入任务 PSP 栈；异常返回时自动恢复。 |
| `R4-R11` | 通用寄存器；按 AAPCS 属于被调用者保存寄存器 | PendSV 用汇编手动压入/恢复。 |
| `R12` | 临时寄存器（IP） | 异常进入时硬件自动压入，异常返回时自动恢复。 |
| `R13` / `SP` | 栈指针 | 任务在线程模式通常用 `PSP`；最终的 `PSP` 保存到 `TCB->pxTopOfStack`。异常 Handler 自己使用 `MSP`。 |
| `R14` / `LR` | 正常函数调用时保存返回地址 | 异常处理时保存 `EXC_RETURN`，它告诉硬件异常返回时使用哪种栈和栈帧。任务原来的 `LR` 位于硬件自动栈帧中。 |
| `R15` / `PC` | 下一条要执行的指令地址 | 异常进入时自动压栈；异常返回时恢复。恢复后的 PC 决定任务从哪里继续执行。 |
| `xPSR` | 条件标志、Thumb 状态和当前异常信息 | 异常进入时自动压栈；异常返回时恢复。 |

这里的“现场”不是逐个保存局部变量。局部变量原本就在该任务自己的栈中；若某个局部值暂存在寄存器中，它会随着寄存器现场一起被保留。

---

## 2.5 PendSV：压栈保存旧任务，恢复新任务

以下以常见 Cortex-M3/M4/M7 的整数寄存器现场为例，不展开 FPU 现场。

### 2.5.1 异常入口：硬件先保存一半现场

任务 A 在线程模式下使用 `PSP` 运行。PendSV 异常被响应时，硬件自动将 8 个寄存器压入 **任务 A 的 PSP 栈**：

```text
R0, R1, R2, R3, R12, LR, PC, xPSR
```

此时 PendSV Handler 自己使用 `MSP` 运行，但任务 A 的硬件栈帧仍保留在 A 的 PSP 栈中：

```text
低地址
┌──────────────────────────────┐
│ 当前 PSP → R0                │
│            R1                │
│            R2                │
│            R3                │
│            R12               │
│            LR                │
│            PC                │
│            xPSR              │
│            A 原有的调用栈、局部变量 │
└──────────────────────────────┘
高地址
```

### 2.5.2 PendSV 手动补全现场并保存 SP

硬件未自动保存 `R4-R11`。PendSV 读取当前 PSP 后，用汇编将这 8 个寄存器继续压入任务 A 的栈，再将压栈后的 PSP 写入当前 TCB：

```text
MRS r0, PSP
    → r0 = A 的当前 PSP（指向硬件栈帧）

STMDB r0!, {r4-r11}
    → R4-R11 压入 A 的 PSP 栈，r0 更新为新的栈顶

通过 pxCurrentTCB 找到 TCB_A，再将 r0 写入其首字段 pxTopOfStack
    → TCB_A->pxTopOfStack = r0
```

保存完成后，A 的完整整数现场布局为：

```text
低地址
┌──────────────────────────────┐
│ TCB_A->pxTopOfStack → R4-R11 │  ← PendSV 软件保存
├──────────────────────────────┤
│ R0-R3, R12, LR, PC, xPSR     │  ← 硬件异常入口自动保存
├──────────────────────────────┤
│ A 原有的调用栈、局部变量         │
└──────────────────────────────┘
高地址
```

### 2.5.3 选择任务 B

旧任务 A 的 PSP 已经保存在 `TCB_A->pxTopOfStack` 后，PendSV 调用：

```c
vTaskSwitchContext();
```

该函数按 `2.2` 的规则从最高优先级就绪列表中选择任务 B，并把：

```c
pxCurrentTCB = TCB_B;
```

### 2.5.4 从任务 B 的栈恢复现场

PendSV 随后从新的 `pxCurrentTCB` 取出 B 的保存栈顶：

```text
LDR r0, [pxCurrentTCB]
    → r0 = TCB_B->pxTopOfStack

LDMIA r0!, {r4-r11}
    → 恢复 B 的 R4-R11，r0 移到 B 的硬件栈帧

MSR PSP, r0
    → PSP 指向 B 的硬件栈帧

BX LR
    → 异常返回，硬件从 PSP 自动恢复 R0-R3、R12、LR、PC、xPSR
```

当 `PC` 被恢复后，CPU 就回到任务 B 上次被打断的位置；如果 B 是第一次运行，则任务创建时预先构造的栈帧使 `PC` 指向 B 的任务函数入口。

### 2.5.5 保存和恢复为什么不需要记录“变量数量”

保存与恢复的寄存器集合由 Cortex-M 异常机制和当前 FreeRTOS port 的 PendSV 汇编固定定义：

```text
硬件固定保存/恢复：R0-R3、R12、LR、PC、xPSR
PendSV 固定保存/恢复：R4-R11
TCB 只记录：该固定布局的起始 PSP，即 pxTopOfStack
```

因此，内核不需要记录“任务有多少个局部变量”或“本次压了多少个变量”。局部变量已位于各自任务的私有栈中；只要恢复相同的 PSP 和寄存器现场，任务就能继续执行。

> [!warning] 带 FPU 的 Cortex-M4F/M7F 在任务使用浮点上下文时还可能有额外浮点寄存器栈帧，具体保存范围由端口汇编和异常返回状态决定。本节先以最基础的整数现场说明主链路。

---

# 3 阻塞延时

阻塞延时的核心不是“任务自己数时间”，而是：

```text
任务记录自己应该在哪个 tick 被唤醒
    → 按唤醒时间放进延时列表
    → SysTick 每次推进 xTickCount
    → 时间到后，内核把任务移回就绪列表
```

## 3.1 `xTickCount`：系统时间基准

`xTickCount` 是 FreeRTOS 的全局 tick 计数器。SysTick 每触发一次，`xTaskIncrementTick()` 就会将它加 1：

```text
configTICK_RATE_HZ = 1000
    → 1 个 tick = 1 ms
    → xTickCount 每增加 1，表示经过约 1 ms
```

`xTickCount` 不是绝对的真实时间，而是内核调度使用的离散时间基准。任务延时、软件定时器和带超时参数的阻塞 API 都以 tick 为单位计算等待时间。

例如当前：

```text
xTickCount = 100
vTaskDelay(20)
```

表示该任务最早应在：

```text
xTimeToWake = 100 + 20 = 120
```

对应的 tick 到来后被解除阻塞。

---

## 3.2 调用 `vTaskDelay()` 后发生什么

```c
vTaskDelay( xTicksToDelay );
```

若 `xTicksToDelay > 0`，当前任务会进入阻塞态。主要过程可简化为：

```text
1. 当前任务从自己的 pxReadyTasksLists[priority] 就绪列表移出

2. 计算唤醒时刻
   xTimeToWake = xTickCount + xTicksToDelay

3. 设置当前 TCB 的状态链表节点
   pxCurrentTCB->xStateListItem.xItemValue = xTimeToWake

4. 将 xStateListItem 按 xItemValue 升序插入延时列表

5. 当前任务不再就绪，调用调度器选择其他就绪任务运行
```

这里使用的是 `xStateListItem`：同一个节点不能同时位于就绪列表和延时列表，所以任务必须先从就绪列表摘下，再插入延时列表。

### 3.2.1 为什么延时列表要按 `xItemValue` 升序插入

`vListInsert()` 会按 `ListItem_t.xItemValue` 从小到大插入。对于延时任务，`xItemValue` 就是它的唤醒 tick：

```text
当前 xTickCount = 100

任务 A：vTaskDelay(10)  → xItemValue = 110
任务 B：vTaskDelay(30)  → xItemValue = 130
任务 C：vTaskDelay(20)  → xItemValue = 120

pxDelayedTaskList：xListEnd → A(110) → C(120) → B(130) → xListEnd
```

因此，延时列表表头永远是**最早到期的任务**。SysTick 不需要在每个 tick 遍历所有阻塞任务，只需先检查表头即可。

> [!important] 这是“按时间排序 + 只检查最早截止项”的思路。表头还没到期时，后面的任务唤醒时间更晚，也一定不需要检查。

---

## 3.3 到期唤醒：`xNextTaskUnblockTime`

内核用 `xNextTaskUnblockTime` 缓存当前延时列表表头任务的唤醒 tick。

```text
xTickCount             → 当前系统 tick
xNextTaskUnblockTime   → 下一个最早需要解除阻塞的 tick
```

每次 SysTick 调用 `xTaskIncrementTick()` 时，先比较：

```text
若 xTickCount < xNextTaskUnblockTime
    → 没有任何延时任务到期，直接结束延时检查

若 xTickCount >= xNextTaskUnblockTime
    → 至少有一个任务可能到期，检查延时列表表头
```

到期后的实际处理顺序：

```text
1. 读取 pxDelayedTaskList 的表头任务
2. 若表头 xItemValue > xTickCount
   → 当前没有任务到期
   → xNextTaskUnblockTime = 表头 xItemValue
   → 结束

3. 若表头 xItemValue <= xTickCount
   → 将该任务的 xStateListItem 从延时列表移出
   → 若它还在事件等待列表，也将 xEventListItem 从事件列表移出
   → 将任务插入 pxReadyTasksLists[该任务优先级]
   → 继续检查下一个表头
```

当没有延时任务时，`xNextTaskUnblockTime` 设为 `portMAX_DELAY`，表示在 tick 回绕前没有任务需要因“纯延时”被唤醒。

> [!important] 到期任务的去向是 **就绪列表 `pxReadyTasksLists[]`**，不是另一个延时列表。它回到就绪列表后，才由优先级调度决定何时真正获得 CPU。

---

## 3.4 两条延时列表：处理 `xTickCount` 溢出

`TickType_t` 是有限位数的无符号整数。假设使用 32 位 tick：

```text
0xFFFFFFFE
0xFFFFFFFF
0x00000000  ← 再加 1 后回绕
```

如果当前 `xTickCount` 已接近最大值，而某个任务的唤醒时刻在回绕之后，直接把所有任务放在同一条按数值升序排列的列表中会出错：回绕后的较小数值会排到列表前面。

FreeRTOS 因此维护两条延时列表，并通过两个指针访问：

| 指针 | 当前含义 |
| --- | --- |
| `pxDelayedTaskList` | 当前 tick 周期内到期的任务，唤醒 tick 不发生回绕。 |
| `pxOverflowDelayedTaskList` | 需要等到下一次 tick 回绕之后才到期的任务。 |

任务阻塞时先计算：

```text
xTimeToWake = xTickCount + xTicksToDelay
```

然后比较 `xTimeToWake` 与当前 `xTickCount`：

```text
xTimeToWake >= xTickCount
    → 本轮 tick 周期内到期
    → 按 xItemValue 插入 pxDelayedTaskList

xTimeToWake < xTickCount
    → 加法发生回绕，到下一轮 tick 周期才到期
    → 按 xItemValue 插入 pxOverflowDelayedTaskList
```

当 `xTickCount` 从最大值回绕到 0 时：

```text
1. 当前 pxDelayedTaskList 中的任务都应已到期并被处理
2. 交换 pxDelayedTaskList 与 pxOverflowDelayedTaskList 指针
3. 原来的 overflow 列表成为新的当前延时列表
4. 更新 xNextTaskUnblockTime 为新列表表头的唤醒 tick
```

这样两条列表内部始终只需要按普通无符号数升序排序，不必在每次比较时处理“时间跨回绕”的特殊逻辑。

---

## 3.5 一个 tick 可能同时唤醒多个任务

多个任务可以具有相同的唤醒 tick，因此一次 `xTaskIncrementTick()` 不能只释放一个任务，而是持续检查表头，直到遇到第一个尚未到期的任务：

```text
xTickCount = 120

延时列表：A(110) → B(120) → C(120) → D(130)

处理结果：
1. A(110) 到期 → 移入就绪列表
2. B(120) 到期 → 移入就绪列表
3. C(120) 到期 → 移入就绪列表
4. D(130) 未到期 → 停止

xNextTaskUnblockTime = 130
```

这些被释放的任务都进入各自优先级的 `pxReadyTasksLists[]`。若其中存在优先级高于当前运行任务的任务，且开启抢占，`xTaskIncrementTick()` 会返回需要切换，随后由 PendSV 完成切换。

---

## 3.6 带 `xTicksToWait` 参数的 API 如何延时等待

很多阻塞式 API 都带有：

```c
TickType_t xTicksToWait
```

例如：

```c
xQueueReceive( xQueue, pvBuffer, xTicksToWait );
xQueueSend( xQueue, pvItem, xTicksToWait );
xSemaphoreTake( xSemaphore, xTicksToWait );
```

它们不是简单调用一次 `vTaskDelay(xTicksToWait)`，因为任务还要等待一个**事件条件**：队列变为非空、队列出现空位、信号量可获取等。

资源暂时不可用且 `xTicksToWait > 0` 时，任务会同时使用 TCB 中的两个列表节点：

```text
xStateListItem
    → 插入 pxDelayedTaskList / pxOverflowDelayedTaskList
    → 负责“最长等待 xTicksToWait，时间到就超时”

xEventListItem
    → 插入 Queue_t 的 xTasksWaitingToReceive / xTasksWaitingToSend
    → 负责“资源一旦可用，就提前唤醒任务”
```

因此等待过程有两条出口：

```text
资源先到
    → 队列/信号量操作从事件等待列表取出一个任务
    → 同时将该任务从延时列表移出
    → 任务进入就绪列表
    → API 返回成功

超时先到
    → xTaskIncrementTick() 从延时列表取出任务
    → 同时将该任务从事件等待列表移出
    → 任务进入就绪列表
    → API 发现等待时间已耗尽，返回超时/失败状态
```

特殊参数：

| `xTicksToWait` | 含义 |
| --- | --- |
| `0` | 不阻塞；资源不可用立刻返回。 |
| 正数 | 最多等待指定 tick 数；资源可提前到来。 |
| `portMAX_DELAY` | 在配置允许时可无限等待，直到资源到来；不应把它理解为“普通的超大有限延时”。 |

当 `xTicksToWait == portMAX_DELAY` 且 `INCLUDE_vTaskSuspend == 1` 时，内核可以把任务的 `xStateListItem` 放入 `xSuspendedTaskList`，而不是延时列表：因为它没有一个需要到期检查的有限唤醒 tick。此时任务仍通过 `xEventListItem` 挂在队列/信号量的事件等待列表中，资源到来时再被唤醒。

一句话总结：

> **`vTaskDelay()` 只有“时间到”这一条唤醒路径；带 `xTicksToWait` 的队列/信号量 API 同时登记“事件到”和“时间到”两条路径，谁先发生就按谁解除阻塞。**

---

# 4 队列概述

FreeRTOS 的队列不仅保存数据，还保存“因为队列空而等着读”的任务和“因为队列满而等着写”的任务。

```text
队列数据区
    → 保存发送者拷贝进来的 item

xTasksWaitingToReceive
    → 队列空时，等待接收数据的任务

xTasksWaitingToSend
    → 队列满时，等待写入数据的任务
```

本节只讨论**普通消息队列**，不展开信号量、互斥量及队列锁定等复用机制。

## 4.1 队列创建：`xQueueCreate()` 到 `xQueueGenericReset()`

普通队列通常通过：

```c
QueueHandle_t xQueue = xQueueCreate( uxQueueLength, uxItemSize );
```

创建。`xQueueCreate()` 是一个宏，它会调用 `xQueueGenericCreate()`；对普通消息队列而言，主线如下：

```text
xQueueCreate(uxQueueLength, uxItemSize)
    → xQueueGenericCreate(uxQueueLength, uxItemSize, queueQUEUE_TYPE_BASE)
    → 分配 Queue_t 结构体 + uxQueueLength * uxItemSize 字节的数据缓冲区
    → prvInitialiseNewQueue(...)
    → 设置队列的固定参数
    → xQueueGenericReset(pxNewQueue, pdTRUE)
    → 返回 QueueHandle_t
```

动态创建时，`xQueueGenericCreate()` 一次分配连续内存：前半段是 `Queue_t`，后半段是实际保存 item 的数据缓冲区。随后把数据区首地址传给 `prvInitialiseNewQueue()`。这里不展开静态创建；静态创建的区别只是 `Queue_t` 和数据缓冲区由用户提供，后续初始化语义相同。

### 4.1.1 `prvInitialiseNewQueue()`：先确定容量、item 大小和数据区首地址

对于普通队列，`prvInitialiseNewQueue()` 先完成三项设置：

```text
pcHead     = 数据缓冲区首地址
uxLength   = uxQueueLength
uxItemSize = uxItemSize
```

然后调用：

```c
xQueueGenericReset( pxNewQueue, pdTRUE );
```

`pdTRUE` 很关键：它明确告诉 `xQueueGenericReset()`，当前对象是一个**刚创建的新队列**，还不可能存在等待发送或等待接收的任务。因此 reset 的目标是建立干净的初始状态，而不是处理已阻塞任务。

### 4.1.2 `xQueueGenericReset()`：建立一个空的环形队列

无论是新建队列，还是运行中调用 reset，函数首先都会把数据区相关字段恢复到“空队列”的状态：

```text
pcTail = pcHead + uxLength * uxItemSize
    → 数据区末尾后的标记地址

uxMessagesWaiting = 0
    → 当前没有任何有效 item

pcWriteTo = pcHead
    → 下一次发送从第 0 个 item 槽位写入

pcReadFrom = pcHead + (uxLength - 1) * uxItemSize
    → 记录为最后一个槽位；第一次接收会先向后移动一个 item，正好绕回 pcHead 并读出第 0 个 item

cRxLock = queueUNLOCKED
cTxLock = queueUNLOCKED
    → 队列不处于锁定状态
```

因此，刚创建完成时可以把读写位置理解为：

```text
pcHead                                      pcTail
  ↓                                            ↓
┌───────┬───────┬───────┬───────┐
│ item0 │ item1 │ item2 │ item3 │
└───────┴───────┴───────┴───────┘
  ↑                                   ↑
pcWriteTo                         pcReadFrom

uxMessagesWaiting = 0
```

`pcReadFrom` 看起来位于最后一个槽位并不表示队列中有数据；它只是让接收路径的“先移动、再读取”逻辑在第一次读取时自然回到 `pcHead`。

### 4.1.3 两个 `xNewQueue` 分支：为什么只处理等待发送列表

字段复位后，`xQueueGenericReset()` 按 `xNewQueue` 分成两种语义：

```c
if( xNewQueue == pdFALSE )
{
    // reset 一个已经在运行中的队列
}
else
{
    // 初始化一个刚创建的队列
}
```

#### 新建队列：`xNewQueue == pdTRUE`

`xQueueCreate()` 走的是这个分支。函数分别调用：

```c
vListInitialise( &pxQueue->xTasksWaitingToSend );
vListInitialise( &pxQueue->xTasksWaitingToReceive );
```

两条事件等待列表都会被初始化为空循环链表：

```text
xTasksWaitingToSend    → 空列表
xTasksWaitingToReceive → 空列表
```

此时没有任务可能在等待该队列，所以这里只需要准备好列表容器，不需要唤醒任何任务。

#### 运行中 reset：`xNewQueue == pdFALSE`

这个分支面对的是已经存在的队列。reset 后 `uxMessagesWaiting = 0`，即队列一定为空。于是两个等待列表不能采用相同处理：

```text
xTasksWaitingToReceive：继续阻塞
xTasksWaitingToSend：若非空，解除其中一个任务的阻塞
```

原因直接来自 reset 后的队列状态：

```text
等待接收的任务等待的是“队列中出现数据”
    → reset 后队列为空，条件仍不满足
    → xTasksWaitingToReceive 中的任务必须继续等待

等待发送的任务等待的是“队列中出现空位”
    → reset 后 uxMessagesWaiting = 0，队列从满变为空
    → 已经具备发送条件
    → 应从 xTasksWaitingToSend 中解除一个任务的阻塞
```

源码只在 `xTasksWaitingToSend` 非空时调用一次 `xTaskRemoveFromEventList()`：

```c
if( listLIST_IS_EMPTY( &pxQueue->xTasksWaitingToSend ) == pdFALSE )
{
    if( xTaskRemoveFromEventList( &pxQueue->xTasksWaitingToSend ) != pdFALSE )
    {
        queueYIELD_IF_USING_PREEMPTION();
    }
}
```

它会从按优先级排列的等待发送列表中取出一个任务，将该任务从事件等待状态转回就绪状态；如果该任务优先级更高且使用抢占调度，则请求切换。这里只解除**一个**发送者，因为一次 reset 虽然清空了整个队列，但被唤醒的任务恢复运行后仍需重新检查队列并实际完成发送；后续发送、接收操作会继续按普通队列规则唤醒其他任务。

> **创建队列时，两个等待列表都只是初始化为空；重置运行中队列时，接收等待者因队列仍为空而保留阻塞，发送等待者因队列已产生空位而唤醒一个。**

---

## 4.2 `Queue_t` / `xQUEUE` 结构

内核源码中结构体历史名称为 `xQUEUE`，随后定义：

```c
typedef xQUEUE Queue_t;
```

普通队列的核心字段可简化为：

```c
typedef struct QueueDefinition
{
    int8_t *pcHead;       // 数据缓冲区起始地址
    int8_t *pcWriteTo;    // 下一次写入的位置

    int8_t *pcTail;       // 数据缓冲区末尾后的标记地址
    int8_t *pcReadFrom;   // 上一次读取的位置

    List_t xTasksWaitingToSend;
    List_t xTasksWaitingToReceive;

    volatile UBaseType_t uxMessagesWaiting;
    UBaseType_t uxLength;
    UBaseType_t uxItemSize;
} Queue_t;
```

> [!note] 真实源码中 `pcTail`、`pcReadFrom` 位于 `u.xQueue` 内的 union 中；这里为只说明普通队列路径，省略 union 层级后直接展示它们的作用。

### 4.2.1 数据缓冲区字段

| 字段 | 普通队列中的作用 |
| --- | --- |
| `pcHead` | 队列数据缓冲区首地址。动态创建时，通常指向紧跟在 `Queue_t` 结构体后的数据区；静态创建时，指向用户提供的缓冲区。 |
| `pcWriteTo` | 下一条发送数据应复制到的位置。写入一个 item 后向后移动 `uxItemSize` 字节，到末尾后绕回 `pcHead`。 |
| `pcTail` | 数据区末尾后的标记地址，用来判断写指针和读指针是否需要绕回。 |
| `pcReadFrom` | 上一次读取 item 的位置。接收时先向后移动一个 item，再从新位置复制数据；到末尾后同样绕回。 |
| `uxLength` | 队列容量，单位是 **item 个数**，不是字节数。 |
| `uxItemSize` | 每个 item 的字节数。普通队列发送的是值拷贝，实际缓冲区大小约为 `uxLength * uxItemSize`。 |
| `uxMessagesWaiting` | 当前有效 item 数。`0` 表示空，等于 `uxLength` 表示满。即使读写指针绕回重合，也能靠它区分“空”和“满”。 |

普通队列可以看作环形缓冲区：

```text
pcHead                                      pcTail
  ↓                                            ↓
┌───────┬───────┬───────┬───────┐
│ item0 │ item1 │ item2 │ item3 │
└───────┴───────┴───────┴───────┘
            ↑                   ↑
       pcReadFrom           pcWriteTo
```

### 4.2.2 两条任务等待列表

| 字段 | 队列状态 | 保存的 TCB 节点 | 作用 |
| --- | --- | --- | --- |
| `xTasksWaitingToReceive` | 队列为空 | 等待接收的任务的 `xEventListItem` | 发送者成功写入数据后，从这里唤醒最高优先级等待接收者。 |
| `xTasksWaitingToSend` | 队列已满 | 等待发送的任务的 `xEventListItem` | 接收者成功取走数据后，从这里唤醒最高优先级等待发送者。 |

这两条列表属于**某一个具体 Queue_t 对象**，不是全局就绪列表。列表内按等待任务的优先级排序，因此资源可用时，内核优先唤醒优先级更高的等待任务。

---

## 4.3 队列发送：`xQueueSend()` 到 `xQueueGenericSend()`

普通的 `xQueueSend()` 是“发送到队尾”的宏封装，最终调用：

```c
xQueueGenericSend( xQueue, pvItemToQueue, xTicksToWait, queueSEND_TO_BACK );
```

`pvItemToQueue` 只是待发送数据的源地址。队列会复制 `uxItemSize` 个字节到自己的缓冲区，**不会保存这个指针**。

### 4.3.1 发送成功时，队列本身如何变化

进入 `xQueueGenericSend()` 后，先在临界区判断：

```text
uxMessagesWaiting < uxLength
```

若成立，队列尚有空位，调用 `prvCopyDataToQueue()` 写入数据。对普通的 `xQueueSend()`，位置参数是 `queueSEND_TO_BACK`，执行顺序为：

```text
1. 将 pvItemToQueue 指向的 uxItemSize 字节复制到 pcWriteTo
2. pcWriteTo += uxItemSize，指向下一个可写槽位
3. 若 pcWriteTo 到达 pcTail，则回绕为 pcHead
4. uxMessagesWaiting 加 1
```

例如容量为 4，当前队列有 2 个 item，发送一次后：

```text
发送前：uxMessagesWaiting = 2，pcWriteTo 指向 item2
发送后：uxMessagesWaiting = 3，pcWriteTo 指向 item3
```

发送任务自己没有阻塞，仍在原优先级的就绪列表中；但队列从“空”或“有数据”变为“多一个 item”后，内核会检查 `xTasksWaitingToReceive`：

```text
xTasksWaitingToReceive 非空
    → 取出其中优先级最高的接收任务 R
    → R.xEventListItem 从 xTasksWaitingToReceive 移出
    → R.xStateListItem 从延时列表或挂起列表移出
    → R.xStateListItem 加入 pxReadyTasksLists[R优先级]
    → 若 R 优先级高于当前发送任务，抢占调度下请求切换
```

这一步只表示接收任务获得了“重新检查队列”的机会；数据仍保存在队列中，真正的取数由 R 后续恢复运行后完成。

### 4.3.2 三种写入位置

`xQueueGenericSend()` 的 `xCopyPosition` 决定 `prvCopyDataToQueue()` 如何写入。普通队列有三种方式：

| 写入方式 | 对应位置参数 | 数据写入和成员变化 |
| --- | --- | --- |
| 发送到队尾 | `queueSEND_TO_BACK` | 复制到 `pcWriteTo`，随后 `pcWriteTo` 向后移动并在 `pcTail` 回绕；`uxMessagesWaiting` 加 1。这是 `xQueueSend()` 的方式。 |
| 发送到队首 | `queueSEND_TO_FRONT` | 复制到当前 `pcReadFrom`，随后 `pcReadFrom` 向前移动一个 item 槽位并在 `pcHead` 前回绕到最后一个槽位；`uxMessagesWaiting` 加 1。下一次接收先将 `pcReadFrom` 前移，因此会先取到这次插入的数据。 |
| 覆盖写入 | `queueOVERWRITE` | 使用与“发送到队首”相同的写入位置，但只允许 `uxLength == 1`。队列原先已有 item 时，先使局部计数减 1，再在结尾加 1，因此 `uxMessagesWaiting` 保持为 1；新 item 覆盖旧 item。 |

前两种方式都要求队列未满。`queueOVERWRITE` 是长度为 1 的覆盖队列专用方式，因此即使 `uxMessagesWaiting == uxLength`，也允许写入。

### 4.3.3 队列已满：`xQueueGenericSend()` 的等待主线

若队列已满且不是 `queueOVERWRITE`，函数根据 `xTicksToWait` 分支：

```text
队列满
    ├── xTicksToWait == 0
    │   → 不阻塞，立即返回 errQUEUE_FULL
    │
    └── xTicksToWait > 0
        → 记录本次等待的起点 TimeOut_t
        → 暂停调度器并锁定队列
        → xTaskCheckForTimeOut() 判断还能否继续等待
```

首次发现队列满时，`vTaskInternalSetTimeOutState()` 将当前：

```text
xNumOfOverflows → xTimeOut.xOverflowCount
xTickCount      → xTimeOut.xTimeOnEntering
```

保存到 `TimeOut_t`。后续即使任务被唤醒后又抢不到空位，也会用同一个起点计算**总等待时间**，不会每次重试都重新获得完整的等待时长。

若 `xTaskCheckForTimeOut()` 返回“尚未超时”，内核还会再次确认队列确实仍满；这样可以避免“刚退出临界区，其他任务已接收数据”的竞态。只有确认仍满，才调用：

```c
vTaskPlaceOnEventList( &pxQueue->xTasksWaitingToSend, xTicksToWait );
```

其中当前发送任务 S 的两个 TCB 节点迁移为：

```text
S.xEventListItem
    → 按优先级插入 Queue_t.xTasksWaitingToSend
    → 表示等待“队列出现空位”这个事件

S.xStateListItem
    → 先从 pxReadyTasksLists[S优先级] 移出
    → 有限等待：按唤醒 tick 插入 pxDelayedTaskList 或 pxOverflowDelayedTaskList
    → 无限等待：在 INCLUDE_vTaskSuspend == 1 时插入 xSuspendedTaskList
```

因此有限等待期间：

```text
S.xStateListItem → 延时列表（负责超时）
S.xEventListItem → xTasksWaitingToSend（负责等空位）
```

真正的无限等待只有：

```text
xTicksToWait == portMAX_DELAY 且 INCLUDE_vTaskSuspend == 1
```

此时 `xTaskCheckForTimeOut()` 直接判定“未超时”，`xStateListItem` 进入 `xSuspendedTaskList`，不会因为 tick 推进而被释放。若 `INCLUDE_vTaskSuspend == 0`，`portMAX_DELAY` 只是最大有限等待 tick 数，最终仍可能超时。

### 4.3.4 被接收操作唤醒，或因超时返回失败

其他任务成功接收一个 item 后，队列出现空位，会从 `xTasksWaitingToSend` 取出优先级最高的发送任务 S：

```text
S.xEventListItem：xTasksWaitingToSend → 移出
S.xStateListItem：延时列表 / xSuspendedTaskList → pxReadyTasksLists[S优先级]
```

S 恢复运行后回到 `xQueueGenericSend()` 的循环开头，重新检查队列；确认有空位后才实际复制数据并返回 `pdPASS`。被事件唤醒不等于已经完成发送。

若没有空位，`xTaskCheckForTimeOut()` 通过以下路径作出判断：

```text
1. 若启用 INCLUDE_xTaskAbortDelay，且任务延时被 vTaskAbortDelay() 中止
   → 结束等待，函数返回“超时/结束等待”

2. 若 xTicksToWait == portMAX_DELAY 且 INCLUDE_vTaskSuspend == 1
   → 无限等待，返回“尚未超时”

3. 若 xNumOfOverflows 已不同于进入等待时记录的 xOverflowCount，
   且当前 xTickCount 又达到或超过 xTimeOnEntering
   → 至少经过完整一圈 tick，必然超时

4. 否则计算 xElapsedTime = 当前 xTickCount - 上次记录的进入 tick
   - xElapsedTime < 剩余 xTicksToWait：扣除已等待时间，更新 TimeOut_t，继续等待
   - xElapsedTime >= 剩余 xTicksToWait：等待耗尽，超时
```

第 3 条专门处理 tick 回绕多圈的情况，避免无符号减法再次得到较小的 `xElapsedTime`。超时由 SysTick 路径解除 S 的阻塞：

```text
S.xStateListItem：延时列表 → 就绪列表
S.xEventListItem：xTasksWaitingToSend → 移出
S 恢复运行 → xTaskCheckForTimeOut() 返回超时 → xQueueGenericSend() 返回 errQUEUE_FULL
```

---

## 4.4 队列接收：`xQueueReceive()` 与 `xQueuePeek()`

发送路径关心“有没有空位”，接收路径关心“有没有数据”。除方向相反外，`xQueueGenericReceive()` 的等待、超时和 TCB 列表迁移与 `4.3` 的发送路径基本对称。

```c
BaseType_t xQueueReceive(
    QueueHandle_t xQueue,
    void *pvBuffer,
    TickType_t xTicksToWait
);
```

`pvBuffer` 是接收目标地址。队列会将一个 item 的 `uxItemSize` 字节复制到这里，返回的不是内部缓冲区指针。

### 4.4.1 队列非空：真正接收如何改变 `Queue_t`

`xQueueGenericReceive()` 在临界区先读取 `uxMessagesWaiting`。只要：

```text
uxMessagesWaiting > 0
```

就调用 `prvCopyDataFromQueue()`：

```text
1. pcReadFrom += uxItemSize，移动到下一个待读 item
2. 若 pcReadFrom 到达 pcTail，则回绕为 pcHead
3. 从新的 pcReadFrom 复制 uxItemSize 字节到 pvBuffer
4. uxMessagesWaiting 减 1
```

`pcReadFrom` 记录的是“上一次读取位置”，所以读取必须先移动再复制。队列创建时它被设为最后一个槽位，也正是为了让第一次移动后落到 `pcHead`。

取走一个 item 后，队列出现一个空位。内核随即检查 `xTasksWaitingToSend`：

```text
xTasksWaitingToSend 非空
    → 取出其中优先级最高的发送任务 S
    → S.xEventListItem 从 xTasksWaitingToSend 移出
    → S.xStateListItem 从延时列表或 xSuspendedTaskList 移出
    → S.xStateListItem 加入 pxReadyTasksLists[S优先级]
    → 若 S 优先级高于当前接收任务，抢占调度下请求切换
```

被唤醒的 S 会重新进入发送函数开头检查队列；只有确认这个空位仍存在，才实际复制待发送数据。

### 4.4.2 队列为空：等待数据与超时

若 `uxMessagesWaiting == 0`，接收路径与“队列满时发送”的流程镜像对应：

```text
队列空
    ├── xTicksToWait == 0
    │   → 不阻塞，立即返回 errQUEUE_EMPTY
    │
    └── xTicksToWait > 0
        → 首次进入时记录 TimeOut_t
        → 暂停调度器并锁定队列
        → xTaskCheckForTimeOut() 检查还能否等待
        → 再次确认队列仍空后才真正阻塞
```

真正阻塞时调用：

```c
vTaskPlaceOnEventList( &pxQueue->xTasksWaitingToReceive, xTicksToWait );
```

当前接收任务 R 的两个节点分别进入：

```text
R.xEventListItem
    → xTasksWaitingToReceive
    → 表示等待“队列中出现数据”

R.xStateListItem
    → 从 pxReadyTasksLists[R优先级] 移出
    → 有限等待：进入 pxDelayedTaskList 或 pxOverflowDelayedTaskList
    → xTicksToWait == portMAX_DELAY 且 INCLUDE_vTaskSuspend == 1：进入 xSuspendedTaskList
```

其他任务发送数据后，会从 `xTasksWaitingToReceive` 唤醒一个最高优先级任务：

```text
R.xEventListItem：xTasksWaitingToReceive → 移出
R.xStateListItem：延时列表 / xSuspendedTaskList → pxReadyTasksLists[R优先级]
R 恢复运行 → 回到 xQueueGenericReceive() 开头 → 重新确认有数据后接收
```

若有限等待的 tick 先到，SysTick 将 R 同时从延时列表和 `xTasksWaitingToReceive` 移出，并放回就绪列表。R 再次进入 `xQueueGenericReceive()` 时，`xTaskCheckForTimeOut()` 使用与发送章节相同的延时中止、`portMAX_DELAY`、经过时间与 tick 回绕判断；若队列仍空，最终返回 `errQUEUE_EMPTY`。

> [!important] 即使 `xTaskCheckForTimeOut()` 已判断超时，源码仍会再检查一次队列是否为空。若刚好在超时边界已有数据进入队列，接收仍会回到循环尝试取出数据，而不是直接报告失败。

### 4.4.3 `xQueuePeek()`：读取第一个 item，但不移除它

`xQueuePeek()` 与 `xQueueReceive()` 的阻塞、超时、`xTasksWaitingToReceive` 迁移流程几乎相同。它们的主要区别发生在队列非空时：

```text
xQueueReceive()
    → 前移 pcReadFrom，复制 item
    → uxMessagesWaiting 减 1
    → 队列产生空位，唤醒一个 xTasksWaitingToSend 中的发送者

xQueuePeek()
    → 保存原 pcReadFrom
    → 临时前移 pcReadFrom，复制第一个 item 到 pvBuffer
    → 将 pcReadFrom 恢复为原值
    → uxMessagesWaiting 不变
    → 队列没有产生空位，不唤醒发送者
```

因此 `xQueuePeek()` 看的是当前队首 item，但不消费它：下次 `xQueueReceive()` 或再次 `xQueuePeek()` 看到的仍是同一个 item。

Peek 后队列仍然非空。源码会检查 `xTasksWaitingToReceive` 并可唤醒另一个等待接收的任务，因为数据仍可被其他接收者读取；而普通 `xQueueReceive()` 消费了一个 item，释放的是发送侧空位，所以检查的是 `xTasksWaitingToSend`。

```text
队列有 1 个 item，且 R1 调用 xQueuePeek()
    → R1 得到该 item 的副本
    → 队列仍有 1 个 item
    → pcReadFrom 与 uxMessagesWaiting 都保持原值
    → R2 之后调用 xQueueReceive() 仍能取走同一个 item
```

---

## 4.5 两个 API 的任务移动对照

```text
xQueueSend() 发现“满”
    当前发送者 S：就绪列表 → 延时列表 + xTasksWaitingToSend
    某接收者取走数据：S 从两条等待列表移除 → 就绪列表

xQueueReceive() 发现“空”
    当前接收者 R：就绪列表 → 延时列表 + xTasksWaitingToReceive
    某发送者写入数据：R 从两条等待列表移除 → 就绪列表
```

一句话总结：

> **队列的数据区解决“数据放在哪里”；`xTasksWaitingToSend/Receive` 解决“资源不可用时谁需要等”；TCB 的 `xStateListItem` 管超时和就绪状态，`xEventListItem` 管具体队列事件。**

---

# 5 互斥量：从零长度队列到优先级继承

FreeRTOS 没有为信号量和互斥量再实现一套完全独立的阻塞机制。它们复用 `Queue_t`、两条事件等待列表和队列的 take/give 路径；区别在于，它们不传递业务数据，而是把队列中的“消息数量”解释为资源计数。

```text
普通队列：uxMessagesWaiting = 当前有多少个可读取 item

二值信号量：uxMessagesWaiting = 0 或 1
    0 → 没有信号，take 必须等待
    1 → 有信号，take 可以成功

互斥量：uxMessagesWaiting = 0 或 1
    0 → 锁已被某个任务持有
    1 → 锁空闲，可被一个任务获取
```

## 5.1 信号量本质：`uxItemSize == 0` 的长度为 1 队列

以二值信号量为例，`semphr.h` 中的创建宏本质上是：

```c
xSemaphoreCreateBinary()
    → xQueueGenericCreate( 1, 0, queueQUEUE_TYPE_BINARY_SEMAPHORE )
```

也就是：

```text
uxLength   = 1     // 最多只记录一个信号
uxItemSize = 0     // 没有任何数据字节需要复制
```

因此创建时没有普通队列的数据缓冲区，也没有可供 `pcWriteTo`、`pcReadFrom` 移动的 item。`xQueueSemaphoreTake()` 和 `xQueueGenericSend()` 仍然在操作同一个 `Queue_t`，只是：

```text
xSemaphoreTake()
    → xQueueSemaphoreTake()
    → uxMessagesWaiting 从 1 减到 0

xSemaphoreGive()
    → xQueueGenericSend( ..., NULL, 0, queueSEND_TO_BACK )
    → uxMessagesWaiting 从 0 加到 1
```

这里没有 `memcpy()`；`pvItemToQueue` 可以是 `NULL`，因为 `uxItemSize == 0`。对信号量来说，“give 一个 item”真正表达的只是“资源可用”或“事件已发生”。

### 5.1.1 `Queue_t` 中哪些成员仍有意义

| 成员 | 普通队列 | 二值信号量/互斥量 |
| --- | --- | --- |
| `uxLength` | 可容纳的 item 数 | 通常为 `1`。 |
| `uxItemSize` | 每个 item 的字节数 | `0`，不保存业务数据。 |
| `uxMessagesWaiting` | 当前排队数据数 | 信号量/锁的计数；二值对象只会是 `0` 或 `1`。 |
| `pcHead`、`pcWriteTo` | 管理环形数据区 | 没有数据区，不参与数据拷贝；互斥量还复用 `pcHead` 作为类型标志。 |
| `u.xQueue.pcTail`、`pcReadFrom` | 管理普通队列读写指针 | 对零 item 的信号量/互斥量没有普通数据区含义。 |
| `xTasksWaitingToReceive` | 等待数据的接收任务 | 等待 take 成功的任务；对互斥量即等待获得锁的任务。 |
| `xTasksWaitingToSend` | 等待空位的发送任务 | 对一般二值信号量可理解为等待 give 的任务；互斥量正常 give 不等待，主要关心 take 等待列表。 |

`Queue_t.u` 是 union：普通队列使用 `u.xQueue` 的 `pcTail`、`pcReadFrom`；互斥量则使用同一块内存的：

```c
u.xSemaphore.xMutexHolder;        // 当前持有互斥量的 TCB
u.xSemaphore.uxRecursiveCallCount; // 递归互斥量的重入次数
```

源码还通过：

```c
#define uxQueueType pcHead
#define queueQUEUE_IS_MUTEX NULL
```

将 `pcHead == NULL` 作为“这是互斥量”的标记。普通队列的 `pcHead` 指向数据区，互斥量没有数据区，因此这个复用不会冲突。

## 5.2 二值信号量：同步与简单互斥

二值信号量常用作同步：一个执行者 give，另一个执行者 take。例如 ISR 或生产任务在事件完成后 give，等待任务 take 成功后继续运行。它也能充当简单的资源门闩：先 take 成功的任务进入临界区，其他任务在计数为 0 时阻塞，从而避免多个任务同时访问同一个外设。

```text
初始计数为 1

任务 A：take 成功，计数 1 → 0，开始访问 UART
任务 B：take 发现计数为 0，挂入 xTasksWaitingToReceive
任务 A：give，计数 0 → 1，唤醒 B
任务 B：再次运行后 take 成功，开始访问 UART
```

但二值信号量不记录“谁拿走了它”，也没有优先级继承：任何允许调用 give 的上下文都可以 give，它更适合事件同步。若对象明确代表“必须由持有者释放的共享资源”，应使用互斥量。

> [!warning] 二值信号量能实现简单互斥，但它没有所有权校验和优先级继承。保护可能导致高优先级任务被低优先级持有者拖住的外设或临界资源时，使用互斥量更合适。

## 5.3 优先级反转：为什么只靠二值信号量不够

设任务优先级满足 `H > M > L`，其中 L 已经获得了保护外设的二值信号量或互斥量：

```text
1. L 获得资源，开始访问外设
2. H 到来，尝试获得同一资源，发现资源不可用，进入阻塞态
3. M 到来。M 不需要该资源，但优先级高于 L
4. M 持续运行，L 无法获得 CPU 去完成外设访问并释放资源
5. H 明明最高优先级，却间接被 M 延后
```

这就是优先级反转：H 并非直接等待 M，而是等待 L 释放资源；M 却抢占了 L，使 H 的等待时间被无关的中优先级任务拉长。二值信号量本身不会改变 L 的优先级，因此不能消除这条链路。

## 5.4 互斥量：在零长度队列上加入“持有者”和优先级继承

互斥量同样创建为长度 1、item 大小 0 的队列：

```text
xSemaphoreCreateMutex()
    → xQueueCreateMutex( queueQUEUE_TYPE_MUTEX )
    → xQueueGenericCreate( 1, 0, ... )
    → prvInitialiseMutex()
```

`prvInitialiseMutex()` 在通用零长度队列初始化后额外执行：

```text
xMutexHolder = NULL
uxQueueType = queueQUEUE_IS_MUTEX   // 即 pcHead = NULL
uxRecursiveCallCount = 0
xQueueGenericSend( mutex, NULL, 0, queueSEND_TO_BACK )
```

最后一次 send 将 `uxMessagesWaiting` 置为 1，表示新互斥量初始空闲。

TCB 也只在 `configUSE_MUTEXES == 1` 时多出两个字段：

```text
uxBasePriority  → 任务创建时设置的基础优先级，不因继承而覆盖
uxMutexesHeld   → 当前任务持有的互斥量总数
```

`uxPriority` 则始终表示当前有效优先级：它可能等于 `uxBasePriority`，也可能在继承期间更高。

### 5.4.1 获取互斥量：`xSemaphoreTake()` 的源码路径

`xSemaphoreTake()` 仍然展开为：

```c
xQueueSemaphoreTake( xMutex, xTicksToWait );
```

互斥量空闲时，`uxMessagesWaiting == 1`：

```text
1. xQueueSemaphoreTake() 将 uxMessagesWaiting 从 1 减为 0
2. 检查对象类型发现它是 mutex
3. pvTaskIncrementMutexHeldCount() 令当前任务的 uxMutexesHeld 加 1
4. 当前任务 TCB 写入 u.xSemaphore.xMutexHolder
5. take 返回成功
```

若互斥量已被 L 持有，`uxMessagesWaiting == 0`，高优先级任务 H 允许等待：

```text
1. xQueueSemaphoreTake() 暂停调度器并锁定该 Queue_t
2. 调用 xTaskPriorityInherit( xMutexHolder )
3. 再调用 vTaskPlaceOnEventList( &xTasksWaitingToReceive, xTicksToWait )
4. H.xEventListItem → 该互斥量的 xTasksWaitingToReceive
5. H.xStateListItem → 延时列表；无限等待时可进入 xSuspendedTaskList
6. H 阻塞，调度器重新选择任务
```

`xTasksWaitingToReceive` 依旧按任务优先级排列，因此表头是最高优先级的等待者。

### 5.4.2 `xTaskPriorityInherit()`：把持有者抬到等待者优先级

在上述第 2 步，若：

```text
L.uxPriority < H.uxPriority
```

源码执行的核心动作是：

```text
1. 将 L.xEventListItem 的排序值更新为 H 的优先级对应值
   （前提是该 item 没有被其他事件机制占用）

2. 若 L 正在就绪列表中：
   从 pxReadyTasksLists[L旧优先级] 移出 L.xStateListItem
   L.uxPriority = H.uxPriority
   将 L.xStateListItem 插入 pxReadyTasksLists[H优先级]

3. 若 L 当前并不在就绪列表：
   直接更新 L.uxPriority；它未来进入就绪列表时会使用新优先级
```

回到优先级反转的例子：H 等待 L 的锁后，L 临时继承 H 的优先级，因此 L 可以抢占 M，尽快运行、完成临界区并释放资源。M 不再能无限延后 H。

### 5.4.3 释放互斥量：恢复资源计数，再考虑降级

持有者 L 调用：

```c
xSemaphoreGive( xMutex );
```

最终仍进入 `xQueueGenericSend()`，但因为 `uxItemSize == 0` 且对象是 mutex，`prvCopyDataToQueue()` 不复制数据，而是：

```text
1. xTaskPriorityDisinherit( xMutexHolder )
2. xMutexHolder = NULL
3. xQueueGenericSend() 将 uxMessagesWaiting 加为 1
4. 从 xTasksWaitingToReceive 取出最高优先级等待者 H
5. H：事件等待列表 + 延时/挂起列表 → 就绪列表
6. 若 H 优先级更高，请求调度
```

`xTaskPriorityDisinherit()` 总会先将 L 的 `uxMutexesHeld` 减 1。但它只有在 `uxMutexesHeld == 0` 时，才把 L 的当前优先级恢复到 `uxBasePriority` 并将 L 放回对应优先级就绪列表。

## 5.5 一个任务持有多个互斥量时的继承与限制

假设 L 同时持有互斥量 A、B，H 正等待 A，于是 L 继承 H 的优先级。FreeRTOS 通过 `uxMutexesHeld` 只维护“持有总数”，不为每个互斥量在 TCB 中建立一张完整的等待优先级表。

因此，当 L 先释放 A 时：

```text
L.uxMutexesHeld：2 → 1
L 仍持有 B
→ xTaskPriorityDisinherit() 不立刻恢复 L.uxBasePriority
→ L 保持已继承的较高优先级，直到最后一个互斥量也被释放
```

这种实现是有意的简化：它避免在每次释放一个互斥量时扫描 L 持有的所有其他互斥量及其等待者，但代价是 L 可能在已经释放 A 后，仍以较高优先级运行一段时间。

等待者超时时也有对应处理：如果 H 等待 A 超时，`vTaskPriorityDisinheritAfterTimeout()` 会计算：

```text
max( L.uxBasePriority, A 的剩余等待者中的最高优先级 )
```

但源码同样只在 L 恰好只持有一个互斥量（`uxMutexesHeld == 1`）时才实际降低优先级；若 L 仍持有多个互斥量，保持当前继承优先级，防止错误地忽略其他互斥量上的高优先级等待者。

> **FreeRTOS 的优先级继承解决“高优先级任务被中优先级任务无限干扰”的典型反转问题；但多互斥量场景使用的是保守的计数式简化，可能延后降级，而不是精确地逐把锁重新计算继承优先级。**
