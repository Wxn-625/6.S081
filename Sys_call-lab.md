# 1 操作系统结构

> 计算机的目标：同时处理多种任务。OS就去帮助实现这个目标。
>
> 所以OS需要完成：
>
> + 多路复用
> + 隔离性：确保某个任务实体失败，不会引起其他任务实体的失败
> + 交流性：多个任务实体并不是完全隔离。
>
> 这三个需求，最后使得OS设计者 使OS实现了四大功能：
>
> + 进程管理
> + 内存管理
> + 文件管理
> + 设备管理
>
> 假设没有OS，那么进程都是处于一个去中心化的状态，而CPU的数量是必然远小于进程数的，所以就会产生争夺CPU的情况，而且一般进程只能假设是一个恶意抢夺的状态。





**CODE ：system calls**：

exec的实现：

+ 把exec的参数放到 寄存器a0 和a1，将这个syscall的号码放入a7
+ 然后ecall会在kernel中执行 uservec、usertrap、syscall
+ syscall通过a7中的数值，去找到这个系统调用的函数，并执行。将返回结果放到a0.一般负数表示error，0表示成功。



**CODE ：system calls arguments**：

系统调用时，kernel通过把用户态的系统调用的包装函数的参数放到 trap frame里面来使用。然后通过argraw(int n) 来取第n个参数

这会遇到一个问题，如果user传递了一个指针，那么就不能简单的用argraw了。

+ user可能是有恶意的，它想读取其他进程的内容
+ user和kernel的page table不同，所以指针所指的内容是不一样的。



使用*fetchstr* 去获取指针所指的内容，这个函数又通过 *copyinstr*去实现复制user所指的内容。

copyinstr 需要把程序的页表和指针地址取出来，然后把对应物理地址的内容复制。





# 2 Lab

## 2.1 trace

> 目标是做一个可以追踪系统调用的功能。
>
> 思路是在进程的结构体 proc中添加一个成员 trace_mask用来表示需要追踪哪些系统调用。
>
> 题目已经给出了user级的trace文件，里面调用了系统调用trace，我们需要去完成sys_trace

+ 根据提示 在 *user/user.h*中添加函数原型， 在*user/usys.pl*中添加trace的信息，在*kernel/syscall.h* 中添加sys_trace的num数。
+ 在proc结构体中添加成员 trace_mask
+ 修改fork，使得子进程复制父进程的 trace_mask
+ 添加 *sys_trace*函数，其中主要是把trace_mask修改
+ 修改syscall，在结尾处根据trace_mask 打印内容



~~~c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
  if(((p->trace_mask >> num) & 1) == 1)
    printf("%d: syscall %s -> %d\n",p->pid,syscallname[num-1],p->trapframe->a0);
}

uint64
sys_trace(void){
    int mask;
    if(argint(0, &mask) < 0)
        return -1;
    myproc()->trace_mask = mask;

    return 0;
}
~~~

打印的地方少了个冒号，还找了半天~~~



## 2.2 sysinfo

> 这个比较简单，就是求整个OS在这个时刻的进程数和空余空间数。
>
> 处理的话就是写两个计算函数，然后把结果通过copyout写入user的sysinfo中。
>
> 因为对于kernel来说，他的地址是物理地址，而进程所见的是虚拟地址，所以它穿的指针需要 kernel去转换成物理地址。
>
> 一开始我把它当作之前CSAPP lab里面那种地址处理了，发现不行。然后看了看分配函数才明白。

~~~c
/* 求空余空间，要去看一下分配的函数，了解这边的空间是如何分配的。最小单位是页。 */


// count the free space
uint64
cFreeSpace(){
    uint64 cnt = 0;
    struct run* p = kmem.freelist;
    acquire(&kmem.lock);
    while(p){
        cnt += PGSIZE;
        p = p->next;
    }
    release(&kmem.lock);
    return cnt;
}
~~~



~~~c
/* 求进程数就直接遍历全局变量数组 proc[NPROC] 就行了，记得释放锁 */

// count the num of process except UNUSED one
uint64 cptNumProc(){
    struct proc* p;
    int cnt = 0;

    for(p = proc; p < &proc[NPROC]; p++) {
        acquire(&p->lock);
        if(p->state != UNUSED) {
            cnt++;
        }
        release(&p->lock);
    }
    return cnt;
}
~~~



~~~c
/* 系统调用 sys_sysinfo ,模仿fstat就行*/
uint64
sys_sysinfo(void){
    uint64 mfree = cFreeSpace();
    uint64 numProc = cptNumProc();
    // pointer to the struct
    uint64 p;
    if(argaddr(0,&p) < 0)
        return -1;
    struct proc *mproc = myproc();
    if(copyout(mproc->pagetable,p,(char *)&mfree,sizeof (uint64)) < 0)
        return -1;

    if(copyout(mproc->pagetable,p + sizeof(uint64),(char *)&numProc ,sizeof (uint64)) < 0)
        return -1;

    return 0;
}
~~~





![image-20210707234509143](D:\截图\image-20210707234509143.png)

快乐~~~
