# 1 页表



## 1.1 paging hardware

> xv6跑在 Sv39 RISC-V上，虚拟地址64位中的低39位是在使用的。所以地址范围是 2^39，而每页的大小为 4K，所以页表有 2^27的页表项(page table entries)。每个PTE包含 44位的物理页号(physical page number)。
>
> **为什么需要44位，而不是对应的27位？**：对于物理地址来说使用的是低56位。
>
> 通过虚拟地址的高27位去查页表，得到44位的物理页号，然后加上虚拟地址剩下的12位，组成56位的物理地址，最后查询。



<img src="D:\截图\image-20210708213442386.png" alt="image-20210708213442386" style="zoom:67%;" />





**多级页表**

这个页表的大小：2^32 <size < 2^33 ，显然这个是无法接受的，而且其中会有许多的PTE是不存在的，

在xv6中采用了三级页表。此时我们只需要在内存中放入 (2 ^14 < size < 2^15)大小的页表即可，这是可以接受的。而且可以去掉没有被映射的页的存储空间。

但这会使得我们 多访问了两次外存，获得页表。

也可以把所有的页表都放在内存里，假设其占用空间不大的情况下。

一般系统好像是7级页表，然后会把高级的页表放到内存中，低级的放在外存。

<img src="D:\截图\image-20210708220228536.png" alt="image-20210708220228536" style="zoom:67%;" />



**PTE flags**

> PTE的标志位表示该页的使用权限。  硬件相关的数据结构都定义在 *kernel/riscv.h*中

+ PTE_V：表示这个地址是否存在
+ PTE_R：表示是否可读该页
+ PTE_W：表示该页是否可写
+ PTE_X：表示CPU是否要把该页的内容当作指令来解释。（汇编里告诉我只要CS和IP指向就可以当作指令解释？？？）
+ PTE_Y：表示用户态的指令是否可以访问该页。



**satp register**

为了CPU可以使用页表，kernel会把根页表的地址放在 satp寄存器中，而每个cpu都有自己的 satp寄存器，所以可以并行多个进程。





## 3.2 kernel 地址空间

> kernel也有一个 page table。QEMU 模拟了一个 RAM，从地址 0X80000000开始，并且到 0X86400000（这个边界用 PHYSTOP记录）
>
> QEMU也模拟了IO设备，将IO设备用空间映射的方式，映射在 0X80000000以下的物理地址。kernel就可以通过地址去访问IO设备。
>
> kernel 的内存映射方式是 “直接映射“，但其中也有部分是非直接映射的：
>
> +  trampoline page：这个page的作用会在 Cha 4中分析，但看图可以知道 两个虚拟地址映射到同一个位置。
> + Guard page：这个是用来保护 Kstack的，Guard page 的 PTE_V 是否的，表示这个不存在，访问了就会返回异常，这可以在 kstack 溢出时 提供保护。

![image-20210708224320626](D:\截图\image-20210708224320626.png)

# 2 解析函数

## 2.1 vm.c/walk()

> walk 算是页表分配里面很关键的函数。
>
> 主要包含两个作用：
>
> + 查找某虚拟地址对应的物理地址所对应的页表项PTE
> + 为物理地址新建页表项：当做新的映射时，发现某表项不存在，就新建页表，并修改页表项

~~~c
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
~~~





## 2.2 vm.c/ walkaddr()

> 比较简单，就是拿着walk得到的最后PTE，去转化成为真正的物理地址。

~~~c
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    return 0;
  if((*pte & PTE_V) == 0)
    return 0;
  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}
~~~



## 2.3 vm.c/ mappages

> 利用walk 函数 做虚拟地址到物理地址的映射，其中虚拟地址需要自己选择

~~~c
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
~~~





## 2.4 vm.c/uvmalloc()

> oldsize应该指的是已经分配的内存，但是这为什么还需要作为参数输入呢？

~~~c
uint64
uvmalloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  char *mem;
  uint64 a;

  if(newsz < oldsz)
    return oldsz;

  oldsz = PGROUNDUP(oldsz);
  for(a = oldsz; a < newsz; a += PGSIZE){
    mem = kalloc();
    if(mem == 0){
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
    memset(mem, 0, PGSIZE);
    if(mappages(pagetable, a, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
      kfree(mem);
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
  }
  return newsz;
}
~~~





## 2.5 vm.c/uvmdealloc()

> 为什么不需要 free？？

~~~c
uint64
uvmdealloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  if(newsz >= oldsz)
    return oldsz;

  if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 1);
  }

  return newsz;
}
~~~





