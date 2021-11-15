# Lab : xv6 lazy page allocation

## Assignment 1: Eliminate allocation from sbrk() (easy)
删除sys_sbkr中对于内存的分配

    addr = myproc()->sz;
    // if(growproc(n) < 0)
    //   return -1;
    // ========================assignment1====================
    myproc()->sz = addr + n;
    return addr;

## Asignment 2 & 3
修改sbrk函数，要注意以下几点：
1. 能够处理n < 0 的情况

        if(n < 0){
            uint sz;
            sz = myproc()->sz;
            myproc()->sz = uvmdealloc(myproc()->pagetable,sz, sz + n);
        }
2. 如果一个虚拟内存地址超出了此时进程的大小，我们应该kill这个进程
3. fork()函数会拷贝一份相同的页表和物理内存空间到子进程。注意这时候如果遇到没有分配的内存空间，应该跳过。
4. 注意copyin 和 copyout.如果传给内核的地址或者内核要写的地址，映射的物理内存还没有被分配，此时要在内核中先分配再进行copyin&out operation
5. 如果kalloc()返回的地址为null,说明此时物理内存已经耗尽。应该kill进程
6. 注意处理低于栈指针的va.

踩的一些坑：
- copyin和out函数在kernel中运行，这里要区别return -1 和 p->killed = 1;
    当va0 >= p->sz || va0 < p->trapframe->sp 时候是用户进程出问题了，return -1 会返回处理这个进程。
    而在kalloc物理内存耗尽应该停止进程.p->killed =1;

      if(va0 >= p->sz || va0 < p->trapframe->sp){
              return -1;
          }
    
         ==================================
        if(pa0 == 0){
            p->killed = 1;
        }else{
            memset((void *)pa0, 0, PGSIZE);
            if (mappages(pagetable, va0, PGSIZE, pa0, PTE_W | PTE_R | PTE_U) != 0)
            {
            kfree((void *)pa0);
            p->killed = 1;
            }
        }
- 在usertrap(kernel/trap.c)中，先判断va0是否超出有效地址范围，注意出错的时候不能进行kalloc操作。

        uint64 va = r_stval();
            if (va >= p->sz || va < p->trapframe->sp)
            {
            p->killed = 1;
            }
            else
            {
            uint64 mem = (uint64)kalloc();
            if (mem == 0)
            {
                p->killed = 1;
            }
            else
            {
                memset((void *)mem, 0, PGSIZE);
                va = PGROUNDDOWN(va);
                if (mappages(p->pagetable, va, PGSIZE, mem, PTE_W | PTE_R | PTE_U) != 0)
                {
                kfree((void *)mem);
                p->killed = 1;
                }
            }
            }