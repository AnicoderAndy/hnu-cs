---
author: AnicoderAndy
---

# 实验六 任务调度

AnicoderAndy


<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=3 orderedList=false} -->

<!-- code_chunk_output -->

- [实验过程](#实验过程)
- [作业](#作业)
  - [添加时间片接口](#添加时间片接口)
  - [添加“移动到队尾”功能](#添加移动到队尾功能)
  - [添加中断处理](#添加中断处理)
  - [测试](#测试)
- [心得体会](#心得体会)

<!-- /code_chunk_output -->

## 实验过程

根据实验文档，新建对应文件并且填充对应代码（包括链表数据结构的接口 `list_types.h` 和实现 `prt_list_external.h`，任务管理内外部的相关接口声明，任务切换陷入内核的代码、内核中 `sched` 和 ` task` 的接口和实现等等）。完成后，修改 `main.c` 利用刚刚实现的接口利用优先级调度算法调度两个任务。

```c
#include "prt_tick.h"
#include "prt_typedef.h"
#include "prt_task.h"
// #define PRINT_NO_FIFO

extern U32 PRT_Printf(const char* format, ...);
extern void PRT_UartInit(void);
extern void CoreTimerInit(void);
extern U32 OsHwiInit(void);
extern U32 OsActivate(void);
extern U32 OsTskInit(void);

void Test1TaskEntry() {
    PRT_Printf("task 1 run ...\n");

    U32 cnt = 5;
    while (cnt > 0) {
        // PRT_TaskDelay(200);
        PRT_Printf("task 1 run ...\n");
        cnt--;
    }
}

void Test2TaskEntry() {
    PRT_Printf("task 2 run ...\n");

    U32 cnt = 5;
    while (cnt > 0) {
        // PRT_TaskDelay(100);
        PRT_Printf("task 2 run ...\n");
        cnt--;
    }
}

S32 main(void) {
    // Initialize GIC
    OsHwiInit();
    // Enable timer
    CoreTimerInit();
    // Initialize Task Manager
    OsTskInit();
    // Initialize UART
    PRT_UartInit();

    const char banner[] = {
        "        _      _ ___     _           _             _           _      "
        "\n  _ __ (_)_ _ (_) __|  _| |___ _ _  | |__ _  _    /_\\  _ _  __| |_ "
        " _ \n | '  \\| | ' \\| | _| || | / -_) '_| | '_ \\ || |  / _ \\| ' "
        "\\/ _` | || |\n |_|_|_|_|_||_|_|___\\_,_|_\\___|_|   |_.__/\\_, | /_/ "
        "\\_\\_||_\\__,_|\\_, |\n                                          "
        "|__/                   |__/ "};

    PRT_Printf("%s\n", banner);

    PRT_Printf(
        "ctr-a h: print help of qemu emulator. ctr-a x: quit emulator.\n\n");

    U32 ret;
    struct TskInitParam param = {0};

    // task 1
    // param.stackAddr = 0;
    param.taskEntry = (TskEntryFunc)Test1TaskEntry;
    param.taskPrio = 35;
    // param.name = "Test1Task";
    param.stackSize = 0x1000; // 固定4096，参见prt_task_init.c的OsMemAllocAlign

    TskHandle tskHandle1;
    ret = PRT_TaskCreate(&tskHandle1, &param);
    if (ret) {
        return ret;
    }

    ret = PRT_TaskResume(tskHandle1);
    if (ret) {
        return ret;
    }

    // task 2
    // param.stackAddr = 0;
    param.taskEntry = (TskEntryFunc)Test2TaskEntry;
    param.taskPrio = 30;
    // param.name = "Test2Task";
    param.stackSize = 0x1000; // 固定4096，参见prt_task_init.c的OsMemAllocAlign

    TskHandle tskHandle2;
    ret = PRT_TaskCreate(&tskHandle2, &param);
    if (ret) {
        return ret;
    }

    ret = PRT_TaskResume(tskHandle2);
    if (ret) {
        return ret;
    }

    // 启动调度
    OsActivate();
    return 0;
}
```

启动模拟器，正常运行高优先级的任务（任务 2）：

```
task 2 run ...
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
task 1 run ...
```

## 作业

本次作业要求实现 RR 调度算法。

### 添加时间片接口

首先给每个任务添加一个时间片的属性，并且在初始化时将其规定为一个默认值，在 `prt_config.h` 中定义这个值：

```c
#define OS_TSK_DEFAULT_TIMESLICE 5
```

在任务管理器外部接口 `prt_task_external.h` 中添加这个属性：

```c
struct TagTskCb {
    // ...实验文档中提供的原代码...
    uintptr_t args[4];

    /* 任务的剩余时间片 */
    U32 timeSlice;

#if (defined(OS_OPTION_TASK_INFO))
    // ...实验文档中提供的原代码...
};
```

在初始化任务的相关代码 `prt_task_init.c` 中添加给这一属性赋值的逻辑：

```c
OS_SEC_L4_TEXT U32 OsTaskCreateOnly(TskHandle* taskPid,
                                    struct TskInitParam* initParam) {
    // ...实验文档中提供的原代码...
    *taskPid = taskId;
    taskCb->timeSlice = OS_TSK_DEFAULT_TIMESLICE;
    OsIntRestore(intSave);
    return OS_OK;
}
```

### 添加“移动到队尾”功能

我们在 `prt_task.c` 中实现一个将任务移动到就绪队列尾部的接口，把当前运行中的任务移动到队尾后再次调用优先级调度程序，会调用在队列相对头部的的同优先级任务，从而达到时间片轮转的效果。

```c
OS_SEC_L0_TEXT void OsTskMoveToBack(struct TagTskCb* task) {
    struct TagOsRunQue* rq = &g_runQueue;
    TSK_STATUS_CLEAR(task, OS_TSK_RUNNING);
    TSK_STATUS_SET(task, OS_TSK_READY);

    OS_TSK_DE_QUE(rq, task, 0);
    OS_TSK_EN_QUE(rq, task, 0);
    OsTskHighestSet();

    return;
}
```

将这个函数的声明添加在任务管理的接口文件 `prt_task_external.h` 中：

```c
extern void OsTskMoveToBack(struct TagTskCb *task);
```

### 添加中断处理

在处理中断的文件 `prt_tick.c` 中，添加对于时间片处理的逻辑，首先引用任务管理相关头文件，并且声明用到的外部函数 `OsGicIntClear`：

```c
#include "prt_task_external.h"
#include "prt_asm_cpu_external.h"

extern void OsGicIntClear(U32 value);
```

为 Tick 中断（中断号为 30）添加以下逻辑：

```c
OS_SEC_TEXT void OsTickDispatcher(void) {
    uintptr_t intSave;

    intSave = OsIntLock();
    g_uniTicks++;
    OsIntRestore(intSave);

    U64 cycle = g_timerFrequency / OS_TICK_PER_SECOND;
    OS_EMBED_ASM("MSR CNTP_TVAL_EL0, %0" : : "r"(cycle) : "memory",
                 "cc"); // 设置中断周期

    // 以下为新增的内容：
    // 调度任务前先禁用中断
    intSave = OsIntLock();
    if ((RUNNING_TASK != NULL) && (RUNNING_TASK->taskPid != IDLE_TASK_ID)) {
        if (RUNNING_TASK->timeSlice > 0) {
            RUNNING_TASK->timeSlice--;
        }

        if (RUNNING_TASK->timeSlice == 0) {
            // 通知 GIC 本次中断已经完成，避免新任务运行时不触发中断
            OsGicIntClear(30);
            // 将旧任务的时间片复原
            RUNNING_TASK->timeSlice = OS_TSK_DEFAULT_TIMESLICE;
            // 将旧任务移动到队尾
            OsTskMoveToBack(RUNNING_TASK);
            // 申请调度任务
            OsTskSchedule();
        }
    }
    OsIntRestore(intSave);
}
```

### 测试

至此 RR 调度算法已经完成。目前实现的版本会优先运行高优先级任务，对于同级别任务会进行时间片轮转。

为了测试正确性，将刚刚测试代码中的循环次数设置为 5000 次，并且将两个任务的优先级设为相同。下面展示的是新的任务 1 代码：

```c
void Test1TaskEntry() {
    PRT_Printf("task 1 run ...\n");

    U32 cnt = 5000;
    while (cnt > 0) {
        PRT_TickGetCount();
        if (cnt % 10 == 0)
            PRT_Printf("task 1: %d\n", 5000 - cnt);
        cnt--;
    }
}
```

下面截取运行结果的部分以展示正确性（为了使切换更频繁，在运行下面代码前将时默认时间片改为了 2）：

```
...
task 2: 4480
task 2: 4490
task 2: 4500
task 2: 4510
task 2: 45 1: 4500
task 1: 4510
task 1: 4520
task 1: 4530
task 1: 4540
task 1: 4550
...
task 1: 4920
task 1: 4930
task 1: 4940
task 1: 4950
task 1: 4960
20
task 2: 4530
task 2: 4540
task 2: 4550
task 2: 4560
task 2: 4570
task 2: 4580
...
task 2: 4970
task 2: 4980
task 2: 4990
task 1: 4970
task 1: 4980
task 1: 4990
```

## 心得体会

本次实验是此课程实验开设以来最复杂的一次，需要对理论知识（进程切换和进程状态等）和实际应用（实验文档中提供的各种接口）有充分的理解。我通过这次实验，深入理解了任务调度的实现原理和细节，掌握了时间片轮转调度算法的设计与实现。

本次实验过程中遇到的最大的问题是切换到新进程时，中断不再触发。通过仔细研究进程切换时的调用逻辑，发现切换到新进程时原来的中断并没有被清除，导致新进程无法触发中断。通过在 `prt_tick.c` 中添加 `OsGicIntClear` 解决了这个问题。