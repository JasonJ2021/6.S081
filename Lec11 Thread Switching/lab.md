# Lab : Multithreading

## Task 1 : Uthread: switching between threads

这个部分主要是模仿kernel thread switch system实现一个用户级别的线程切换

    struct context{
        uint64 ra;
        uint64 sp;
        // callee-saved
        uint64 s0;
        uint64 s1;
        uint64 s2;
        uint64 s3;
        uint64 s4;
        uint64 s5;
        uint64 s6;
        uint64 s7;
        uint64 s8;
        uint64 s9;
        uint64 s10;
        uint64 s11;
    };
线程结构如下，包括上下文（其中包含ra sp & callee saved registers),属于每个线程的栈，线程的状态(这里只有runnable , free , running)

    struct thread {
    struct     context context;
    char       stack[STACK_SIZE]; /* the thread's stack */
    int        state;             /* FREE, RUNNING, RUNNABLE */

    };

下面是线程的初始化,它会设置main为第一个执行的线程，而且之后scheduler不会再执行main,因为它一直被设置为Running.

    void 
    thread_init(void)
    {
    // main() is thread 0, which will make the first invocation to
    // thread_schedule().  it needs a stack so that the first thread_switch() can
    // save thread 0's state.  thread_schedule() won't run the main thread ever
    // again, because its state is set to RUNNING, and thread_schedule() selects
    // a RUNNABLE thread.
    current_thread = &all_thread[0];
    current_thread->state = RUNNING;
    }

thread_create创建一个线程，它接受执行函数的指针把它传给ra，然后在线程数组中寻找空闲的线程，把该线程状态改为rannable,**把栈指针指向数组的末端**.

    void 
    thread_create(void (*func)())
    {
    struct thread *t;

    for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
        if (t->state == FREE) break;
    }
    t->state = RUNNABLE;
    // YOUR CODE HERE
    t->context.sp = (uint64)(t->stack+STACK_SIZE);
    t->context.ra = (uint64)func;
    }

下面是thread_schedule函数，它寻找下一个runnable的线程，进行上下文切换

    void 
    thread_schedule(void)
    {
    struct thread *t, *next_thread;

    /* Find another runnable thread. */
    next_thread = 0;
    t = current_thread + 1;
    for(int i = 0; i < MAX_THREAD; i++){
        if(t >= all_thread + MAX_THREAD)
        t = all_thread;
        if(t->state == RUNNABLE) {
        next_thread = t;
        break;
        }
        t = t + 1;
    }

    if (next_thread == 0) {
        printf("thread_schedule: no runnable threads\n");
        exit(-1);
    }

    if (current_thread != next_thread) {         /* switch threads?  */
        next_thread->state = RUNNING;
        t = current_thread;
        current_thread = next_thread;
        /* YOUR CODE HERE
        * Invoke thread_switch to switch from t to next_thread:
        * thread_switch(??, ??);
        */
        thread_switch((uint64)(&(t->context)) , (uint64)(&(next_thread->context)));

    } else
        next_thread = 0;
    }
注意这里thread_switch返回到next_thread的ra指向的位置，而ra的值是函数指针，它会指向该线程执行到了哪行代码！

thread_yield函数的调用类似于timer interrupt，这里省略了中断的步骤

    void 
    thread_yield(void)
    {
    current_thread->state = RUNNABLE;
    thread_schedule();
    }

thread_switch函数的代码和swtch.S中一样。


## Task 2 : Using Threads
这个部分要求我们在真正的unix系统下使用线程！ph程序主要实现了哈希表的put & get,但是没有加锁，我们首先的任务就是找出并行下会出错的代码，加锁

      pthread_mutex_lock(&(locks[i]));
      insert(key, value, &table[i], table[i]);
      pthread_mutex_unlock(&locks[i]);
这里注意要对整个insert操作加锁，似乎它的malloc函数也是线程不安全的...

另外这里有5张哈希表，如果插入的不是一张表实际上就没有必要等待锁的释放。所以可以给每张哈希表都赋一个锁

    pthread_mutex_t locks[5];

    static void 
    Pinsert(int key, int value, struct entry **p, struct entry *n,int i ){
        pthread_mutex_lock(&(locks[i]));
        insert(key, value, &table[i], table[i]);
        pthread_mutex_unlock(&locks[i]);
    }


## Task 3 : Barrier(moderate)

这个部分也是要求我们在真实的unix环境下编写线程程序，实现一个barrier,即在一个点停止下来，等待所有线程都达到这个点之后才继续执行。

这里给了两个辅助函数

    pthread_cond_wait(&cond, &mutex);  // go to sleep on cond, releasing lock mutex, acquiring upon wake up
    pthread_cond_broadcast(&cond);     // wake up every thread sleeping on cond

第一个函数让线程等待并且释放mutex锁，**等到它被唤醒了之后还会acquire**(这里注意，因此之后要释放锁)

第二个函数可以唤醒所有在cond上等待的线程

    static void 
    barrier()
    {
    // YOUR CODE HERE
    //
    // Block until all threads have called barrier() and
    // then increment bstate.round.
    //
    pthread_mutex_lock(&bstate.barrier_mutex);
    bstate.nthread++;
    if(bstate.nthread == nthread){
        bstate.nthread = 0;
        bstate.round++;
        pthread_cond_broadcast(&bstate.barrier_cond);
    }else{
        pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
    }
    pthread_mutex_unlock(&bstate.barrier_mutex);
    }
