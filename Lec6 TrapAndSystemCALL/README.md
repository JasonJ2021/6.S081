- [Lecture 5: system call](#lecture-5-system-call)
  - [Book Reading : Chapter 4 Traps and system calls](#book-reading--chapter-4-traps-and-system-calls)
    - [4.1 Risc-V trap machinery](#41-risc-v-trap-machinery)
    - [4.2 Traps from user space](#42-traps-from-user-space)
# Lecture 5: system call

## Book Reading : Chapter 4 Traps and system calls
有三种事件会导致CPU放下原来的指令流，强迫把控制转移到特定的代码。
- System call: 一个用户程序执行ecall来调用system call
- exception:一个指令(user or kernel)做了一些非法的行为，例如除0 or 使用无效的VA.
- interrupt:一个设备想要“说”他需要CPU的”关注“,例如一个磁盘硬件完成了一个read or write的指令

这本书用trap作为一个适用于前面三种情况的通用形式，通常来说trap之后会继续执行原来的程序流。一般的流程：trap 强迫控制转移到kernel -> kernel 保存寄存器和一些状态以便于之后继续执行指令 -> kernel执行适当的handler code(e.g. a system call impletentation or device driver) -> kernel 恢复原来的状态 ， 然后从trap返回 -> original code 从原来停止的地方继续执行


xv6在kernel中处理所有的trap
- 对于system call 来说很自然
- 对于interrupt 实现了隔离性：只有kernel能够使用device
- 对于exception , kernel 还可以kill offending program



xv6 陷阱处理有四个阶段：
- CPU执行的硬件行为
- 一些汇编指令为去到kernel C code作准备
- 一个 C function 来决定如何处理这个trap
- 系统调用 或者 设备驱动服务例程

我们虽然可以用同一个代码路径来处理所有的trap,但是把三种情况分开处理比较方便：用户空间的trap , 内核空间的trap , 时钟中断

处理trap的kernel code(C or 汇编)通常被称为handler,第一个handler 通常被写为汇编语言，有时候被称为*vector*

### 4.1 Risc-V trap machinery
每个RISC-V CPU都有一系列的由内核写入的控制寄存器用来告诉CPU如何处理trap.
下面是一些比较重要的
- stvec:内核会把trap handler的地址写到stvec中.RISC-V跳转到这个地址来进行处理操作
- sepv:当一个trap发生的时候，RISC-V会把原来的PC值村到sepv，因为马上PC就会被stvec中的值替代. *sret*(return from trap)复制sepc的值到pc.
- scause: RISC-V会放一个数字在这里来描述产生trap的原因
- sscratch:内核放一个值，在trap handler刚开始的时候派上用场
- sstatus: SIE bit in sstatus controls whether device interrupts are enabled.如果清除了SIE位，device interrupt会被延缓到它被设置。 SPP bit 描述一个trap是从user mode 还是 supervisor mode产生的，同时控制sret返回时候的模式

以上的一些寄存器只能在supervisor模式下访问，还有一些控制寄存器是只能在machine mode访问，用于一些特殊的timer interrupts

When it needs to force a trap,RISC-V硬件作以下操作(所有trap types除了timer interrupts 都一样)：
1. 如果trap 是device interrupt,而且sstattus SIE bit is clear，以下操作都不做
2. clear the SIE bit in sstatus
3. copy pc to sepc
4. 保存当前的mode in the SPP bit in sstatus
5. 设置 scause 来反映trap cause
6. 更改模式到supervisor
7. 复制stvec 到 pc
8. 开始在新的pc值执行指令

CPU不会切换到kernel pagetable,不会切换到kernel中的栈，不会保存除了pc以外的寄存器值

### 4.2 Traps from user space
The high-level path of a trap from user space is uservec(kernel/trampoline.s ) -> usertrap(kernel/trap.c)
返回的时候,usertrapret(kernel/trap.c) -> userret(kernel/trampoline.S)
4.1节提到了,when forcing a trap , RISC-V硬件不会切换页表，因此stvec中的地址在用户页表和内核页表中都要有映射
trampoline page解决了这个问题，它存在在user and kernel page中，包含了uservec(trap handling code that stvec points to )