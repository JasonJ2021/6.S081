# Lab: system calls

## System call tracing 
程序call trace(1<< SYS_fork)来调用system call trace;
我们需要修改xv6 kernel来打印一行信息当每个系统调用要返回的时候,trace应该能够跟踪父进程和它所有子进程，但不影响其他的进程。

    $ trace 32 grep hello README
    3: syscall read -> 1023
    3: syscall read -> 966
    3: syscall read -> 70
    3: syscall read -> 0
    $
    $ trace 2147483647 grep hello README
    4: syscall trace -> 0
    4: syscall exec -> 3
    4: syscall open -> 3
    4: syscall read -> 1023
    4: syscall read -> 966
    4: syscall read -> 70
    4: syscall read -> 0
    4: syscall close -> 0
    $
    $ grep hello README
    $
    $ trace 2 usertests forkforkfork
    usertests starting
    test forkforkfork: 407: syscall fork -> 408
    408: syscall fork -> 409
    409: syscall fork -> 410
    410: syscall fork -> 411
    409: syscall fork -> 412
    410: syscall fork -> 413
    409: syscall fork -> 414
    411: syscall fork -> 415
    ...
    $

hints:
- Add $U/_trace to UPROGS in Makefile
- 添加函数原型到user/user.h,a stub to user/usys.pl，和一个系统调用号到kernel/syscall.h
- 添加sys_trace()具体实现到kernel/sysproc.c.它可以获取参数到一个新的变量中(proc structure --- kernel/proc.h).获取调用参数的函数在kernel/syscall.c
- 修改fork()(kernel/proc.c)来把trace mask复制给子进程
- 修改syscall()(kernel/syscall.c)函数来打印输出消息，你需要添加一个调用名字数组。


## Sysinfo
实现一个系统调用:sysinfo,收集目前运行系统的信息。这个函数接受一个参数，a pointer to a struct sysinfo(kernel /sysinfo.h)
内核需要填满这个结构的区域，freemem代表目前空闲的内存比特数.nproc应该被设置为目前state不是UNUSED的进程数量

hints:
- Add $U/_sysinfotest to UPROGS in Makefile
- 添加函数原型到user/user.h,a stub to user/usys.pl，和一个系统调用号到kernel/syscall.h,添加sys_sysinfo()具体实现到kernel/sysproc.c.
  特别注意在user/h中
  >struct sysinfo;
    int sysinfo(struct sysinfo *);
- sysinfo需要复制一份结构sysinfo到用户空间中，sys_fstat()(kernel/sysfile.c) & filestat()(kernel/file.c)中由使用copyout()的例子
- 为了实现计算free memory的数量，实现一个函数kernel/kalloc.c
- 为了实现计算processes,添加一个函数到kernel/proc.c