## 2.6 vm.c/uvmunmap()

> 可选择是否删除物理页。 问题是如果不删除，它也不会删除页表，只是把页表项置为0，那这块在之后如何再用呢。
>
> pte是最底层的PTE，即高层的是不会置为0的。然后再看walk函数，再walk函数中，如果是alloc的情况，遇到最底层PTE为空时是可分配的。

~~~c
// Remove npages of mappings starting from va. va must be
// page-aligned. The mappings must exist.
// Optionally free the physical memory.
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
~~~



## 2.7 freewalk()

> 递归删除内容。这个函数的递归截至条件很令人疑惑。这个函数会删除物理内存吗？不会，应该是再uvmunmap函数中删除物理内存的。那么freewalk 是如何在底层的页表上停止递归的呢？只有当pte == 0 或者PTE_V == 0 时，才会停止递归。那么到真实的物理地址是怎么停止递归的呢？ 
>
> 看注释可以了解到所有的叶子节点映射需要已经被移除，所以这个时候的递归截至条件就在pte = 0这边。

~~~c
// Recursively free page-table pages.
// All leaf mappings must already have been removed.
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}
~~~



## 2.8  proc_freepagetable()

> 不知道处理TRAMPOLINE和TRAPFRAME的意义在哪。讲道理对于后面的uvmunmap和freewalk都无关啊。
>
> **猜测**：
>
> + TRAMPOLINE的PTE所带的flag可能对于freewalk的条件有所冲突
> + 预防uvmunmap(pagetable, 0, PGROUNDUP(sz)/PGSIZE, 1)，该语句可能会释放掉TRAMPOLINE
>
> **更正**
>
> freewalk函数需要已经把叶子节点的映射全部移除

~~~c
// Free a process's page table, and free the
// physical memory it refers to.
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmfree(pagetable, sz);
}

// Free user memory pages,
// then free page-table pages.
void
uvmfree(pagetable_t pagetable, uint64 sz)
{
  if(sz > 0)
    uvmunmap(pagetable, 0, PGROUNDUP(sz)/PGSIZE, 1);
  freewalk(pagetable);
}
~~~



# 3 Lab



## 3.2

**很疑惑，没什么头绪，然后参考了他人的作业**

**Q：**

+ 每个进程需要有一张内核页表，且该内核页表可以直接对用户页表的虚拟内存进行映射， 问题是 内核页表还需要映射内核栈的内容，那内核页表可以映射的内容就少了一块，那怎么才能完全映射出用户页表的内容呢(考虑是不需要映射用户页表的代码段内容，但我也没办法用这块内容，因为长度不一定)。看了别人的说明，说是用户页表只映射到CLINT以下 部分，不知道这个表示啥。

+ 问题描述的时候没懂 “For this step, each per-process kernel page table should be identical to the existing global kernel page table.”这句话。

  就是会有单独的全局页表，在不执行进程内容的时候 CPU所指的内核页表需要是**全局内核页表**

+ 还有就是lab的第二部分是只需要完成内核页表的建立，和用户页表的同步是在第三部分做。







> 在struct proc 中添加成员 k_pagetable

~~~c
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
  pagetable_t k_pagetable;     // Kernel page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
~~~



> 增加kvmmake()函数：新建内核页表，基本是照抄 kvminit中的内容，除了没有复制CLINT的部分。疑惑，不知道哪里有说明user的空间是小于PLINT部分。

~~~c
pagetable_t kvmmake(){
    pagetable_t k_pagetable;
    k_pagetable = (pagetable_t) kalloc();
    memset(k_pagetable, 0, PGSIZE);

    // uart registers
    mappages(k_pagetable,UART0,  PGSIZE,UART0, PTE_R | PTE_W);

    // virtio mmio disk interface
    mappages(k_pagetable,VIRTIO0,  PGSIZE,VIRTIO0, PTE_R | PTE_W);


    // PLIC
    mappages(k_pagetable,PLIC,  0x400000,PLIC, PTE_R | PTE_W);

    // map kernel text executable and read-only.
    mappages(k_pagetable,KERNBASE, (uint64)etext-KERNBASE, KERNBASE, PTE_R | PTE_X);

    // map kernel data and the physical RAM we'll make use of.
    mappages(k_pagetable,(uint64)etext, PHYSTOP-(uint64)etext, (uint64)etext,  PTE_R | PTE_W);

    // map the trampoline for trap entry/exit to
    // the highest virtual address in the kernel.
    mappages(k_pagetable,TRAMPOLINE, PGSIZE,(uint64)trampoline,  PTE_R | PTE_X);
    return k_pagetable;
}

