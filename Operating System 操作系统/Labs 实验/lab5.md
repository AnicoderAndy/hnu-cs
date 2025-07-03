---
author: AnicoderAndy
---

# 实验五 时钟 Tick

AnicoderAndy

请注意，作业在实验过程中已经完成，[点击这里](#作业)跳转到作业部分。

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->

<!-- code_chunk_output -->

- [实验过程及作业](#实验过程及作业)
  - [GIC 控制逻辑（**作业**）](#gic-控制逻辑作业)
  - [处理中断](#处理中断)
  - [处理时钟中断](#处理时钟中断)
  - [测试中断](#测试中断)
- [心得体会](#心得体会)

<!-- /code_chunk_output -->

## 实验过程及作业
### GIC 控制逻辑（**作业**）
首先了解 ARM 架构下的中断控制逻辑，其主要由 GIC Distributor 和 GIC CPU Interface 两组寄存器控制，Distributor 负责中断的分发和优先级仲裁，而 CPU Interface 处理与 CPU 核心的交互。

我们首先在 `src/bsp/hwi_init.c` 当中完成初始化 GIC 的逻辑：

```c
#include "prt_typedef.h"
#include "os_attr_armv8_external.h"

#define OS_GIC_VER 2

#define GIC_DIST_BASE 0x08000000
#define GIC_CPU_BASE 0x08010000

#define GICD_CTLR (GIC_DIST_BASE + 0x0000U)
#define GICD_TYPER (GIC_DIST_BASE + 0x0004U)
#define GICD_IIDR (GIC_DIST_BASE + 0x0008U)
#define GICD_IGROUPRn (GIC_DIST_BASE + 0x0080U)
#define GICD_ISENABLERn (GIC_DIST_BASE + 0x0100U)
#define GICD_ICENABLERn (GIC_DIST_BASE + 0x0180U)
#define GICD_ISPENDRn (GIC_DIST_BASE + 0x0200U)
#define GICD_ICPENDRn (GIC_DIST_BASE + 0x0280U)
#define GICD_ISACTIVERn (GIC_DIST_BASE + 0x0300U)
#define GICD_ICACTIVERn (GIC_DIST_BASE + 0x0380U)
#define GICD_IPRIORITYn (GIC_DIST_BASE + 0x0400U)
#define GICD_ICFGR (GIC_DIST_BASE + 0x0C00U)

#define GICD_CTLR_ENABLE 1  /* Enable GICD */
#define GICD_CTLR_DISABLE 0 /* Disable GICD */
#define GICD_ISENABLER_SIZE 32
#define GICD_ICENABLER_SIZE 32
#define GICD_ICPENDR_SIZE 32
#define GICD_IPRIORITY_SIZE 4
#define GICD_IPRIORITY_BITS 8
#define GICD_ICFGR_SIZE 16
#define GICD_ICFGR_BITS 2

#define GICC_CTLR (GIC_CPU_BASE + 0x0000U)
#define GICC_PMR (GIC_CPU_BASE + 0x0004U)
#define GICC_BPR (GIC_CPU_BASE + 0x0008U)
#define IAR_MASK 0x3FFU
#define GICC_IAR (GIC_CPU_BASE + 0xc)
#define GICC_EOIR (GIC_CPU_BASE + 0x10)
#define GICD_SGIR (GIC_DIST_BASE + 0xf00)

#define BIT(n) (1 << (n))

#define GICC_CTLR_ENABLEGRP0 BIT(0)
#define GICC_CTLR_ENABLEGRP1 BIT(1)
#define GICC_CTLR_FIQBYPDISGRP0 BIT(5)
#define GICC_CTLR_IRQBYPDISGRP0 BIT(6)
#define GICC_CTLR_FIQBYPDISGRP1 BIT(7)
#define GICC_CTLR_IRQBYPDISGRP1 BIT(8)

#define GICC_CTLR_ENABLE_MASK (GICC_CTLR_ENABLEGRP0 | GICC_CTLR_ENABLEGRP1)

#define GICC_CTLR_BYPASS_MASK                                                  \
    (GICC_CTLR_FIQBYPDISGRP0 | GICC_CTLR_IRQBYPDISGRP0 |                       \
     GICC_CTLR_FIQBYPDISGRP1 | GICC_CTLR_IRQBYPDISGRP1)

#define GICC_CTLR_ENABLE 1
#define GICC_CTLR_DISABLE 0
// Priority Mask Register. interrupt priority filter, Higher priority
// corresponds to a lower Priority field value.
#define GICC_PMR_PRIO_LOW 0xff
// The register defines the point at which the priority value fields split into
// two parts, the group priority field and the subpriority field. The group
// priority field is used to determine interrupt preemption. NO GROUP.
#define GICC_BPR_NO_GROUP 0x00

#define GIC_REG_READ(addr) (*(volatile U32*)((uintptr_t)(addr)))
#define GIC_REG_WRITE(addr, data)                                              \
    (*(volatile U32*)((uintptr_t)(addr)) = (U32)(data))

// src/arch/drv/gic/prt_gic_init.c
/*
 * 描述：去使能（禁用）指定中断
 */
OS_SEC_L4_TEXT void OsGicDisableInt(U32 intId) {
    // Interrupt Clear-Enable Registers
    // 每个寄存器管理 32 个中断，除以 32 得到寄存器偏移
    uintptr_t reg_offset = intId / GICD_ICENABLER_SIZE;
    // 计算对应中断位
    uint32_t bit_position = intId % GICD_ICENABLER_SIZE;
    // 写入对应的寄存器，将对应中断位置 1，即禁用该中断
    GIC_REG_WRITE(GICD_ICENABLERn + reg_offset * sizeof(U32),
                  1 << bit_position);
}

/*
 * 描述：使能指定中断
 */
OS_SEC_L4_TEXT void OsGicEnableInt(U32 intId) {
    // Interrupt Set-Enable Registers
    uintptr_t reg_offset = intId / GICD_ISENABLER_SIZE;
    // 计算对应中断位
    uint32_t bit_position = intId % GICD_ISENABLER_SIZE;
    // 写入对应的寄存器，将对应中断位置 1，即使能该中断
    GIC_REG_WRITE(GICD_ISENABLERn + reg_offset * sizeof(U32),
                  1 << bit_position);
}

// 直接清除中断的 pending 状态
OS_SEC_L4_TEXT void OsGicClearIntPending(uint32_t interrupt) {
    // Interrupt Clear-Pending state of an interrupt Registers
    GIC_REG_WRITE(GICD_ICPENDRn + (interrupt / GICD_ICPENDR_SIZE) * sizeof(U32),
                  1 << (interrupt % GICD_ICPENDR_SIZE));
}

// 设置中断号为 interrupt 的中断的优先级为 priority
OS_SEC_L4_TEXT void OsGicIntSetPriority(uint32_t interrupt, uint32_t priority) {
    uint32_t shift = (interrupt % GICD_IPRIORITY_SIZE) * GICD_IPRIORITY_BITS;
    volatile uint32_t* addr =
        ((volatile U32*)(uintptr_t)(GICD_IPRIORITYn +
                                    (interrupt / GICD_IPRIORITY_SIZE) *
                                        sizeof(U32)));
    uint32_t value = GIC_REG_READ(addr);
    value &= ~(0xff << shift); // 每个中断占 8 位，所以掩码为 0xFF
    value |= priority << shift;
    GIC_REG_WRITE(addr, value);
}

// 设置中断号为 interrupt 的中断的属性为 config
OS_SEC_L4_TEXT void OsGicIntSetConfig(uint32_t interrupt, uint32_t config) {
    uint32_t shift = (interrupt % GICD_ICFGR_SIZE) * GICD_ICFGR_BITS;
    volatile uint32_t* addr =
        ((volatile U32*)(uintptr_t)(GICD_ICFGR + (interrupt / GICD_ICFGR_SIZE) *
                                                     sizeof(U32)));
    uint32_t value = GIC_REG_READ(addr);
    value &= ~(0x03 << shift);
    value |= config << shift;
    GIC_REG_WRITE(addr, value);
}

/*
 * 描述：中断确认
 */
OS_SEC_L4_TEXT U32 OsGicIntAcknowledge(void) {
    // reads this register to obtain the interrupt ID of the signaled interrupt.
    // This read acts as an acknowledge for the interrupt.
    U32 value = GIC_REG_READ(GICC_IAR);
    return value;
}

/*
 * 描述：标记中断完成，清除相应中断位
 */
OS_SEC_L4_TEXT void OsGicIntClear(U32 value) {
    // A processor writes to this register to inform the CPU interface either:
    // • that it has completed the processing of the specified interrupt
    // • in a GICv2 implementation, when the appropriate GICC_CTLR.EOImode bit
    // is set to 1, to indicate that the interface should perform priority drop
    // for the specified interrupt.
    GIC_REG_WRITE(GICC_EOIR, value);
}

U32 OsHwiInit(void) {

    // 初始化 Gicv2 的 distributor 和 cpu interface
    // 禁用 distributor 和 cpu interface 后进行相应配置
    GIC_REG_WRITE(GICD_CTLR, GICD_CTLR_DISABLE);
    GIC_REG_WRITE(GICC_CTLR, GICC_CTLR_DISABLE);
    // 将 PMR 设置为最低优先级，也就是说可以处理所有中断（请注意 0xff 是最低）
    GIC_REG_WRITE(GICC_PMR, GICC_PMR_PRIO_LOW);
    // 禁用优先级分组
    GIC_REG_WRITE(GICC_BPR, GICC_BPR_NO_GROUP);

    // 启用 distributor 和 cpu interface
    GIC_REG_WRITE(GICD_CTLR, GICD_CTLR_ENABLE);
    GIC_REG_WRITE(GICC_CTLR, GICC_CTLR_ENABLE);

    return OS_OK;
}
```

<a id="作业"></a>
文档要求我们实现 `OsGicDisableInt` 和 `OsGicEnableInt` 函数，查阅文档可以知道，为了使能“禁用”某个中断号，或者使能“启用”某个中断号，分别是对 `Interrupt Clear-Enable Registers` 和 `Interrupt Set-Enable Registers` 做操作。这两个寄存器组中每个寄存器的每一位都代表一个中断号，而每个寄存器只有 32 位，我们只需要将中断号除以 32 即可找到需要操作哪个寄存器，对 32 取模即可找到需要操作寄存器的哪一位。具体实现*在上面的代码中已经包含*。

文件中引用的 `os_attr_armv8_external.h` 定义了 hwi 所需要的接口，可以在 [实验文档网站](https://os2024lab.readthedocs.io/zh-cn/latest/_static/os_attr_armv8_external.h) 下载。

下面在 `prt_config.h` 中设置中断周期：

```c
/* Tick 中断时间间隔，tick 处理时间不能超过 1/OS_TICK_PER_SECOND(s) */
#define OS_TICK_PER_SECOND                              1000
```

下面在 `src/bsp/timer.c` 中对定时和中断做配置：
```c
#include "os_cpu_armv8.h"
#include "prt_config.h"
#include "prt_typedef.h"

U64 g_timerFrequency;
extern void OsGicIntSetConfig(uint32_t interrupt, uint32_t config);
extern void OsGicIntSetPriority(uint32_t interrupt, uint32_t priority);
extern void OsGicEnableInt(U32 intId);
extern void OsGicClearIntPending(uint32_t interrupt);

void CoreTimerInit(void) {
    // 配置中断控制器
    OsGicIntSetConfig(30, 0);   // 配置为电平触发
    OsGicIntSetPriority(30, 0); // 优先级为 0
    OsGicClearIntPending(30);   // 清除中断 pending
    OsGicEnableInt(30);         // 启用中断

    // 配置定时器（这里会用汇编指令给 g_timerFrequency 赋值为 CPU 时钟频率，单位 Hz）
    OS_EMBED_ASM("MRS %0, CNTFRQ_EL0" : "=r"(g_timerFrequency) : : "memory",
                 "cc"); // 读取时钟频率

    U32 cfgMask = 0x0;
    // 一次 OS Tick 中断所需要的 CPU 时钟周期数
    U64 cycle = g_timerFrequency / OS_TICK_PER_SECOND;

    OS_EMBED_ASM("MSR CNTP_CTL_EL0, %0" : : "r"(cfgMask) : "memory");
    PRT_ISB();
    OS_EMBED_ASM("MSR CNTP_TVAL_EL0, %0" : : "r"(cycle) : "memory",
                 "cc"); // 设置中断周期

    cfgMask = 0x1;
    OS_EMBED_ASM("MSR CNTP_CTL_EL0, %0" : : "r"(
        cfgMask) : "memory"); // 启用定时器 enable=1, imask=0, istatus= 0,
    OS_EMBED_ASM("MSR DAIFCLR, #2");
}
```

### 处理中断

在异常处理表中将 `prt_vector.S` 中的 `EXC_HANDLE 5 OsExcDispatch` 改为 `EXC_HANDLE 5 OsHwiDispatcher`，表明我们将对 IRQ 类型的异常（即中断）使用 OsHwiDispatcher 处理。

在这一文件中加入 `OsHwiDispatcher` 的实现：

```armasm
    .globl OsHwiDispatcher
    .type OsHwiDispatcher, @function
    .align 4
OsHwiDispatcher:
    mrs    x5, esr_el1
    mrs    x4, far_el1
    mrs    x3, spsr_el1
    mrs    x2, elr_el1
    stp    x4, x5, [sp,#-16]!
    stp    x2, x3, [sp,#-16]!

    mov    x0, x1  // 异常类型 0~15，参见异常向量表
    mov    x1, sp  // 异常时寄存器信息，通过栈及其 sp 指针提供
    bl     OsHwiDispatch

    ldp    x2, x3, [sp],#16
    add    sp, sp, #16        // 跳过 far, esr, HCR_EL2.TRVM==1 的时候，EL1 不能写 far, esr
    msr    spsr_el1, x3
    msr    elr_el1, x2
    dsb    sy
    isb

    RESTORE_EXC_REGS // 恢复上下文

    eret //从异常返回
```

然后在 `prt_exc.c` 实现 `OsTickDispatch`，顺手再实现 `PRT_HwiLock` 等几个常用的关于全局可屏蔽中断的功能：

```c
extern void OsTickDispatcher(void);
extern U32 OsGicIntAcknowledge(void);
extern void OsGicIntClear(U32 value);

OS_SEC_ALW_INLINE INLINE void OsHwiHandleActive(U32 irqNum) {
    switch (irqNum) {
    case 30:
        OsTickDispatcher();
        // PRT_Printf(".");
        break;
    default:
        break;
    }
}

// src/arch/cpu/armv8/common/hwi/prt_hwi.c OsHwiDispatch(),OsHwiDispatchHandle()
/*
 * 描述：中断处理入口，调用处外部已关中断
 */
OS_SEC_L0_TEXT void OsHwiDispatch(
    U32 excType,
    struct ExcRegInfo* excRegs) // src/arch/cpu/armv8/common/hwi/prt_hwi.c
{
    // 中断确认，相当于 OsHwiNumGet()
    U32 value = OsGicIntAcknowledge();
    U32 irq_num = value & 0x1ff;
    U32 core_num = value & 0xe00;

    OsHwiHandleActive(irq_num);

    // 清除中断，相当于 OsHwiClear(hwiNum);
    OsGicIntClear(irq_num | core_num);
}

/*
 * 描述：开启全局可屏蔽中断。
 */
OS_SEC_L0_TEXT uintptr_t
PRT_HwiUnLock(void) // src/arch/cpu/armv8/common/hwi/prt_hwi.c
{
    uintptr_t state = 0;

    OS_EMBED_ASM(
        "mrs %0, DAIF      \n"
        "msr DAIFClr, %1   \n" : "=r"(state) : "i"(DAIF_IRQ_BIT) : "memory",
        "cc");

    return state & INT_MASK;
}

/*
 * 描述：关闭全局可屏蔽中断。
 */
OS_SEC_L0_TEXT uintptr_t
PRT_HwiLock(void) // src/arch/cpu/armv8/common/hwi/prt_hwi.c
{
    uintptr_t state = 0;
    OS_EMBED_ASM(
        "mrs %0, DAIF      \n"
        "msr DAIFSet, %1   \n" : "=r"(state) : "i"(DAIF_IRQ_BIT) : "memory",
        "cc");
    return state & INT_MASK;
}

/*
 * 描述：恢复原中断状态寄存器。
 */
OS_SEC_L0_TEXT void
PRT_HwiRestore(uintptr_t intSave) // src/arch/cpu/armv8/common/hwi/prt_hwi.c
{
    if ((intSave & INT_MASK) == 0) {
        OS_EMBED_ASM("msr DAIFClr, %0\n" : : "i"(DAIF_IRQ_BIT) : "memory",
                     "cc");
    } else {
        OS_EMBED_ASM("msr DAIFSet, %0\n" : : "i"(DAIF_IRQ_BIT) : "memory",
                     "cc");
    }
    return;
}
```

### 处理时钟中断

我们在 `prt_tick.h` 头文件中声明我们需要实现的接口：

```c
#ifndef PRT_TICK_H
#define PRT_TICK_H

#include "prt_typedef.h"

extern U64 PRT_TickGetCount(void);

#endif /* PRT_TICK_H */
```

下面我们实现 `tick` 实际处理的逻辑，新建 `src/kernel/tick/prt_tick.c`：

```c
#include "os_attr_armv8_external.h"
#include "os_cpu_armv8_external.h"
#include "prt_config.h"
#include "prt_typedef.h"

extern U64 g_timerFrequency;

/* Tick 计数 */
OS_SEC_BSS U64 g_uniTicks; // src/core/kernel/sys/prt_sys.c

/*
 * 描述：Tick 中断的处理函数。扫描任务超时链表、扫描超时软件定时器、扫描 TSKMON 等。
 */
OS_SEC_TEXT void OsTickDispatcher(void) {
    uintptr_t intSave;

    intSave = OsIntLock();
    g_uniTicks++;
    OsIntRestore(intSave);

    U64 cycle = g_timerFrequency / OS_TICK_PER_SECOND;
    OS_EMBED_ASM("MSR CNTP_TVAL_EL0, %0" : : "r"(cycle) : "memory",
                 "cc"); // 设置中断周期
}

/*
 * 描述：获取当前的 tick 计数
 */
OS_SEC_L2_TEXT U64 PRT_TickGetCount(void) // src/core/kernel/sys/prt_sys_time.c
{
    return g_uniTicks;
}
```

并且在 `os_cpu_armv8_external.h` 头文件中定义用到的功能：

```c
#include <stdint.h>
#ifndef OS_CPU_ARMV8_EXTERNAL_H
#define OS_CPU_ARMV8_EXTERNAL_H

extern uintptr_t PRT_HwiUnLock(void);
extern uintptr_t PRT_HwiLock(void);
extern void PRT_HwiRestore(uintptr_t intSave);

#define OsIntUnLock() PRT_HwiUnLock()
#define OsIntLock()   PRT_HwiLock()
#define OsIntRestore(intSave) PRT_HwiRestore(intSave)

#endif
```

### 测试中断

最后修改 `main.c` 做测试：

```c
#include "prt_tick.h"
#include "prt_typedef.h"
// #define PRINT_NO_FIFO

extern U32 PRT_Printf(const char* format, ...);
extern void PRT_UartInit(void);
extern void CoreTimerInit(void);
extern U32 OsHwiInit(void);

U64 delay_time = 10000;

S32 main(void) {
    PRT_UartInit();
    OsHwiInit();
    CoreTimerInit();

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

    for (int i = 0; i < 10; i++) {

        U32 tick = PRT_TickGetCount();
        PRT_Printf("[%d] current tick: %u\n", i, tick);

        // delay
        int delay_time = 50000000; // 根据自己机器计算能力不同调整该值
        while (delay_time > 0) {
            PRT_TickGetCount(); // 消耗时间，防止延时代码被编译器优化
            delay_time--;
        }
    }

    while (1)
        ;
    return 0;
}
```

构建并启动模拟器：

```
[0] current tick: 1
[1] current tick: 631
[2] current tick: 1271
[3] current tick: 1907
[4] current tick: 2540
[5] current tick: 3173
[6] current tick: 3813
[7] current tick: 4449
[8] current tick: 5087
[9] current tick: 5730
```

说明 Tick 功能正常运作。

## 心得体会

通过本次时钟 Tick 实验，我对 ARM 架构的中断机制和操作系统的中断管理有了更深刻的理解，我了解了 ARM 中断机制：通过配置 Distributor 和 CPU Interface 寄存器，理解了中断优先级仲裁、中断屏蔽等硬件机制；从定时器周期触发中断，到异常向量跳转至 OsHwiDispatcher，再到确认中断号和标记完成，理解了硬件中断到软件处理的完整链路。

实验过程中遇到了 `uintptr_t` 未定义的问题，通过引入 `stdint.h` 解决了此问题。尽管调整头文件引用顺序可能也可以解决此问题，但在头文件中事先定义未定义的类型减少头文件的依赖关系才是更好的选择。