# pagetable

## 打印页表

实现`vmprint()`函数来打印页表。

可以参考`freewalk()`的实现来打印页表

```c
void vmprint_dfs(pagetable_t pagetable, int level){
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(!(pte & PTE_V)){
      continue;
    }
    for(int j = 0; j <= level; ++j){
      printf(".. ");
    }
    printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
    // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      vmprint_dfs((pagetable_t)child, level + 1);
    }
  }
}
void vmprint(pagetable_t pagetable){
  printf("page table %p\n", pagetable);
  vmprint_dfs(pagetable, 0);
}
```

加入测试代码：

```c
/// kernel/exec.c
int exec(char *path, char **argv) {
// ...
if(p->pid==1) vmprint(p->pagetable); // Code Added
return argc;
// ...
}
```



## 给每个进程实现自己的内核态页表

首先在`struct proc()`中增加一个保存内核页表的结构`kpagetable`

```c
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  pagetable_t kpagetable;     //kernel pagetable
};
```

在调用`allocproc()`的时候，为每个进程分配一个内核页表。这里参照`kvminit`来实现`kvmcreate()`函数，用于创建内核页表。

```c
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  // 创建kernel pagetable
  if((p->kpagetable = kvmcreate()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  // Allocate a page for the process's kernel stack.
  // Map it high in memory, followed by an invalid
  // guard page.
  char *pa = kalloc();
  if(pa == 0)
    panic("kalloc");
  // 映射内核栈
  uint64 va = KSTACK((int) (p - proc));
  kvm_map(p->kpagetable, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack = va;

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

```c
// 创建一个kernel pagetable
pagetable_t kvmcreate(){
  pagetable_t kpagetable = (pagetable_t)kalloc();
  if(kpagetable == 0){
    return 0;
  }
  memset(kpagetable, 0, PGSIZE);
  // uart registers
  kvm_map(kpagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvm_map(kpagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  kvm_map(kpagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvm_map(kpagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvm_map(kpagetable, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvm_map(kpagetable, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvm_map(kpagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
  return kpagetable;
}
```



因为在映射内核页表时传入的参数和系统自带的`kvmmap()`不同，所以参考`kvmmap()`的写法实现了`kvm_map()`。

```c
// add a mapping to the kernel page table.
// only used when booting.
// does not flush TLB or enable paging.
void
kvmmap(uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(kernel_pagetable, va, sz, pa, perm) != 0)
    panic("kvmmap");
}
void kvm_map(pagetable_t kpagetable, uint64 va, uint64 pa, uint64 sz, int perm){
  if(mappages(kpagetable, va, sz, pa, perm) != 0)
    panic("kvm_map");
}
```

因为在`kvmcreate()`中为每个进程映射了自己的内核栈，所以在`procinit()`中就不需要分配栈了。

```c
// initialize the proc table at boot time.
void
procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");
  }
}
```

当进程调度的时候，需要切换页表：

- 参考`kvminithart()`的实现
- 先调用`w_satp()`，设置第 1 级页表的基地址
- 再调用`sfence_vma()`清空 TLB（失效了）

```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        //切换内核页表
        w_satp(MAKE_SATP(p->kpagetable));
        sfence_vma();
        swtch(&c->context, &p->context);
        
        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
        // use kernel_pagetable when no process is running.
		    kvminithart();
        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```

在`freeproc()`的时候

- 首先释放栈的映射
- 释放页表，但是不能释放第 3 级页表指向的物理页
  - 因为他们都共享原来内核的代码之类的，这些不能释放
  - 内核栈也不用释放，一次分配，多次使用

```c
// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;

  // 解除kernel stack的映射
  kvmunmap(p->kpagetable, p->kstack);
  p->kstack = 0;
  // free kernel pagetable
  if(p->kpagetable){
    proc_free_kpagetable(p->kpagetable);
  }
  p->kpagetable = 0;


}
```

在这里并没有把第3级页表所指向的物理地址给free掉。

```c
void proc_free_kpagetable(pagetable_t kpagetable){
// there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = kpagetable[i];
    if(pte & PTE_V){
      kpagetable[i] = 0;
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      proc_free_kpagetable((pagetable_t)child);
      }
      
    }
  }
  kfree((void*)kpagetable);
}
```

## Simplify copyin/copyinstr

- `copyin()`函数在读取用户空间的虚拟地址的时候，需要先使用用户空间的页表，将其翻译成物理地址
- 我们在这里需要实现将每个进程用户空间保存的页表映射存到内核页表中，从而可以直接翻译访问

先在`defs.h`里面进行函数声明：

```c
// vmcopyin.c
int             copyinstr_new(pagetable_t, char *, uint64, uint64);
int             copyin_new(pagetable_t, char *, uint64, uint64);
```

修改原来的`copyin()`和`copyinstr()`

```
/ Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}

// Copy a null-terminated string from user to kernel.
// Copy bytes to dst from virtual address srcva in a given page table,
// until a '\0', or max.
// Return 0 on success, -1 on error.
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}
```

