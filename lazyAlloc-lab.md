# 1 Lab 

> lab 5就是去实现xv6 book 4.6中写的 Lazy page allocation

有个问题：page fault的trap是如何出现的？

## 1.1 Eliminate allocation from sbrk()

~~~
你的第一个任务是在sbrk系统调用中删除页分配的实现部分，这部分内容在 sysproc.c/sys_sbrk()中。sbkr(n)系统调用会给进程增加 n byte的空间，并且返回新空间的起始地址（等于old sz）。新的sbrk(n)应该只给进程的sz增加n，并且返回old sz。并且不会分配任何空间--所以你应该删除growproc()的调用（但是你仍要去增加 p->sz!)。
~~~

> 这个部分是要在调用 sbrk的情况下不分配新的内存页，但是需要修改 p->sz。
>
> 我这边选择直接修改 growproc的内容，这个函数除了sbrk 没有其他函数进行调用，所以直接修改。
>
> 这边是直接把 n > 0的内容注释掉，加上一条修改sz的。其实本来第一部分的lab，只需要把sbrk调用growproc 去掉，然后修改sz就行了，但是考虑 n < 0的情况，我们是应该直接删除页的，看了后面部分的lab，也确实有这个要求。

~~~c
// Grow or shrink user memory by n bytes.
// Return 0 on success, -1 on failure.
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){
      sz += n;
//    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
//      return -1;
//    }
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }
  p->sz = sz;
  return 0;
}
~~~

然后去测试 echo hi，发生了跟他一样的错误。这是为什么呢？然后从头开始找哪调用了sbrk()，最后发现是 shell里面会通过 malloc 给struct cmd分配内存。



## 1.2 Lazy allocation

> 这边直接把2、3部分都写在一起了。

~~~
通过在发生fault的地址映射新分配的物理页，来修改 trap.c 中与用户空间page fault相关的代码，然后返回用户空间使得进程重新执行。你应该在 打印"usertrap(): ..." 的 printf 调用前添上你的代码。为了使得echo hi运行，你可以修改 xv6 中任何 kernel code。
~~~



**修改trap.c**

> 这边注意几个点

+ 首先是区分fault的类型，不处理的包括：
  + 非page fault
  + 虚拟地址高于进程sz的
  + 用户栈的保护页，因为用户栈位置与内核栈不同，是在底下的，所以需要当作非alloc部分。还有就是注意从trapframe中去取sp，因为此时的sp应该不是之前的栈指针。
+ 然后就是处理 lazy alloc部分
  + 如果内存不够了，就杀死进程。
  + 参考 uvmalloc，写映射的部分。

~~~c
//修改 usertrap()里面对异常处理部分
else {
      uint64 va = r_stval();
      uint64 sp = p->trapframe->sp;
      // not a page fault
      if((r_scause() != 13 && r_scause() != 15) || va >= p->sz || (va < PGROUNDDOWN(sp) && va >= PGROUNDDOWN(sp - PGSIZE))){
          printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
          printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
          p->killed = 1;
      }
      // lazy alloc
      else{
          va = PGROUNDDOWN(va);
          char* mem;
          // if kalloc fails,kill the proc
          if((mem = kalloc()) == 0){
              p->killed = 1;
          }
          else{
              memset(mem,0,PGSIZE);
              if((mappages(p->pagetable, va, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U)) != 0){
                  kfree(mem);
                  // if fails,unmap and free the page
                  uvmunmap(p->pagetable,va,1,0);
              }
          }
      }

  }
~~~



**修改 vm.c/uvmunmap**

> 在lazy alloc 的情况下，是有可能产生这样的情况：sbrk修改了sz，但是最终没有分配物理内存的情况下，进行了uvmunmap。所以需要修改里面panic的情况：1.没有walk到pte 2.pte是不可访问的

~~~c
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;
  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
        continue;
//      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
        continue;
//      panic("uvmunmap: not mapped");
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



**修改 vm.c/uvmcopy**

> 类似于 uvmunmap，uvmcopy也要处理这样的情况。
>
> uvmcopy主要是在fork中被调用。如果父进程的有上述的情况，那么它也需要去冷处理page fault。

~~~c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
        continue;
//      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
        continue;
//      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
~~~



**再修改proc.c/growproc**

> 其他改完基本上所有case都能过了，但是那个 sbrkargs过不了。情况应该是在系统调用的时候不会发生page fault的情况，然后就没有办法去分配内存。这边卡了很久，最后查到 write的系统调用函数那边， 但是看不懂~~。最后试了一下每次sbrk 都再多分配一个字节，那么 内存位置 sbrk()，就是有效的。

~~~c
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){

    if((uvmalloc(p->pagetable, sz, sz + 1)) == 0) {
      return -1;
    }
    sz += n;
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }
  p->sz = sz;
  return 0;
}
~~~











![image-20210720162616640](D:\截图\image-20210720162616640.png)

![image-20210720162543464](D:\截图\image-20210720162543464.png)

