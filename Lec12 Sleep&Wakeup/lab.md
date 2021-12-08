# Lab: locks
Before writing code, make sure to read the following parts from the xv6 book :

Chapter 6: "Locking" and the corresponding code.
Section 3.5: "Code: Physical memory allocator"
Section 8.1 through 8.3: "Overview", "Buffer cache layer", and "Code: Buffer cache"
## Memory allocator (moderate)
这个任务主要要求给每个CPU都分配一个独立的链表，避免不同CPU之间争抢锁的情况(上个lab完成过)

## Buffer cache (hard)
所有的CPU都共享一个相同的Buffer cache,用来存储最近访问过的disk block以及作为一个读写disk的buffer.
这个任务的挑战性在于它不能像上一个任务一样给每个CPU都分配一个独立的buffer cache.但是hints提示我们可以利用哈希表。 可以设置BUCKET = 13 有13 个筒，通过对blockno的hashing,实现13个不同的双向链表，从而达到减少争抢的情况。；
- 对bcache的修改,注意我这里创建了13个不同的双向链表。
  
      struct {
      struct spinlock lock;
      struct buf buf[NBUF]; //共同资源！

      // Linked list of all buffers, through prev/next.
      // Sorted by how recently the buffer was used.
      // head.next is most recent, head.prev is least.
      struct buf head[BUCKETS];
      struct spinlock hashlock[BUCKETS];
      } bcache;

- 对binit的修改，

      void
      binit(void)
      {
        struct buf *b;

        initlock(&bcache.lock, "bcache");

        // Create linked list of buffers
        //现在添加到了13个bucket head
        for(int i = 0 ; i < BUCKETS ; i++){
            initlock(&bcache.hashlock[i],"bcache");
            bcache.head[i].prev = &bcache.head[i];
            bcache.head[i].next = &bcache.head[i];
        }
        int hashnum;
        for(int i = 0 ; i < NBUF ; i++){
          b = bcache.buf + i ;
          hashnum = b->blockno % BUCKETS;
          b->next = bcache.head[hashnum].next;
          b->prev = &bcache.head[hashnum];
          initsleeplock(&b->lock, "buffer");
          bcache.head[hashnum].next->prev = b;
          bcache.head[hashnum].next = b;
        }
      }

- 对bget的修改，这里如果block已经被缓存了，做法和之前的code差不多，就是修改了一下获取的锁为对应链表的锁。而对于没有缓存的块，我借用了之前双向链表的LRU方法，
- 首先在hashnum所在的Bucket寻找空闲的块，如果有把它放到链表头的next,设置相应参数然后返回。如果没有，那就到别的bucket寻找，注意这里获取锁的顺序。找到了之后把它取出来放到hashnum所在的BUCKET中。
  

      static struct buf*
      bget(uint dev, uint blockno)
      {
        struct buf *b;
        uint hashnum =  blockno % BUCKETS;
        acquire(&bcache.hashlock[hashnum]);
        // Is the block already cached?
        for(b = bcache.head[hashnum].next; b != &bcache.head[hashnum]; b = b->next){
          if(b->dev == dev && b->blockno == blockno){
            b->refcnt++;
            release(&bcache.hashlock[hashnum]);
            acquiresleep(&b->lock);
            return b;
          }
        }
        // Not cached.
        // Recycle the least recently used (LRU) unused buffer.
        for (b = bcache.head[hashnum].prev; b != &bcache.head[hashnum]; b = b->prev)
        {
          if (b->refcnt == 0)
          {
            b->dev = dev;
            b->blockno = blockno;
            b->valid = 0;
            b->refcnt = 1;

            b->prev->next = b->next;
            b->next->prev = b->prev;

            b->next = bcache.head[hashnum].next;
            b->prev = &bcache.head[hashnum];
            
            bcache.head[hashnum].next->prev = b;
            bcache.head[hashnum].next = b;
            release(&bcache.hashlock[hashnum]);
            acquiresleep(&b->lock);
            return b;
          }
        }

        for (int j = (blockno + 1 ) % BUCKETS; j != hashnum; j = (j + 1) % BUCKETS)
        {
          acquire(&bcache.hashlock[j]);
          for (b = bcache.head[j].prev; b != &bcache.head[j]; b = b->prev)
          {
            if (b->refcnt == 0)
            {
              b->dev = dev;
              b->blockno = blockno;
              b->valid = 0;
              b->refcnt = 1;
              b->prev->next = b->next;
              b->next->prev = b->prev;
              release(&bcache.hashlock[j]);
              b->next = bcache.head[hashnum].next;
              b->prev = &bcache.head[hashnum];
              bcache.head[hashnum].next->prev = b;
              bcache.head[hashnum].next = b;
              release(&bcache.hashlock[hashnum]);
              acquiresleep(&b->lock);
              return b;
            }
          }
          release(&bcache.hashlock[j]);
        }
        release(&bcache.hashlock[hashnum]);
        panic("bget: no buffers");
      }

- 对brelse的修改，这个比较简单，就是获取相应的bucket锁，然后加到对应head的next之后

      void
      brelse(struct buf *b)
      {
        if(!holdingsleep(&b->lock))
          panic("brelse");

        releasesleep(&b->lock);
        uint hashnum = b->blockno % BUCKETS;
        acquire(&bcache.hashlock[hashnum]);
        b->refcnt--;
        if (b->refcnt == 0) {
          // no one is waiting for it.
          b->next->prev = b->prev;
          b->prev->next = b->next;
          b->next = bcache.head[hashnum].next;
          b->prev = &bcache.head[hashnum];
          bcache.head[hashnum].next->prev = b;
          bcache.head[hashnum].next = b;
        }
        
        release(&bcache.hashlock[hashnum]);
      }