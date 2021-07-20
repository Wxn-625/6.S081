# 1 Traps and  system call



> 有三种事件会导致CPU不按照原先的执行顺序执行：系统调用(ecall)、异常、硬件中断。
>
> 本书把以上几种统称为 trap(csapp中统称为异常)。
>
> Xv6分四个阶段处理trap:
>
> + RISC-V CPU的硬件处理
> + 为内核C语言这边的汇编向量
> + C语言形式的trap处理器
> + 系统调用程序或者是硬件驱动服务程序



## 1.1 RIS-V trap machinery

> 每个RISC-V CPU都有一系列的控制寄存器来处理trap。可参见riscv.h。
>
> 以下寄存器只能在内核态才能修改。

**以下是主要的几个寄存器**（kernel中也有一套一样的寄存器）:

+ **stvec**:kernel 把 trap handler的地址放在里面。
+ **sepc**：当trap发生时，stvec寄存器中的内容会写到pc 中，而sepc就是用来存pc之前的副本。trap执行完之后，**sret**命令会把sepc的内容写回pc中
+ **scause**：RISC-V会存一个表示该trap原因的数在这个寄存器
+ **sscratch**：内核会在此写入一个在trap handler 开始 时很有用的数在里面 （大雾~~）
+ **sstatus**：顾名思义是表示状态。
  + SIE bit 表示是否接受设备中断，在清除状态下会推迟设备中断，直到SIE bit设置。
  + SPP bit 表示trap 是来自用户态还是内核态，并且控制sret返回的状态。



**硬件执行trap的流程**

+ 如果是设备中断，且sstatus的 SIE 被清除了，那么不再执行以下步骤
+ 清除SIE
+ 将pc中的内容复制到 sepc 中保存
+ 保存目前的状态到 sstatus 中的SPP中
+ 设置 scause ，即表示 trap 原因的数字。
+ 改为特权模式
+ 将 stvec中的内容复制到 pc 中
+ 开始执行pc



## 1.2  Trap from user space

> High level 的执行路径 ： uservec(trampoline.S) -> usertrap(trap.c) -> usertrapret(trap.c) -> userret(trampoline.S)
>
> 用户态的trap会比内核态的复杂，因为用户页表并不映射完内核页表，栈指针可能会带有非法的内容。



**uservec**

因为用户态的trap不会切换页表，所以用户页表必须映射了**uservec**的内容，（即stvec寄存器指向的内容，**疑惑？？**）。uservec 必须将satp所指的页表转为内核页表。为了在转换后继续执行指令，uservec必须要内核页表和用户页表都映射到相同的地址。

Xv6使用了trampoline 去包含uservec。对于内核页表以及用户页表，所以trampoline都会映射到同一页物理内存。

当 uservec 开始运行，所有32个寄存器都保存了trap用户的信息，但是uservec需要寄存器去切换页表和产生存放寄存器内容的地址。

RISC-V通过把寄存器内容保存到一页物理内存上面，这个页叫做 trapframe，trapframe在进程创建的时候就会被映射到页表上面。具体实现方法：可见 uservec里面第一步是一个 **csrrw**命令，这个是把 **a0** 和**sscratch**寄存器的内容交换，而**sscratch**原本就是指向traoframe。因此此时 **a0**就指向trapframe。（注意此时我们的页表是用户页表，但是在内核页表的时候我们可以通过 p->trapframe 去访问这个保存页。）之后就是将寄存器的内容全部复制到trapframe上面。

trapframe还包括了内核栈指针、CPU的hartid、usertrap 的地址和内核页表的地址。在uservec中取得这些值，加载到寄存器中。并将satp转到kernel页表，并调用 **usertrap**

~~~assembly
        # restore kernel stack pointer from p->trapframe->kernel_sp
        ld sp, 8(a0)

        # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)

        # load the address of usertrap(), p->trapframe->kernel_trap
        ld t0, 16(a0)

        # restore kernel page table from p->trapframe->kernel_satp
        ld t1, 0(a0)
        csrw satp, t1
        sfence.vma zero, zero

        # a0 is no longer valid, since the kernel page
        # table does not specially map p->tf.

        # jump to usertrap(), which does not return
        jr t0
~~~





**usertrap**

> usertrap 是来确定trap的原因，并进行处理。

**疑惑**：上述说的控制寄存器的修改在哪里。

+ 先查看mode是否是用户态。
+ 修改 stvec，虽然不知道参数调的 kernelvec，注释和book上的都看不明白。。。
+ 把sepc中的副本传到trapframe上面。
+ 判断是否是系统调用或者设备中断，并执行相应命令。其中是系统调用的话会使得 pc的内容+4，因为需要执行吓一跳命令。
+ 调用 usertrapret

