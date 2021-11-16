# Lab: Copy-on-Write Fork for xv6
COW的目标是推迟分配和复制物理内存给子进程到它真正需要这个PA页的时候。
COW使得释放一个物理内存页更加小心，因为我们需要判断还有没有COWF指向这个页，如果没有一个引用指向这个页，那么就可以释放内存。有点类似CPP的smart pointer
我们可以把COW理解为共享物理内存页。

plan of attack:
1. 修改uvmcopy只给子进程分配相同的pagetable,然后修改父子进程的pte都为只读
2. 修改usetrap，使其能够识别page fault.当page fault产生在COW PAGE中，用kalloc分配一个新的页，把旧的COW page 内容复制到new page ,然后给new page权限加上PTE_W
3. 确保只有在最后一个对物理内存的引用被取消之后才能释放这页物理内存
    在kalloc的时候使reference = 1;fork()时候加1；在一个进程丢弃这个page的时候减一
    可以创建一个int数组作为计数器。我们需要设计一个索引方案。
4. 修改copyout,方案和usertrap处理page fault相同

小提示：
- 不能使用lazy allocation的副本
- 可以使用PTE中RSW(reserved for software)来记录这个page是否为COW PAGE
- kernel/riscv.h中的一些宏定义会派上用处
- 如果COW pagefault出现，然后又没有free memory这时候应该杀死进程

## 具体实现
修改fork,只复制pagetable，不分配新的物理内存.这里需要注意的是我们把父子页表的Pte都设置为PTE_C & ~PTE_W.每fork一次都要使Physical PA的引用值++;

    @@ -319,12 +319,13 @@ uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
        if((*pte & PTE_V) == 0)
        panic("uvmcopy: page not present");
        pa = PTE2PA(*pte);
    +    *pte &= ~PTE_W;
    +    *pte |= PTE_C;
        flags = PTE_FLAGS(*pte);
    -    if((mem = kalloc()) == 0)
    -      goto err;
    -    memmove(mem, (char*)pa, PGSIZE);
    -    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
    -      kfree(mem);
    +    
    +    incRef(pa);
    +
    +    if(mappages(new, i, PGSIZE, pa, flags) != 0){
        goto err;
        }
    }

在usertrap(kernel/trap.c)中添加对page fault的处理。因为在cowfork只是把页的PTE_W清除，所以我们这里只会产生store pagefault(对应代码是15)

    @@ -67,7 +67,12 @@ usertrap(void)
        syscall();
    } else if((which_dev = devintr()) != 0){
        // ok
    -  } else {
    +  }else if(r_scause() == 15){
    +      if(PfHandler(p->pagetable , r_stval()) != 0){
    +        p->killed = 1;
    +      }
    +  }
    +  else {


PfHandler(我把它放在vm.c中，方便使用walk)函数的实现,主要实现的功能是分配新的物理内存给page fault的地址。每次在pagetable中丢弃对一个physical page的mapping ,都要尝试free原来的那个物理内存页。

    +int PfHandler(pagetable_t pagetable , uint64 va){
    +  if(va >= MAXVA){
    +    return -1;
    +  }
    +  pte_t *pte = walk(pagetable , va , 0);
    +  if(pte == 0){
    +    return -1;
    +  }
    +
    +  uint64 pa;
    +  if(*pte & PTE_C){
    +      uint flags = PTE_FLAGS(*pte);
    +      char *mem;
    +      pa = PTE2PA(*pte);
    +      flags &= ~PTE_C;
    +      flags |= PTE_W;
    +      //allocate a new page 
    +      mem = kalloc(); 
    +      if(mem == 0){
    +        return -1;
    +      }
    +      if(mem == 0){
    +        return -1;
    +      }
    +      //copy the old page
    +      memmove(mem , (char *)pa , PGSIZE);
    +      //unmap the old PP
    +      
    +
    +      kfree((void *)pa);
    +      //map the new page
    +      *pte = PA2PTE(mem) | flags;
    +  }
    +  return 0;
    +}


copyout因为是在kernel space运行，所以如果产生page fault不会进入usertrap中，应该添加与PfHandler相同的代码。但是这里注意要把pa 指向新分配的物理内存mem！(找了好久这个错误)

在kalloc.c中，我们需要修改kfree和kalloc.在这里我们需要记录引用值，从PHYSTOP - end 值我们可以推算出可以建立一个大小为32739的int数组，记录每一个PP的引用值。 
> int index[32739];

因为它是一个共享全局变量，所以我们需要对它加锁

    +struct {
    +  struct spinlock lock;
    +  int index[32739];
    +}refindex;


锁的初始化

    @@ -27,16 +31,23 @@ void
    kinit()
    {
    initlock(&kmem.lock, "kmem");
    +  initlock(&refindex.lock,"refindex");
    freerange(end, (void*)PHYSTOP);
    }

在freerange中可以对index数组进行初始化
void

    freerange(void *pa_start, void *pa_end)
    {
    char *p;
    -  p = (char*)PGROUNDUP((uint64)pa_start);
    -  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    -    kfree(p);
    +  p = (char*)PGROUNDUP((uint64)pa_start); //0 -> 4095
    +  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE){
    +      acquire(&refindex.lock);
    +      refindex.index[(p-(char *)pa_start)/PGSIZE] = 1;
    +      release(&refindex.lock);
    +      kfree(p);
    +  }
    +  
    }


在kfree中，只有引用值为0的时候，才可以继续下面的操作释放物理内存，否则返回。注意这里decRef和判断引用值为0需要用一把锁。

    @@ -50,11 +61,19 @@ kfree(void *pa)
    
    if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
        panic("kfree");
    +  
    +  acquire(&refindex.lock);
    +  refindex.index[((char *)pa-(char*)PGROUNDUP((uint64)end))/PGSIZE] -= 1;
    
    +  if(refindex.index[((char *)pa-(char*)PGROUNDUP((uint64)end))/PGSIZE] != 0 ){
    +    release(&refindex.lock);
    +    return;
    +  }
    +  release(&refindex.lock);


在kalloc中，对physical Page Reference进行初始化。

    @@ -74,9 +93,28 @@ kalloc(void)
    r = kmem.freelist;
    if(r)
        kmem.freelist = r->next;
    +    
    release(&kmem.lock);
    -
    +  if(r){
    +      acquire(&refindex.lock);
    +      refindex.index[((char *)r-(char*)PGROUNDUP((uint64)end))/PGSIZE] = 1;
    +      release(&refindex.lock);
    +  }

对了，在riscv.h中添加PTE_C，来判断是否为COW PAGE

    +#define PTE_C (1L << 8)