/*
 * create a direct-map page table for the kernel.
 */
void
kvminit()
{
    kernel_pagetable = kvmmake();
    // CLINT
    kvmmap(CLINT, CLINT, 0x10000, PTE_R | PTE_W);

}
~~~



> 修改proc.c/procinit()，该函数是将所有的进程（包括UNUSED进程）所带的内核栈都初始化。现在我们为每个进程都维护了一个内核页表，所以内核栈就都放在各自的内核页表中，全局页表中不再维护内核栈的内容。只为进程初始化 lock

~~~c
void
procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");

      // Allocate a page for the process's kernel stack.
      // Map it high in memory, followed by an invalid
      // guard page.
//      char *pa = kalloc();
//      if(pa == 0)
//        panic("kalloc");
//      uint64 va = KSTACK((int) (p - proc));
//      kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
//      p->kstack = va;
  }
  kvminithart();
}
~~~



> 将初始化内核栈的内容加到 allocproc中

~~~c
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
  
  // An empty kernel_pagetable.
  p->k_pagetable = kvmmake();
  
  // Alloc kernel stack
  char *pa = kalloc();
  if(pa == 0)
      panic("kalloc");
  uint64 va = KSTACK((int) (0));
  mappages(p->k_pagetable,va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack = va;
  

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
~~~



> 修改 scheduler()，即每次运行进程时，把cpu的页表换成进程的内核页表。参考kvminithart()，修改cpu的satp寄存器内容。

~~~c
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
          w_satp(MAKE_SATP(p->k_pagetable));
          sfence_vma();

        swtch(&c->context, &p->context);

        kvminithart();
        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

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
~~~





> 提供删除内核页表的函数。注意在删除的时候不会删除物理页，除了内核栈的物理页。

~~~c
// 在freeproc中添加删除内核页表的函数
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  if(p->k_pagetable)
      proc_free_kernel_pagetable(p->k_pagetable);
  p->pagetable = 0;
  p->k_pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}

// 注意根据freewalk函数的作用，我们需要先把所有的leaf都删除，即kvmmake 的映射部分。kstack部分需要显式删除内存页。最后调用freewalk
void
proc_free_kernel_pagetable(pagetable_t pagetable)
{
    uvmunmap(pagetable, UART0, 1, 0);
    uvmunmap(pagetable, VIRTIO0, 1, 0);
//    uvmunmap(pagetable,CLINT, 0x10000/PGSIZE, PTE_R | PTE_W);
    uvmunmap(pagetable, PLIC, 0x400000/PGSIZE, 0);
    uvmunmap(pagetable, KERNBASE, ((uint64)etext-KERNBASE)/PGSIZE, 0);
    uvmunmap(pagetable, (uint64)etext, (PHYSTOP-(uint64)etext)/PGSIZE, 0);
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);

    uvmunmap(pagetable, KSTACK(0),1,1);
    freewalk(pagetable);
}
~~~



> 需要修改kvmpa，将全局页表改为进程的内核页表。之后使用的就是新的内核页表。其中myproc()->k_pagetable一直用不了，然后头文件添加了 "spinlock.h" 和 "proc.h"，解决了 struct proc 的不完全定义问题

~~~c
uint64
kvmpa(uint64 va)
{
  uint64 off = va % PGSIZE;
  pte_t *pte;
  uint64 pa;

  pte = walk(myproc()->k_pagetable, va, 0);
  if(pte == 0)
    panic("kvmpa");
  if((*pte & PTE_V) == 0)
    panic("kvmpa");
  pa = PTE2PA(*pte);
  return pa+off;
}
~~~



**疑惑**

如果子进程拷贝了父进程的页表，那么如果子进程进行了freeproc ，那么所对应的物理页不会被释放吗？

**答**

Xv6 没有实现Copy on write技术。所以父进程和子进程的物理内存是不重复的。





## 3.3 



> 将copyin_new 和copyinstr_new放到对应的函数体里

~~~c
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable,dst,srcva,len);
}

