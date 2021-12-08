- [Lecture 12 : Sleep & Wakeup](#lecture-12--sleep--wakeup)
  - [Book Reading](#book-reading)
    - [7.5 : Sleep & Wake up](#75--sleep--wake-up)
    - [7.6 : Code Sleep & Wake up](#76--code-sleep--wake-up)
    - [7.7 : Code Pipes](#77--code-pipes)
    - [7.8 ： Code Wait exit and kill](#78--code-wait-exit-and-kill)
# Lecture 12 : Sleep & Wakeup

Preparation: Read remainder of "Scheduling", and corresponding parts of kernel/proc.c, kernel/sleeplock.c

## Book Reading 

### 7.5 : Sleep & Wake up
调度和锁实现了进程之间的隔离，但无法实现进程之间的交互，xv6使用sleep & wake up 机制来实现，这个机制允许一个进程进入到休眠状态(e.g. 等待一个IO事件)，之后另外一个进程可以使用wake up 唤醒休眠的进程。

下面以信号量为例，阐述s & w machanisms

    struct semaphore{
        struct spinlock lock;
        int count;
    }

    void V(struct semaphore *s){
        acquire(&s->lock);
        s->count += 1;
        release(&s->lock);
    }

    void P(struct semaphore *s){
        while(!s->count);
        acquire(&s->lock);
        s->count -= 1;
        release(&s->lock);
    }

上面的实现中，P函数的循环会占用大量的CPU资源，所以我们要实现一种机制，当s->count == 0 时,P先进入休眠状态，让出CPU，等待s->count > 0 ;
我们可以修改上面的函数

    void V(struct semaphore *s){
        acquire(&s->lock);
        s->count += 1;
        wakeup(s);
        release(&s->lock);
    }

    void P(struct semaphore *s){
        while(!s->count){
    a-------- sleep(s);
        }
        acquire(&s->lock);
        s->count -= 1;
        release(&s->lock);
    }
sleep(*c)函数可以使一个进程在c channel上休眠，wakeup函数会唤醒在这个channel上的所有进程

但是上面的实现会造成wake up loss;例如，在CPU 1 上运行到a代码还没有执行的时候，CPU 2 上运行wakeup,这时候CPU 1 上再执行sleep()这个进程不能被唤醒。即使这时候s->count > 0;

那么加一个锁可以么？

    void V(struct semaphore *s){
        acquire(&s->lock);
        s->count += 1;
        wakeup(s);
        release(&s->lock);
    }

    void P(struct semaphore *s){
        acquire(&s->lock);
        while(!s->count){
            sleep(s);
        }
        s->count -= 1;
        release(&s->lock);
    }
这样显然会造成死锁的问题。改进的办法：在进入sleep后，把s->lock释放了！ Final scheme.

    void V(struct semaphore *s){
        acquire(&s->lock);
        s->count += 1;
        wakeup(s);
        release(&s->lock);
    }

    void P(struct semaphore *s){
        acquire(&s->lock);
        while(!s->count){
            sleep(s,&s->lock);
        }
        s->count -= 1;
        release(&s->lock);
    }


### 7.6 : Code Sleep & Wake up 
sleep 函数:

    void
    sleep(void *chan, struct spinlock *lk)
    {
      struct proc *p = myproc();
      
      // Must acquire p->lock in order to
      // change p->state and then call sched.
      // Once we hold p->lock, we can be
      // guaranteed that we won't miss any wakeup
      // (wakeup locks p->lock),
      // so it's okay to release lk.
      if(lk != &p->lock){  //DOC: sleeplock0
        acquire(&p->lock);  //DOC: sleeplock1
        release(lk);
      }

      // Go to sleep.
      p->chan = chan;
      p->state = SLEEPING;

      sched();

      // Tidy up.
      p->chan = 0;

      // Reacquire original lock.
      if(lk != &p->lock){
        release(&p->lock);
        acquire(lk);
      }
    }

wake up 函数：

    void
    wakeup(void *chan)
    {
      struct proc *p;

      for(p = proc; p < &proc[NPROC]; p++) {
        acquire(&p->lock);
        if(p->state == SLEEPING && p->chan == chan) {
          p->state = RUNNABLE;
        }
        release(&p->lock);
      }
    }
    
在进入sleep函数后，进行的第一件事就是释放参数锁，防止死锁的情况产生，获取进程的锁(注意这里判断传入的参数锁和进程锁是否一样，如果一样就不要释放）

> 获取了进程锁之后，在哪里释放？
> 就像yield函数中的情况，比方说一个进程通过调用sleep从running state change to sleeping state.这时候这个进程运行在的CPU的scheduler实际上运行到了swtch(&c->context, &p->context);
sleep函数在修改了进程状态之后会调用sched进入到scheduler，通过release(&p->lock)释放进程锁。

      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
      }
      release(&p->lock);
    }

sleep函数在被别的进程唤醒后从sched调用回来，把进程的channel信息清空，然后恢复原来的锁。


wakeup 函数比较直观,它传入一个channel信息，然后从进程数组中寻找state = sleeping 和有相同channel的进程，然后修改它的process state;

有时候wakeup会唤醒多个进程，而如果多个进程都在等待pipe的数据输入，那么首先只有以个进程能够获得这个数据输入，那么对于其他进程来说这个wakeup is spurious.因此常常使用循环条件判断的方式进行sleep();

### 7.7 : Code Pipes

    struct pipe {
      struct spinlock lock;
      char data[PIPESIZE];
      uint nread;     // number of bytes read
      uint nwrite;    // number of bytes written
      int readopen;   // read fd is still open
      int writeopen;  // write fd is still open
    };

    int
    pipewrite(struct pipe *pi, uint64 addr, int n)
    {
      int i;
      char ch;
      struct proc *pr = myproc();

      acquire(&pi->lock);
      for(i = 0; i < n; i++){
        while(pi->nwrite == pi->nread + PIPESIZE){  //这里检查pipe buffer有没有满
          if(pi->readopen == 0 || pr->killed){
            release(&pi->lock);
            return -1;
          }
          wakeup(&pi->nread); //唤醒pi->nread channel上的进程
          sleep(&pi->nwrite, &pi->lock); //这里让pipewrite进程休眠，释放pi->lock，以便于pipe read operation
        }
        if(copyin(pr->pagetable, &ch, addr + i, 1) == -1)
          break;
        pi->data[pi->nwrite++ % PIPESIZE] = ch;
      }
      wakeup(&pi->nread);   //在这里已经完成了Pipe write操作，唤醒pi->nread channel上的进程进行pipe read opearation
      release(&pi->lock); 
      return i;
    }


    int
    piperead(struct pipe *pi, uint64 addr, int n)
    {
      int i;
      struct proc *pr = myproc();
      char ch;

      acquire(&pi->lock);
      while(pi->nread == pi->nwrite && pi->writeopen){  //DOC: pipe-empty 注意这里使用了while循环，体现了7.6节介绍的思想
        if(pr->killed){
          release(&pi->lock);
          return -1;
        }
        sleep(&pi->nread, &pi->lock); //DOC: piperead-sleep // sleep(&pi->read)相当于记录了这个pipe目前读到哪里
      }
      for(i = 0; i < n; i++){  //DOC: piperead-copy
        if(pi->nread == pi->nwrite)
          break;
        ch = pi->data[pi->nread++ % PIPESIZE];
        if(copyout(pr->pagetable, addr + i, &ch, 1) == -1)
          break;
      }
      wakeup(&pi->nwrite);  //DOC: piperead-wakeup 这里可能会唤醒写操作时候满的进程
      release(&pi->lock);
      return i;
    }



### 7.8 ： Code Wait exit and kill

