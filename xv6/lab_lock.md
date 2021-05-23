# lock

## 1 Memory allocator

### 1.1 任务

实现每个CPU一个memeory allocator来管理空闲链表。从而能更细粒度地并发分配内存。如果一个CPU的空闲链表空了，它可以去其他CPU中“偷”一个空闲块。

### 1.2 实现

更改文件`kernel/kalloc.c`

首先给每个CPU分配一个空闲链表和锁：

```c
struct kmem{
  struct spinlock lock;
  struct run *freelist;
};

struct kmem kmems[NCPU];
```

在`kinit()`中初始化每个kmem的锁。`kinit()`只会被id为0的CPU执行1次。

```c
void
kinit()
{
  push_off();
  int currentid = cpuid();
  pop_off();
  printf("# cpuId:%d \n",currentid);// 只有0号CPU会调用
  char name[128];
  for(int i = 0; i < NCPU; ++i){
    snprintf(name, 128, "kmem_%d", i);
    initlock(&kmems[i].lock, name);
  }
  freerange(end, (void*)PHYSTOP);
}
```

`kfree()`和原来类似，就是把一个页放到对应CPU的kmem的空闲链表中。这里只是改动了加锁和解锁步骤。根据提示，在获取cpuid时要用`push_off()`关中断，然后再用`pop_off()`开中断。

```c
void
kfree(void *pa)
{
  struct run *r;
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;
  push_off();
  int cid = cpuid();
  pop_off();
  acquire(&kmems[cid].lock);
  r->next = kmems[cid].freelist;
  kmems[cid].freelist = r;
  release(&kmems[cid].lock);
}
```

`kalloc()`首先会看自己的空闲链表中是否有空闲块。如果没有，就遍历其他CPU的空闲链表，从中偷一个空闲块。

```c
void *
kalloc(void)
{
  push_off();
  int cid = cpuid();
  pop_off();
  struct run *r;

  acquire(&kmems[cid].lock);
  r = kmems[cid].freelist;
  if(r){
    kmems[cid].freelist = r->next;
  }else{
    for(int i = 0; i < NCPU; ++i){// 遍历其他的CPU
      if(i == cid){
        continue;
      }
      acquire(&kmems[i].lock);
      if(kmems[i].freelist){
        r = kmems[i].freelist;
        kmems[i].freelist = kmems[i].freelist->next;
        release(&kmems[i].lock);
        break;
      }
      release(&kmems[i].lock);
    }
  }
  release(&kmems[cid].lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

## 2 Buffer cache

### 2.1 任务

之前的buffer cache是用双向链表+锁实现的，锁的粒度太大。要改成哈希表做cache，给每个桶分配一个锁，从而减小锁的粒度。

### 2.2 实现

定义哈希桶的个数，根据提示定义大小为13。

```c
// kernel/param.h
#define NBUCKETS     13
```

修改`struct buf`的定义：

根据提示，加入了`timestamp`字段来实现LRU。因为加入时间戳后不需要维护链表的顺序了，所以可以删掉指向前驱结点的指针。

```c
// kernel/buf.h
struct buf {
  int valid; // has data been read from disk?
  int disk;  // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  // struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
  uint timestamp;
};
```

以下修改均在`kernel/bio.c`文件中。

定义哈希表和哈希函数：

```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  // struct buf head;
  struct buf buckets[NBUCKETS];
  struct spinlock locks[NBUCKETS];

} bcache;
int bhash(uint blockno){
  return blockno % NBUCKETS;
}
```

修改`binit()`，初始化每个锁。把`bcache.buf`的每个节点均匀放到哈希桶中。

```c
void
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    initsleeplock(&b->lock, "buffer");
  }
  char name[128];
  b = bcache.buf;
  for(int i = 0; i < NBUCKETS; ++i){
    bcache.buckets[i].next = &bcache.buckets[i];
    snprintf(name, 128, "bcache_%d", i);
    initlock(&bcache.locks[i], name);
      // 把bcache.buf的每个节点均匀放到哈希桶中
    for(int j = 0; j < NBUF / NBUCKETS; ++j){
      b->blockno = i;
      b->refcnt = 0;
      b->next = bcache.buckets[i].next;
      bcache.buckets[i].next = b;
      b++;
    }
  }
  // 因为不是整除的，所以把余下的节点放到0号桶中
  for(;b < bcache.buf + NBUF; ++b){
    b->blockno = 0;
    b->refcnt = 0;
    b->next = bcache.buckets[0].next;
    bcache.buckets[0].next = b;
  }
}