// Copy a null-terminated string from user to kernel.
// Copy bytes to dst from virtual address srcva in a given page table,
// until a '\0', or max.
// Return 0 on success, -1 on error.
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable,dst,srcva,max);
}
~~~





> 将改变user page table mapping的位置加上 kernel page table 的改变，主要是 4个函数：fork、exec、sbrk、userinit、注意我们这边假定user space 是小于PLIC的所以在exec中需要进行判断。

~~~c
// fork
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }

  //Copy kernel page table from parent
  for(int va = 0;va < p->sz;va += PGSIZE){
      pte_t* pte_u = walk(np->pagetable,va,0);
      pte_t* pte_k = walk(np->k_pagetable,va,1);
      *pte_k = (*pte_u) & ~PTE_U;
  }

  np->sz = p->sz;

  np->parent = p;

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  np->state = RUNNABLE;

  release(&np->lock);

  return pid;
}
~~~

~~~c
int
exec(char *path, char **argv)
{
  char *s, *last;
  int i, off;
  uint64 argc, sz = 0, sp, ustack[MAXARG+1], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();

  begin_op();

  if((ip = namei(path)) == 0){
    end_op();
    return -1;
  }
  ilock(ip);

  // Check ELF header
  if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  if(elf.magic != ELF_MAGIC)
    goto bad;

  if((pagetable = proc_pagetable(p)) == 0)
    goto bad;

  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    uint64 sz1;
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    if(sz1 >= PLIC)
        goto bad;
    sz = sz1;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op();
  ip = 0;

  p = myproc();
  uint64 oldsz = p->sz;

  // Allocate two pages at the next page boundary.
  // Use the second as the user stack.
  sz = PGROUNDUP(sz);
  uint64 sz1;
  if((sz1 = uvmalloc(pagetable, sz, sz + 2*PGSIZE)) == 0)
    goto bad;
  sz = sz1;
  uvmclear(pagetable, sz-2*PGSIZE);
  sp = sz;
  stackbase = sp - PGSIZE;

  // Push argument strings, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      goto bad;
    sp -= strlen(argv[argc]) + 1;
    sp -= sp % 16; // riscv sp must be 16-byte aligned
    if(sp < stackbase)
      goto bad;
    if(copyout(pagetable, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    ustack[argc] = sp;
  }
  ustack[argc] = 0;

  // push the array of argv[] pointers.
  sp -= (argc+1) * sizeof(uint64);
  sp -= sp % 16;
  if(sp < stackbase)
    goto bad;
  if(copyout(pagetable, sp, (char *)ustack, (argc+1)*sizeof(uint64)) < 0)
    goto bad;

  // arguments to user main(argc, argv)
  // argc is returned via the system call return
  // value, which goes in a0.
  p->trapframe->a1 = sp;

  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));
    
  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;

    uvmunmap(p->k_pagetable,0, PGROUNDUP(oldsz)/PGSIZE,0);
  //Copy kernel page table from parent
    for(int va = 0;va < sz;va += PGSIZE){
        pte_t* pte_u = walk(p->pagetable,va,0);
        pte_t* pte_k = walk(p->k_pagetable,va,1);
        *pte_k = (*pte_u) & ~PTE_U;
    }

  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp; // initial stack pointer
  proc_freepagetable(oldpagetable, oldsz);
  if(p->pid == 1)
      vmprint(p->pagetable);

  return argc; // this ends up in a0, the first argument to main(argc, argv)

 bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}

//sbrk
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  if(growproc(n) < 0)
    return -1;

  struct proc* p = myproc();

  if(n > 0){
      for(int j = addr;j < addr + n;j += PGSIZE){
          pte_t *pte_u = walk(p->pagetable,j,0);
          pte_t *pte_k = walk(p->k_pagetable,j,1);
          *pte_k = (*pte_u) & ~PTE_U;
      }
  }else{
      for(int j = addr - PGSIZE;j >= addr + n;j -= PGSIZE){
          uvmunmap(p->k_pagetable,j,1,0);
      }
  }
  return addr;
}

// userinit
// Set up first user process.
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

    for(uint64 va = 0;va < p->sz;va += PGSIZE){
        pte_t *pte_u = walk(p->pagetable,va,0);
        pte_t *pte_k = walk(p->k_pagetable,va,1);
        *pte_k = (*pte_u) & ~PTE_U;
    }

  release(&p->lock);
}
~~~

![image-20210714201510896](D:\截图\image-20210714201510896.png)
