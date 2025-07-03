---
author: AnicoderAndy
---

# 实验七 信号量

AnicoderAndy


<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=3 orderedList=false} -->

<!-- code_chunk_output -->

- [实验过程](#实验过程)
- [作业](#作业)
  - [1. 死锁](#1-死锁)
  - [2. 生产者消费者问题](#2-生产者消费者问题)
  - [3. 读者 - 写者锁问题](#3-读者---写者锁问题)
- [心得体会](#心得体会)

<!-- /code_chunk_output -->


## 实验过程

按照实验文档创建相关文件并且填充相关内容，kernel 文件夹中新增了 sem 的实现（`prt_sem_init.c` 包含初始化信号量的逻辑，`prt_sem.c` 包含对信号量的功能实现），在 include 目录下新建的 `prt_sem.h` 和 `prt_sem_external.h` 提供了相应的接口，此外还对部分之前的文件做了少量修改以兼容信号量。之后将新增文件纳入构建系统。在 `main.c` 中创建信号量并且在两个任务中对其进行测试（任务 2 具有更高的优先级）：

```c
void Test1TaskEntry() {
    PRT_Printf("task 1 run ...\n");
    PRT_SemPost(sem_sync);
    U32 cnt = 5;
    while (cnt > 0) {
        // PRT_TaskDelay(200);
        PRT_Printf("task 1 run ...\n");
        cnt--;
    }
}

void Test2TaskEntry() {
    PRT_Printf("task 2 run ...\n");

    PRT_SemPend(sem_sync, OS_WAIT_FOREVER);
    U32 cnt = 5;
    while (cnt > 0) {
        // PRT_TaskDelay(100);
        PRT_Printf("task 2 run ...\n");
        cnt--;
    }
}
```

任务 2 因为具有更高的优先级而先运行，其运行到 `PRT_SemPend` 时会被挂起，任务 1 运行到 `PRT_SemPost` 时会唤醒任务 2。最终的输出结果如下：

```
task 2 run ...
task 1 run ...
task 2 run ...
task 2 run ...
task 2 run ...
task 2 run ...
task 2 run ...
task 1 run ...
task 1 run ...
task 1 run ...
task 1 run ...
task 1 run ...
```

## 作业

本次作业要求用信号量模拟三个常见的并发问题。

### 1. 死锁

#### 问题描述

任务 1 和任务 2 分别持有信号量 A 和 B，互相等待对方释放信号量。代码如下：

```c
#include "prt_tick.h"
#include "prt_typedef.h"
#include "prt_task.h"
#include "prt_sem.h"

extern U32 PRT_Printf(const char* format, ...);
extern void PRT_UartInit(void);
extern U32 OsHwiInit(void);
extern void CoreTimerInit(void);
extern U32 OsActivate(void);
extern U32 OsTskInit(void);
extern U32 OsSemInit(void);

// 两个互斥二元信号量
static SemHandle semA;
static SemHandle semB;

// 简单的忙等待延时（仅用于模拟，不建议生产用）
static void BusyDelay(U32 loops)
{
    volatile U32 i;
    for (i = 0; i < loops; i++);
}

void DeadlockTask1(void)
{
    PRT_Printf("T1: Try requiring semA...\n");
    PRT_SemPend(semA, OS_WAIT_FOREVER);
    PRT_Printf("T1: Required semA, working...\n");
    BusyDelay(1000000);

    PRT_Printf("T1: Try requiring semB...\n");
    // 此时若 T2 已持有 semB，则 T1 阻塞
    PRT_SemPend(semB, OS_WAIT_FOREVER);

    // 到不了这里
    PRT_Printf("T1: Required semB, continue...\n");
    PRT_SemPost(semB);
    PRT_SemPost(semA);
}

void DeadlockTask2(void)
{
    PRT_Printf("T2: Try requiring semB...\n");
    PRT_SemPend(semB, OS_WAIT_FOREVER);
    PRT_Printf("T2: Required semB, working...\n");
    BusyDelay(1000000);

    PRT_Printf("T2: Try requiring semA...\n");
    // 此时若 T1 已持有 semA，则 T2 阻塞
    PRT_SemPend(semA, OS_WAIT_FOREVER);

    // 到不了这里
    PRT_Printf("T2: Required semA, continue...");
    PRT_SemPost(semA);
    PRT_SemPost(semB);
}

S32 main(void)
{
    OsHwiInit();
    CoreTimerInit();
    OsTskInit();
    OsSemInit();
    PRT_UartInit();

    // 创建两个二元互斥信号量，初始值都为 1
    if (PRT_SemCreate(1, &semA) != OS_OK ||
        PRT_SemCreate(1, &semB) != OS_OK) {
        return -1;
    }

    struct TskInitParam param = {0};
    TskHandle th1, th2;

    // 创建 Task1，优先级 30
    param.taskEntry = (TskEntryFunc)DeadlockTask1;
    param.taskPrio  = 30;
    param.stackSize = 0x1000;
    if (PRT_TaskCreate(&th1, &param) != OS_OK ||
        PRT_TaskResume(th1) != OS_OK) {
        return -1;
    }

    // 创建 Task2，优先级 30
    param.taskEntry = (TskEntryFunc)DeadlockTask2;
    param.taskPrio  = 30;
    if (PRT_TaskCreate(&th2, &param) != OS_OK ||
        PRT_TaskResume(th2) != OS_OK) {
        return -1;
    }

    // 启动调度
    OsActivate();

    return 0;
}

```

#### 运行结果

```
T1: Try requiring semA...
T1: Required semA, working...
T2: Try requiring semB...
T2: Required semB, working...
T1: Try requiring semB...
T2: Try requiring semA...
```

两个任务都因为获取不到对方的锁而阻塞，导致死锁。

### 2. 生产者消费者问题

#### 问题描述

我们需要模拟多个生产者向缓冲区生产数据，多个消费者从缓冲区消费数据的实例。用 `semEmpty` 记录剩余槽数量，`semFull` 记录已用槽数量，`semMutex` 用于互斥访问缓冲区。具体代码如下：

```c
// producer_consumer.c
#include "prt_tick.h"
#include "prt_typedef.h"
#include "prt_task.h"
#include "prt_sem.h"

extern U32 PRT_Printf(const char* format, ...);
extern void PRT_UartInit(void);
extern U32 OsHwiInit(void);
extern void CoreTimerInit(void);
extern U32 OsActivate(void);
extern U32 OsTskInit(void);
extern U32 OsSemInit(void);

#define BUFFER_SIZE 5

// 环形缓冲区
static int buffer[BUFFER_SIZE];
static U32 in = 0;
static U32 out = 0;

// 信号量句柄
static SemHandle semEmpty; // 表示可用空槽的计数
static SemHandle semFull;  // 表示已填充槽的计数
static SemHandle semMutex; // 保护缓冲区访问的二元互斥信号量

// 简单忙等待，用于模拟生产/消费耗时
static void BusyDelay(U32 loops) {
    volatile U32 i;
    for (i = 0; i < loops; i++);
}

void ProducerTask(void) {
    int item;
    while (1) {
        // 生成一个“产品”（这里只用序号模拟）
        static int nextItem = 0;
        item = nextItem++;
        PRT_SemPend(semEmpty, OS_WAIT_FOREVER);
        PRT_SemPend(semMutex, OS_WAIT_FOREVER);

        // 放入缓冲区
        buffer[in] = item;
        PRT_Printf("Producer: produced item %d at index %u\n", item, in);
        in = (in + 1) % BUFFER_SIZE;

        PRT_SemPost(semMutex);
        PRT_SemPost(semFull);

        BusyDelay(2000000); // 模拟生产耗时
    }
}

void ConsumerTask(void) {
    int item;
    while (1) {
        PRT_SemPend(semFull, OS_WAIT_FOREVER);
        PRT_SemPend(semMutex, OS_WAIT_FOREVER);

        // 从缓冲区取出
        item = buffer[out];
        PRT_Printf("Consumer: consumed item %d from index %u\n", item, out);
        out = (out + 1) % BUFFER_SIZE;

        PRT_SemPost(semMutex);
        PRT_SemPost(semEmpty);

        BusyDelay(3000000); // 模拟消费耗时
    }
}

S32 main(void) {
    OsHwiInit();
    CoreTimerInit();
    OsTskInit();
    OsSemInit();
    PRT_UartInit();

    // 创建三个信号量：
    // semEmpty 初始值 BUFFER_SIZE（所有槽都空）
    // semFull  初始值 0           （无产品可取）
    // semMutex 初始值 1           （互斥访问缓冲区）
    if (PRT_SemCreate(BUFFER_SIZE, &semEmpty) != OS_OK ||
        PRT_SemCreate(0, &semFull) != OS_OK ||
        PRT_SemCreate(1, &semMutex) != OS_OK) {
        PRT_Printf("Failed to create semaphores\n");
        return -1;
    }

    struct TskInitParam param = {0};
    TskHandle prodHandle[3], consHandle[3];

    // 创建 3 个 Producer 任务，优先级 30
    for (U32 i = 0; i < 3; i++) {
        param.taskEntry = (TskEntryFunc)ProducerTask;
        param.taskPrio = 30;
        param.stackSize = 0x1000;
        if (PRT_TaskCreate(&prodHandle[i], &param) != OS_OK ||
            PRT_TaskResume(prodHandle[i]) != OS_OK) {
            PRT_Printf("Failed to start ProducerTask\n");
            return -1;
        }
    }

    // 创建 3 个 Consumer 任务，优先级 30
    for (U32 i = 0; i < 3; i++) {
        param.taskEntry = (TskEntryFunc)ConsumerTask;
        param.taskPrio = 30;
        if (PRT_TaskCreate(&consHandle[i], &param) != OS_OK ||
            PRT_TaskResume(consHandle[i]) != OS_OK) {
            PRT_Printf("Failed to start ConsumerTask\n");
            return -1;
        }
    }

    // 启动调度
    OsActivate();

    return 0;
}
```

#### 运行结果

```
Producer: produced item 0 at index 0
Producer: produced item 1 at index 1
Producer: produced item 2 at index 2
Consumer: consumed item 0 from index 0
Consumer: consumed item 1 from index 1
Consumer: consumed item 2 from index 2
Producer: produced item 3 at index 3
Producer: produced item 4 at index 4
Producer: produced item 5 at index 0
Consumer: consumed item 3 from index 3
Consumer: consumed item 4 from index 4
Consumer: consumed item 5 from index 0
Producer: produced item 6 at index 1
Producer: produced item 7 at index 2
Producer: produced item 8 at index 3
Producer: produced item 9 at index 4
Producer: produced item 10 at index 0
Consumer: consumed item 6 from index 1
Consumer: consumed item 7 from index 2
Consumer: consumed item 8 from index 3
...
```

可以看到程序正常运行，消费者永远只会取到生产者生产的产品，且不会出现缓冲区溢出或空槽取值的情况。

### 3. 读者 - 写者锁问题

#### 问题描述

读者 - 写者锁问题是一个经典的并发问题，要求多个读者可以同时读取数据，但在写者写入数据时，所有读者和其他写者都必须等待。我们用信号量来实现这个功能。代码如下：

```c
// reader_writer.c
#include "prt_tick.h"
#include "prt_typedef.h"
#include "prt_task.h"
#include "prt_sem.h"

extern U32 PRT_Printf(const char* format, ...);
extern void PRT_UartInit(void);
extern U32 OsHwiInit(void);
extern void CoreTimerInit(void);
extern U32 OsActivate(void);
extern U32 OsTskInit(void);
extern U32 OsSemInit(void);

#define NUM_READERS 3
#define NUM_WRITERS 2

// 信号量句柄
static SemHandle semReadCountMutex; // 保护 readersCount 互斥信号量
static SemHandle semResource;       // 控制对资源的独占访问信号量
static SemHandle semQueue;          // 写者优先：控制进入队列的信号量

static int readersCount = 0;

// 模拟读操作耗时
static void BusyDelay(U32 loops) {
    volatile U32 i;
    for (i = 0; i < loops; i++);
}

void ReaderTask(void* param) {
    int id = (int)(uintptr_t)param;
    while (1) {
        // 写者优先：在读者进入前先获取队列信号量，避免饥饿写者
        PRT_SemPend(semQueue, OS_WAIT_FOREVER);
        // 读者增加时保护 readersCount
        PRT_SemPend(semReadCountMutex, OS_WAIT_FOREVER);
        readersCount++;
        if (readersCount == 1) {
            // 第一个读者占用资源，阻止写者
            PRT_SemPend(semResource, OS_WAIT_FOREVER);
        }
        // 允许下一个读者或写者进入排队
        PRT_SemPost(semQueue);
        PRT_SemPost(semReadCountMutex);

        // —— 临界区 (读取共享资源) ——
        PRT_Printf("Reader %d: reading (active readers = %d)\n", id,
                   readersCount);
        BusyDelay(1000000);

        // 退出时更新 readersCount
        PRT_SemPend(semReadCountMutex, OS_WAIT_FOREVER);
        PRT_Printf("Reader %d: done reading\n", id);
        readersCount--;
        if (readersCount == 0) {
            // 最后一个读者释放资源，允许写者
            PRT_SemPost(semResource);
        }
        PRT_SemPost(semReadCountMutex);

        // 模拟读者间隔
        BusyDelay(2000000);
    }
}

void WriterTask(void* param) {
    int id = (int)(uintptr_t)param;
    while (1) {
        // 写者优先：先进入队列
        PRT_SemPend(semQueue, OS_WAIT_FOREVER);
        // 再独占资源
        PRT_SemPend(semResource, OS_WAIT_FOREVER);
        // 允许下一个等待者进入队列
        PRT_SemPost(semQueue);

        // —— 临界区 (写入共享资源) ——
        PRT_Printf("Writer %d: writing\n", id);
        BusyDelay(1500000);
        PRT_Printf("Writer %d: done writing\n", id);

        // 释放资源
        PRT_SemPost(semResource);

        // 模拟写者间隔
        BusyDelay(3000000);
    }
}

S32 main(void) {
    OsHwiInit();
    CoreTimerInit();
    OsTskInit();
    OsSemInit();
    PRT_UartInit();

    // 创建信号量
    // semReadCountMutex: 二元互斥，初始值 1
    // semResource:  控制对共享资源的独占访问，初始值 1
    // semQueue:     保证写者优先的队列锁，初始值 1
    if (PRT_SemCreate(1, &semReadCountMutex) != OS_OK ||
        PRT_SemCreate(1, &semResource) != OS_OK ||
        PRT_SemCreate(1, &semQueue) != OS_OK) {
        PRT_Printf("Failed to create semaphores\n");
        return -1;
    }

    struct TskInitParam param = {0};
    TskHandle th;
    int i;

    // 创建多个 Reader 任务
    for (i = 0; i < NUM_READERS; i++) {
        param.taskEntry = (TskEntryFunc)ReaderTask;
        param.taskPrio = 30; // 读者优先级，可调整
        param.stackSize = 0x1000;
        param.args[0] = (uintptr_t)i; // 传递读者 ID
        if (PRT_TaskCreate(&th, &param) != OS_OK ||
            PRT_TaskResume(th) != OS_OK) {
            PRT_Printf("Failed to start Reader %d\n", i);
            return -1;
        }
    }

    // 创建多个 Writer 任务
    for (i = 0; i < NUM_WRITERS; i++) {
        param.taskEntry = (TskEntryFunc)WriterTask;
        param.taskPrio = 30;
        param.args[0] = (uintptr_t)i;
        if (PRT_TaskCreate(&th, &param) != OS_OK ||
            PRT_TaskResume(th) != OS_OK) {
            PRT_Printf("Failed to start Writer %d\n", i);
            return -1;
        }
    }

    // 启动调度
    OsActivate();
    return 0;
}
```

#### 运行结果

```
Reader 0: reading (active readers = 1)
Reader 1: reading (active readers = 2)
Reader 2: reading (active readers = 3)
Reader 2: done reading
Reader 0: done reading
Reader 1: done reading
Writer 0: writing
Writer 0: done writing
Writer 1: writing
Writer 1: done writing
Reader 1: reading (active readers = 1)
Reader 2: reading (active readers = 2)
...
```

保证了读者可以并发读取，写者独占访问资源。读者和写者之间不会发生死锁。

## 心得体会

本次实验我首先通过实现信号量的基本功能，了解了信号量的基本原理和使用方法。通过创建和使用信号量，我能够在多个任务之间进行同步和互斥操作，从而有效地管理共享资源。

我还体会到了信号量的强大和灵活性，能够有效地解决多任务之间的同步和互斥问题。通过实现死锁、生产者消费者问题和读者 - 写者锁问题，我对信号量的使用有了更深入的理解。同时，我也意识到在设计并发程序时需要仔细考虑任务之间的关系，以避免潜在的死锁和资源竞争问题。