~~~c
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
  } else {
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
~~~



**usertrapret**

> usertrapret 主要是把一些寄存器什么的内容复原，为之后的trap做准备。最后会调用userret。



**userret**

> usertrapret 会传递 TRAMPFRAME 和 用户页表地址给 userret 到 a0 和 a1上面。（book上面把这两个搞反了）
>
> 会把 a0 的值 传给 sscratch 和 a1 传到 satp上面。然后把trampframe上面的值重新放回寄存器上面。

~~~assembly
.globl userret
userret:
        # userret(TRAPFRAME, pagetable)
        # switch from kernel to user.
        # usertrapret() calls here.
        # a0: TRAPFRAME, in user page table.
        # a1: user page table, for satp.

        # switch to the user page table.
        csrw satp, a1
        sfence.vma zero, zero

        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.
        ld t0, 112(a0)
        csrw sscratch, t0

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        ld s0, 96(a0)
        ld s1, 104(a0)
        ld a1, 120(a0)
        ld a2, 128(a0)
        ld a3, 136(a0)
        ld a4, 144(a0)
        ld a5, 152(a0)
        ld a6, 160(a0)
        ld a7, 168(a0)
        ld s2, 176(a0)
        ld s3, 184(a0)
        ld s4, 192(a0)
        ld s5, 200(a0)
        ld s6, 208(a0)
        ld s7, 216(a0)
        ld s8, 224(a0)
        ld s9, 232(a0)
        ld s10, 240(a0)
        ld s11, 248(a0)
        ld t3, 256(a0)
        ld t4, 264(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)

	# restore user a0, and save TRAPFRAME in sscratch
        csrrw a0, sscratch, a0
        
        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret

~~~





## 1.3  Code:Calling system calls

> 以 **initcode.S**调用exec为例进行分析。比较简单就不放代码了。

+ user code把exec的参数放在 a0 和 a1 寄存器上面，把系统调用号放在 a7 上面。系统调用号会匹配一个系统调用的列表，上面记录了系统调用函数的指针。
+ ecall会trap到kernel并且执行uservec，usertrap，最后是syscall。
+ syscall会根据a7的值选择执行的系统调用函数，最后会把返回值放到 p->trapframe->a0。





## 1.4 Code: System call arguments

> user code 会把参数通过寄存器传给内核，但是由于上述的trap处理，寄存器的内容会被存到trapframe里面，所以系统调用取值是从p->trapframe->a0 等上面取值的。因为内核页表和用户页表不同，所以需要进行转换。这部分在之前都有涉及，也不多做介绍了。

**取参数**

~~~c
# syscall.c
// argint、argaddr、argfd 通过argraw取值为相应的形式
static uint64
argraw(int n)
{
  struct proc *p = myproc();
  switch (n) {
  case 0:
    return p->trapframe->a0;
  case 1:
    return p->trapframe->a1;
  case 2:
    return p->trapframe->a2;
  case 3:
    return p->trapframe->a3;
  case 4:
    return p->trapframe->a4;
  case 5:
    return p->trapframe->a5;
  }
~~~



## 1.5 Traps from kernel space

> 在kernel在CPU上面运行的时候，**stvec**就指向 **kernelvec**代码。因为satp已经指向内核页表，所以不需要切换。kernelvec会保存所有的寄存器内容到 trapframe。
>
> kernelvec -> kerneltrap

**kernelvec**

> kernelvec 会保存所有的寄存器内容到**中断内核线程**（为什么是线程？？）的栈上面。这在转换线程的情况下显得很重要。



**kerneltrap**

> kerneltrap会处理两种类型的trap：设备中断和异常。

+ 如果是时钟中断，而且是一个内核线程在运行，那么会调用yield放弃CPU的使用权。
+ 当kerneltrap完成了，就要去恢复之前的状态，因为yield 可能会导致状态的混乱，所以之前右用变量巨鹿他们。

~~~c
void 
kerneltrap()
{
  int which_dev = 0;
  uint64 sepc = r_sepc();
  uint64 sstatus = r_sstatus();
  uint64 scause = r_scause();
  
  if((sstatus & SSTATUS_SPP) == 0)
    panic("kerneltrap: not from supervisor mode");
  if(intr_get() != 0)
    panic("kerneltrap: interrupts enabled");

  if((which_dev = devintr()) == 0){
    printf("scause %p\n", scause);
    printf("sepc=%p stval=%p\n", r_sepc(), r_stval());
    panic("kerneltrap");
  }

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2 && myproc() != 0 && myproc()->state == RUNNING)
    yield();

  // the yield() may have caused some traps to occur,
  // so restore trap registers for use by kernelvec.S's sepc instruction.
  w_sepc(sepc);
  w_sstatus(sstatus);
}
~~~





## 1.6 Page-fault exception

> Xv6中，在用户态出现异常，kernel会直接kill该进程，在kernel态出现异常，会panic。而在真实的OS中会很多不同的处理方式。
>
> 在OS中会利用page fault去提高资源利用率。

**COW fork**

> Xv6中未实现Copy on write fork 技术。

子进程直接使用父进程的物理内存，这当然会导致一些读写不一致的问题。

处理方式是：当子进程或父进程进行写的时候，报page exception。然后复制该物理页，然后修改父子进程的页表，使得一人一张对应的物理页。然后再resume同样的命令，此时就可以正常完成。

一般子进程都会采用exec，用新的内容去填充地址空间，所以只会产生少量的page falut，并且可以避免全部复制。



**Lazy alloc**

> 一般用sbrk申请的内存我们不会用完，所以在真实需要分配的时候，我们才会进行allocate

到真实需要使用的时候，会报page fault，然后再进程alloc，然后resume命令就可以完成。



**虚拟内存**

根据局部性原理，有很大一部分程序执行的内容是在短时间内不需要使用的。所以就把他们放在外存中，等到需要使用的时候，因为内存中不存在，所以会有 page fault，然后再调入内存。





# 2 Lab

> 开始做之前，先阅读xv6 book 的第四章内容，以及相关的源代码：
>
> + `kernel/trampoline.S`:有关用户空间与内核空间转换的汇编代码
> + `kernel/trap.c`: 处理中断的代码

## 2.1 RISC-V assembly

> 这边需要看一下 RISC-V的指令含义。

**哪些寄存器存放函数的参数？如main中调用printf在哪个寄存器中存放了参数13？**

~~~c
寄存器a0 - a7用来存放函数的参数。
a2 中存放了 13
~~~



**在main的汇编代码部分的哪里调用了 函数f，以及g？**

~~~
代码里没有调用f和g的部分，看hint就知道编译器优化成了inline function
~~~



**printf函数的地址在哪？**

~~~c
0x 628
~~~



**main中，在执行jarl指令进入printf之后， 寄存器ra中的值是什么？**

~~~c
0x	30
因为 指令auipc是把当前pc的值放入指定的位置。
~~~



**打印出的内容是什么**

~~~
unsigned int i = 0x00646c72;
printf("H%x Wo%s", 57616, &i);

HE110 Wolrd

%x是以十六进制打印内容，就是0x110
%s是打印字符串直到遇到0，因为是以小端形式存储，所以内存中的排布是  72 6c 64 00 ,对应的 ASCII码是 r l d 空

如果RISC-V是以大端形式存储，那么 57616不需要改，i需要改成 0x72 6c 64 00
~~~



**在“y=” 后面会打印什么**

~~~c
a2 寄存器中的内容。
riscv64-unknown-elf-addr2line -e kernel/kernel
0x0000003fffff9f80
0x0000003fffff9fc0
0x0000003fffff9fe0
~~~





## 2.2 Backtrace

> s0寄存器里存的地址是用户空间的地址还是内核空间的地址。就是内核空间的地址。

一开始打印出来的全是问号，然后发现要printf的是 *（Return Address），而不是地址的值。做的时候有点想当然了，没有搞清楚（Return Address）的意义。

此时应该是在内核空间，因为目前还在系统调用中。

~~~c
static void backtrace_recur(uint64* fp){
    if(fp == (uint64 *)PGROUNDUP((uint64 )fp))
        return;
    printf("%p\n",*(fp - 1));
    fp = (uint64 *)(*(fp - 2));
    backtrace_recur(fp);
}

void
backtrace(){
    uint64* fp = (uint64 *)r_fp();
    printf("backtrace:\n");
    backtrace_recur(fp);
}
~~~

<img src="D:\截图\image-20210716202241974.png" alt="image-20210716202241974" style="zoom:50%;" />





## 2.3  Alarm

~~~
这个实验中，你需要好给xv6增加一个特性：随着进程使用CPU，周期性地提醒进程。这可能对于那些想要限制使用CPU时间的进程以及那些想要正常使用CPU时间的同时做周期性活动的进程非常有用。更宽泛地说，你将会实现一个用户级中断/错误的处理器原型；比如，你可以在应用中用一些类似于处理页错的方法。如果通过了alarmtest和usertests，那么你的处理方法就是正确的。
~~~

**test0: invoke handler**



> 需要在struct proc中添加几个成员

~~~c
  int ticks;                   // Alarm interval,if the ticks = 0,there is no alarm call
  int tickLeft;                // The left time to next call
  void (*handler)();           // Alarm handler
~~~



> 在分配proc的地方，增加初始化的内容。

~~~c
  // Alarm part
  p->ticks = 0;
  p->handler = 0;
  p->tickLeft = 0;
~~~



> 添加sys_sigalarm函数

~~~c
uint64
sys_sigalarm()
{
    int tick;
    uint64 handler;
    struct proc* p = myproc();
    if(argint(0, &(tick)) < 0)
        return -1;
    if(argaddr(0, &handler) < 0)
        return -1;
    p->ticks = tick;
    p->tickLeft = tick;
    p->handler = (void (*)())handler;
    return 0;
}
~~~



> 周期函数是按照时钟中断的次数来决定是否执行的。所以需要去修改trap.c/usertrap()里面的内容。
>
> 修改比较简单，问题是如何去调用handler。
>
> 通过risc-v book可以知道，后面回到用户空间后，是从 proc->trapframe->epc 开始执行的。所以直接修改 epc 为handler所指的地址即可。



这边想了很久，到死胡同了。我本来打算把 epc 减小，然后在其所指的位置加入汇编指令去执行handler，然后遇到两个问题：

+ auipc 和 jalr 命令的数字值不会写，也不太会用。
+ 大概明白之后，也不怎么能用，因为jalr用的是相对地址，我不可能在编写的时候就知道他们的相对地址。所以得用绝对地址，如果用绝对地址，我就需要一个寄存器去存handler的地址，然后好像也没有可以借用的寄存器。



**为什么可以直接把handler写入到 epc 呢？**

等价于 函数调用完毕之后如何返回caller呢？

~~通过 ra寄存器，所以可以知道 在trap发生的时候pc会写入到ra中。~~

现在问题就是运行完handler之后，无法回到原指令，会在test1/2上处理该问题。



~~~c
// 修改trap.c/usertrap()
// give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
  {
      if(p->ticks != 0){
          p->tickLeft--;
          if(p->tickLeft == 0){
              p->tickLeft = p->ticks;
              p->trapframe->epc = (uint64 )p->handler;
          }
      }
      yield();
  }
~~~



**test1/test2(): resume interrupted code**

> 目前问题是程序只能运行到handler，然后就会出错。所以我们需要做一个系统调用函数来处理这个问题。

根据test0，我们已经知道了 在进入trap之后，返回的指令是epc所指的指令，所以我们可以考虑运用这个特点返回原指令。

显然在系统调用sigreturn中，我们需要完成将原指令处的寄存器内容都返还给他，且之后我们需要开始执行原指令。所以我们需要把在执行周期函数前原指令处的trapframe记录下来，并且在 sigreturn中写入 trapframe。

> 给struct proc 增加成员

~~~c
  uint64 total_ticks;          // 从上一次调用开始走过的时间
  uint64 alarm_interval;       // alarm 时间
  void (*handler)();           // Alarm handler
~~~

> 调用handler之前备份 trapframe

~~~c
  // give up the CPU if this is a timer interrupt.
    if(which_dev == 2){
        if(p->alarm_interval > 0){
            p->total_ticks--;
            if(p->total_ticks == 0){
                    p->savetrapframe = *p->trapframe;	//把此时的trapframe备份
                    p->trapframe->epc = (uint64)p->handler;
            }
        }
        yield();
    }
~~~

> 在sigreturn的时候把之前备份的内容重新写入trapframe，因为副本中的epc刚好是原指令的地址，所以只需要这样一条语句即可，不需要再修改epc了。
>
> 还有就是 sigreturn中会重置 total_ticks，而不是在 trap中，是因为不把执行handler的时间算在里面。

~~~c
uint64
sys_sigalarm(void)
{
    struct proc *p = myproc();
    int interval;
    uint64 handler;
    if(argint(0, &interval) < 0)
        return -1;
    if(argaddr(1, &handler) < 0)
        return -1;
    p->alarm_interval = interval;
    p->total_ticks = interval;
    p->handler = (void (*)())handler;
    return 0;
}


//把记录的trapframe副本内容复制到现在的trapframe上面。
uint64
sys_sigreturn(void)
{
    struct proc *p = myproc();
    *p->trapframe = p->savetrapframe;
    p->total_ticks = p->alarm_interval;
    return 0;
}
~~~



**还需要看一下视频的内容，尤其是栈帧部分，目前还不太理解**
