因为`fork()`子进程时会调用`uvmcopy()`给子进程复制父进程的内存，所以如果要实现COW，需要先改`uvmcopy()`函数，使得它不再直接进行内存拷贝。如果是该页是可写的(标记了PTE_W)，就把置为不可写并把该页设置成PTE_COW。然后在子进程页表中加入映射，并增加引用计数。
```C
// Given a parent process's page table, copy
// its memory into a child's page table.
// Copies both the page table and the
// physical memory.
// returns 0 on success, -1 on failure.
// frees any allocated pages on failure.
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint64 flags;
  // char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    pte_t old_pte = *pte;
    if((flags & PTE_W)){// 如果该页是可写的
      flags = (flags | PTE_COW) & (~PTE_W);//设置为不可写，并标记PTE_COW
      *pte = PA2PTE(pa) | flags;
    }
    
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){// 加入映射
      *pte = old_pte;
      goto err;
    }
    acquire_refcnt();
    inc_refcnt(pa, 1);  // 引用数加1
    release_refcnt();
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```
当出现了缺页异常时，就用使用COW的机制为进程分配新的物理页。首先要在用户态陷入内核时进行处理。主要流程就是得到page fault对应的虚拟地址，并调用`copy_on_write()`函数进行COW操作。
```C
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if(r_scause() == 15){ // 处理page fault
    uint64 va = r_stval(); // 得到page fault的虚拟地址
    if(va > MAXVA) exit(-1);
    if(copy_on_write(va, p->pagetable) == 0){
      p->killed = 1;
    }
  }
  else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```
`copy_on_write()`的逻辑如下：
- 判断地址合法性
- 查页表得到对应的PTE和pa
- 构造新的PTE:置空PTE_COW，标记为PTE_W
- 分配一个新的页mem，并把pa的内容拷贝到mem中
- 释放原有的页表项
- 添加新的映射页表项
```C
uint64 copy_on_write(uint64 va, pagetable_t pt){
  va = PGROUNDDOWN(va);
  if(va >= MAXVA){
    return 0;
  }
  pte_t* pte = walk(pt, va, 0);
  if(pte == 0)return 0;
  if((*pte & PTE_V) == 0) return 0;
  if((*pte & PTE_U) == 0) return 0;
  uint64 pa = PTE2PA(*pte);
  uint64 flags = PTE_FLAGS(*pte);

  if(!(flags & PTE_COW)){
    printf("not cow\n");
    return 0;
  }

  char* mem = kalloc();
  if(mem == 0)
    return 0;
  memmove(mem, (char*)pa, PGSIZE);
  uvmunmap(pt, va, 1, 1);
  if(mappages(pt, va, PGSIZE, (uint64)mem, (flags & (~PTE_COW)) | PTE_W) != 0){
    kfree(mem);
    return 0;
  }
  return (uint64)mem;
}
```
因为需要对页进行引用计数，一个页只有当没有被使用时才free掉，所以在`kalloc.c`中增加和引用计数有关的数据结构。
```C
struct {
  struct spinlock lock;
  int counter[(PHYSTOP - KERNBASE)/PGSIZE];
}refcnt;
void acquire_refcnt(){
  acquire(&refcnt.lock);
}
void release_refcnt(){
  release(&refcnt.lock);
}
int get_ref_idx(uint64 pa){
  return (pa - KERNBASE) / PGSIZE;
}

void set_refcnt(uint64 pa, int n){
  refcnt.counter[get_ref_idx(pa)] = n;
}
int get_refcnt(uint64 pa){
  return refcnt.counter[get_ref_idx(pa)];
}
void inc_refcnt(uint64 pa, int n){
  refcnt.counter[get_ref_idx(pa)] += n;
}
```
在`kinit()`, `kalloc()`和`kfree()`中修改引用计数。
```C
void
kinit()
{
  initlock(&refcnt.lock, "refcnt");
  acquire_refcnt();
  int n = (PHYSTOP - KERNBASE)/PGSIZE;
  for(int i = 0; i < n; ++i){
    refcnt.counter[i] = 0;
  }
  release_refcnt();
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}

void
kfree(void *pa)
{
  struct run *r;
  acquire_refcnt();
  
  if(get_refcnt((uint64)pa) > 1){
    inc_refcnt((uint64)pa, -1);
    release_refcnt();
    return;
  }
  if(get_refcnt((uint64)pa) == 1){
    // 这个判断很重要，因为在xv6启动时会先执行kinit()，然后执行kfree()。此时要把内存加到空闲链表中，同时保证引用计数还都是0
    inc_refcnt((uint64)pa, -1);
  }
  release_refcnt();
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}

void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  if(r){
    acquire_refcnt();
    inc_refcnt((uint64)r, 1);
    release_refcnt();
  }
  return (void*)r;
}
```
最后是`copyout()`函数，它是把内核空间的数据拷贝到用户空间中。因此它要先判断用户空间所对应的虚拟地址是不是被标记为COW的，如果是的话，要先执行COW机制，分配新的空间。然后再执行拷贝操作。
```C
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if(va0 > MAXVA) return -1;
    pte_t* pte = walk(pagetable, va0, 0);
    if(pte == 0)
      return -1;
    if((*pte & PTE_V) == 0)
      return -1;
    if((*pte & PTE_U) == 0)
      return -1;
    if((*pte & PTE_COW) == 0) { // 不是 COW 页
      pa0 = walkaddr(pagetable, va0);
    } else {
      pa0 = copy_on_write(va0, pagetable);
    }
    
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```