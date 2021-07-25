# 1 Copy-on-Write Fork Lab

> 实现xv6 4.6节中的COW技术。



**起始工作**

~~~c
// 1.增加标志位PTE_C的宏，表示是否为COW页
// riscv.h
#define PTE_C (1L << 8) // COW page flag

// 2.增添全局变量 pageCount 来对所有分配的物理页引用计数。
// kalloc.c
uint *pageCount[32];

~~~



**初始化pageCount**

> + 需要用32个页去计数从 KERBASE- PHYSTOP(这部分是可用于分配给进程的物理内存) 的所有页。
>   + 注意：xv6 book上写的PHYSTOP 是0x86400000，而在代码里面是0x88000000
> + 另外写了一个分配函数count_kalloc() 去专门分配这32个页，因为原来的kalloc()函数中添加了引用计数部分的修改，所以需要在调用kalloc之前完成这32页的分配。

~~~c
// kalloc.c
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
  for(int i = 0;i < 32;i++){
      pageCount[i] = count_kalloc();
      memset((char*)pageCount[i], 0, PGSIZE);
  }
}


void *
count_kalloc(void)
{
    struct run *r;

    acquire(&kmem.lock);
    r = kmem.freelist;
    if(r)
        kmem.freelist = r->next;
    release(&kmem.lock);

    if(r)
        memset((char*)r, 5, PGSIZE); // fill with junk

    return (void*)r;
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
    pageCount[GETPAGEOFF(r)][GETINTOFF(r)] = 1;
  }

  return (void*)r;
}
~~~



**修改uvmcopy()**

> uvmcopy就是用在fork中去拷贝父进程的所有物理页，现在我们不需要去复制物理页，但是需要在这里做一些工作：
>
> + 修改标志位：父子进程的PTE_W都要置空，PTE_C都要置1
> + 增加物理页的引用计数
> + 删除复制物理内存部分的代码

~~~c
// vm.c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
//  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    *pte = *pte & ~PTE_W;
    *pte = *pte | PTE_C;
    pa = PTE2PA(*pte);
    // increment the page count
    pageCount[GETPAGEOFF(pa)][GETINTOFF(pa)]++;

      flags = PTE_FLAGS(*pte);
//    if((mem = kalloc()) == 0)
//      goto err;
//    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
//      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
//  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
~~~



**修改 usertrap.c**

> 跟LazyAlloc差不多，主要的处理逻辑就在trap里面，内容也很像。
>
> + 先判断是否是COW引起的缺页
> + 如果是的话，复制一份新的物理页 ，并且修改pte的内容，包括指向的物理页，以及标志位。
> + 减少引用计数，如果为0，就释放。

~~~c
// trap.c
else {
      uint64 va = r_stval();
      if((r_scause() != 13 && r_scause() != 15) || va >= p->sz){
          printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
          printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
          p->killed = 1;
      }else{
          pte_t *pte = walk(p->pagetable,va,0);
          if((*pte & PTE_C) && !(*pte & PTE_W)){
              uint64 pa = PTE2PA(*pte);
              char* mem = (char *)kalloc();
              if(mem == 0){
                  p->killed = 1;
              }else{
                  memmove(mem, (char*)(pa), PGSIZE);

                  // modify the flag bit
                  int flag = PTE_FLAGS(*pte);
                  flag = (flag | PTE_W) & ~PTE_C;
                  *pte = PA2PTE(mem) | flag;

                  // when the reference become zero ,free it
                  if(--pageCount[GETPAGEOFF(pa)][GETINTOFF(pa)] == 0){
                      kfree((void *)pa);
                  }
              }
          }
          else{
              printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
              printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
              p->killed = 1;
          }
      }

  }
~~~





**修改copyout()**

> 一开始还蛮疑惑为啥要改copyout，因为copyout是在kernel态进行的，缺页不会进入trap，所以需要在这边重新写一遍 trap的部分处理逻辑。
>
> 这里注意使用的页表应该是传入参数，而不是进程页表。一开始直接复制的 trap，然后debug de了贼旧~~
>
> 传入参数的话，主要是处理，exec新建页表的情况。

~~~c
//vm.c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
    
  // 有个测试点需要用到
  if(dstva > MAXVA)
  	return -1;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pte_t  *pte = walk(pagetable,va0,0);
    pa0 = walkaddr(pagetable, va0);
 	if(pa0 == 0)
      return -1;
    // If cow page
      if((*pte & PTE_C) && !(*pte & PTE_W)){

          char* mem = (char *)kalloc();
          if(mem == 0){
              return -1;
          }else{
              memmove(mem, (char*)(pa0), PGSIZE);

              // modify the flag bit
              int flag = PTE_FLAGS(*pte);
              flag = (flag | PTE_W) & ~PTE_C;
              *pte = PA2PTE(mem) | flag;

              // when the reference become zero ,free it
              if(--pageCount[GETPAGEOFF(pa0)][GETINTOFF(pa0)] == 0){
                  kfree((void *)pa0);
              }
          }
          pa0 = (uint64 )mem;
      }


   
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
~~~



**修改uvmunmap()**

> 现在对于物理页的删除，我们是依靠引用计数为0的条件，所以需要修改uvmunmap的显式删除。

~~~c
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
      if(--pageCount[GETPAGEOFF(pa)][GETINTOFF(pa)] == 0)
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
~~~





![image-20210725042029762](D:\截图\image-20210725042029762.png)