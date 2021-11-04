# Lab3 : page tables

## 1. Print a page table (easy)
这个任务要求我们打印一个页表

    void vmprint(pagetable_t pagetable, int level)
    {
      if (level == 1)
      {
        printf("page table %p\n", pagetable);
      }
      int next = level + 1;
      for (int i = 0; i < 512; i++)
      {
        pte_t pte = pagetable[i];
        if (pte & PTE_V)
        {
          for (int j = 1; j < level; j++)
          {
            printf(".. ");
          }
          printf("..%d: pte %p pa %p\n", i, (pagetable_t)pte, PTE2PA(pte));
          if (next <= 3)
          {
            uint64 child = PTE2PA(pte);
            vmprint((pagetable_t)child, next);
          }
        }
      }
    }
## 2. A kernel page table per process (hard)
我们在内核中无法直接引用用户进程的指针，因为kernel pagetable 中没有用户指针VA->PA的mapping,这个assignment就是让我们能够直接引用指针。
- 修改kernel让每个进程使用属于自己的kernel pagetable
- 修改struct proc，创建一个保存kernel page的地方
  修改scheduler ,让每次进程切换的时候也要切换kernel pagetable

首先在proc结构中添加属于进程独立的kernelpage
      struct proc {
        struct spinlock lock;
      ......
      ++pagetable_t kpagetable;
      };

      
添加一个函数让proc_kernelpage能够添加va->pa的mapping

    void
    proc_kvmmap(pagetable_t pagetable , uint64 va, uint64 pa, uint64 sz, int perm)
    {
      if(mappages(pagetable, va, sz, pa, perm) != 0)
        panic("prockvmmap");
    }

这里创建的proc_kernelpage和之前的kernelpage不太一样，
它没有映射CLINT，CLINET所处的位置是用户虚拟内存的最高位置
我们在allocproc()函数中添加create_kpagetable
    pagetable_t create_kpagetable()
    {
      pagetable_t pagetable = uvmcreate();
      // uart registers
      proc_kvmmap(pagetable , UART0, UART0, PGSIZE, PTE_R | PTE_W);

      // virtio mmio disk interface
      proc_kvmmap(pagetable , VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

      // // CLINT
      // proc_kvmmap(pagetable , CLINT, CLINT, 0x10000, PTE_R | PTE_W);

      // PLIC
      proc_kvmmap(pagetable , PLIC, PLIC, 0x400000, PTE_R | PTE_W);

      // map kernel text executable and read-only.
      proc_kvmmap(pagetable , KERNBASE, KERNBASE, (uint64)etext - KERNBASE, PTE_R | PTE_X);

      // map kernel data and the physical RAM we'll make use of.
      proc_kvmmap(pagetable , (uint64)etext, (uint64)etext, PHYSTOP - (uint64)etext, PTE_R | PTE_W);

      // map the trampoline for trap entry/exit to
      // the highest virtual address in the kernel.
      proc_kvmmap(pagetable , TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
      return pagetable;
    }


发现这里proc_kernelpage没有对kernel stack的映射

    uint64 va = KSTACK((int)(p - proc));
      uint64 pa = kvmpa(va);
      proc_kvmmap(p->kpagetable, va , pa, PGSIZE, PTE_R | PTE_W );
      p->kstack = va;


我们还需要有一个函数在进程被删除的时候释放kernelpage，注意这里只需要释放页表，不需要释放level3 pagetable所映射的物理内存

    void freepage(pagetable_t pagetable)
    {
      // there are 2^9 = 512 PTEs in a page table.
      for (int i = 0; i < 512; i++)
      {
        pte_t pte = pagetable[i];

        if (pte & PTE_V)
        {
          pagetable[i] = 0;
          if ((pte & (PTE_R | PTE_W | PTE_X)) == 0)
          {
            // this PTE points to a lower-level page table.
            uint64 child = PTE2PA(pte);
            freepage((pagetable_t)child);
          }
        }
      }
      kfree((void *)pagetable);
    }

在每次进程切换的时候，我们都要切换相应的内核页表：

        w_satp(MAKE_SATP(p->kpagetable));
        sfence_vma();

        swtch(&c->context, &p->context);
        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
        kvminithart();
        found = 1;




## 3 . Simplify copyin/copyinstr (hard)
这个任务要求我们为proc_kernelpage添加对用户虚拟内存的映射

    int
    p2kpmapping(pagetable_t pagetable, pagetable_t kpagetable, uint start , uint64 sz)
    {
      pte_t *pte;
      uint64 pa, i;
      uint flags;
      start = PGROUNDUP(start);
      for(i = start; i < sz; i += PGSIZE){
        if((pte = walk(pagetable, i, 0)) == 0)
          panic("uvmcopy: pte should exist");
        if((*pte & PTE_V) == 0)
          panic("uvmcopy: page not present");
        pa = PTE2PA(*pte);
        flags = PTE_FLAGS(*pte) & (~PTE_U); //! ! !
        if(mappages(kpagetable, i, PGSIZE, (uint64)pa, flags) != 0){
          goto err;
        }
      }
      return 0;

    err:
      uvmunmap(kpagetable, 0, i / PGSIZE, 1);
      return -1;
    }


    uint64
    kpgdemapping(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
    {
      if(newsz >= oldsz)
        return oldsz;

      if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
        int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
        uvmunmap(pagetable, PGROUNDUP(newsz), npages, 0);
      }

      return newsz;
    }

首先需要两个函数，添加映射和解除映射

其次在fork() , exec(),growproc()三个会改变进程虚拟内存映射的函数中添加对proc_kernelpage的改变

    fork(){
        ...
        if(p2kpmapping(np->pagetable,np->kpagetable, 0 , np->sz) < 0){
        freeproc(np);
        release(&np->lock);
        return -1;
      }
    }

    exec(){
        ...
        kpgdemapping(p->kpagetable,p->sz , 0 );
        p2kpmapping(p->pagetable , p->kpagetable ,0 , sz);
    }

    growproc(){
        ...
        // n > 0 用 p2kpmapping
        if(n > 0){
          p2kpmapping(p->pagetable , p->kpagetable , p->sz , p->sz + n);
        }
        //  n < 0 用uvmdealloc
        if(n < 0){
          kpgdemapping(p->kpagetable , p->sz , p->sz + n);
        }
    }

最后不要忘了为userinit添加映射

    void
    userinit(void)
    {
      struct proc *p;

      p = allocproc();
      initproc = p;
      
      // allocate one user page and copy init's instructions
      // and data into it.
      uvminit(p->pagetable, initcode, sizeof(initcode));
      p->sz = PGSIZE;

      // prepare for the very first "return" from kernel to user.
      p->trapframe->epc = 0;      // user program counter
      p->trapframe->sp = PGSIZE;  // user stack pointer

      safestrcpy(p->name, "initcode", sizeof(p->name));
      p->cwd = namei("/");

      p->state = RUNNABLE;

      p2kpmapping(p->pagetable , p->kpagetable , 0 , p->sz);
      
      release(&p->lock);
    }