```

修改`brelse()`，减小锁的粒度。更改引用计数，如果它的引用计数减为0，就记录当前时间戳。在替换时要替换引用计数为0且时间戳最小的。

```c
// Release a locked buffer.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  int idx = bhash(b->blockno);
  acquire(&bcache.locks[idx]);
  if(b->refcnt > 0){
   b->refcnt--;
  }
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->timestamp = ticks;
  }
  
 
  release(&bcache.locks[idx]);
}
```

修改`bget()`，主要流程是：

- 先查看是否缓存了该块，如果缓存了就直接返回
- 如果没缓存，就先查看本桶中是否有空闲的块
- 如果本桶中没有，就遍历其他的桶，看是否有空闲的块

```c
static struct buf*
bget(uint dev, uint blockno)
{
  
  struct buf *b;
  int idx = bhash(blockno);

  acquire(&bcache.locks[idx]);
  for(b = bcache.buckets[idx].next; b != &bcache.buckets[idx]; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.locks[idx]);
      acquiresleep(&b->lock);
      return b;
    }
  }


  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  // 尝试在本桶中寻找
  int min_time = 0x8fffffff;
  struct buf *replace_buf = 0;
  for(b = bcache.buckets[idx].next; b != &bcache.buckets[idx]; b = b->next){
    if(b->refcnt == 0 && b->timestamp < min_time){
      min_time = b->timestamp;
      replace_buf = b;
    }
  }
  if(replace_buf){// 如果找到了
    replace_buf->dev = dev;
    replace_buf->refcnt = 1;
    replace_buf->blockno = blockno;
    replace_buf->valid = 0;
    release(&bcache.locks[idx]);
    acquiresleep(&replace_buf->lock);
    return replace_buf;
  }
  // 在其他桶中寻找
  acquire(&bcache.lock);
  min_time = 0x8fffffff;
  replace_buf = 0;
  for(b = bcache.buf; b < bcache.buf + NBUF; ++b){
    if(b->refcnt == 0 && b->timestamp < min_time){
      replace_buf = b;
      min_time = b->timestamp;
    }
  }
  if(replace_buf){// 找到空闲块，要把它移动到idx的桶中
      // 因为是单链表，需要先找到它的前驱，将它从ridx的桶中移除
    int ridx = bhash(replace_buf->blockno);
    acquire(&bcache.locks[ridx]);
    struct buf *rb; 
    for(rb = &bcache.buckets[ridx]; rb->next != &bcache.buckets[ridx]; rb = rb->next){
      if(rb->next == replace_buf){
        break;
      }
    }
    rb->next = replace_buf->next;
    replace_buf->next = bcache.buckets[idx].next;
    
    bcache.buckets[idx].next = replace_buf;
    replace_buf->dev = dev;
    replace_buf->refcnt = 1;
    replace_buf->blockno = blockno;
    replace_buf->valid = 0;
    release(&bcache.locks[ridx]);
    release(&bcache.lock);
    release(&bcache.locks[idx]);
    acquiresleep(&replace_buf->lock);
    return replace_buf;

  }else{
    release(&bcache.lock);
    release(&bcache.locks[idx]);
    panic("bget: no buffers");
  }
  
}
```

### 注意

- 注意锁的释放和释放的顺序。第一次忘记释放锁造成死锁
- 这个链表是带虚拟头结点的链表。链表的尾部指向了虚拟头结点
- 注意加锁的代码块，反正大一点总是不会出错的













