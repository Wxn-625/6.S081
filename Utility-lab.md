# 1 操作系统接口

> OS是将一个计算机的资源分享给所有进程，并提供了比单独硬件提供的更有用服务，因为OS抽象了底层的硬件。
>
> OS通过接口来给user提供服务。当user需要OS服务时，直接使用系统调用。
>
> kernel通过CPU的硬件保护机制来确保进程的访问权限。

## 1.1 进程与内存

~~~c
int fork() Create a process, return child’s PID.
int exit(int status) Terminate the current process; status reported to wait(). No return.
int wait(int *status) Wait for a child to exit; exit status in *status; returns child PID.
int kill(int pid) Terminate process PID. Returns 0, or -1 for error.
int getpid() Return the current process’s PID.
int sleep(int n) Pause for n clock ticks.
int exec(char *file, char *argv[]) Load a file and execute it with arguments; only returns if error.
char *sbrk(int n) Grow process’s memory by n bytes. Returns start of new memory.
int open(char *file, int flags) Open a file; flags indicate read/write; returns an fd (file descriptor).
int write(int fd, char *buf, int n) Write n bytes from buf to file descriptor fd; returns n.
int read(int fd, char *buf, int n) Read n bytes into buf; returns number read; or 0 if end of file.
int close(int fd) Release open file fd.
int dup(int fd) Return a new file descriptor referring to the same file as fd.
int pipe(int p[]) Create a pipe, put read/write file descriptors in p[0] and p[1].
int chdir(char *dir) Change the current directory.
int mkdir(char *dir) Create a new directory.
int mknod(char *file, int, int) Create a device file.
int fstat(int fd, struct stat *st) Place info about an open file into *st.
int stat(char *file, struct stat *st) Place info about a named file into *st.
int link(char *file1, char *file2) Create another name (file2) for the file file1.
int unlink(char *file) Remove a file.
~~~



**Fork()**

>  fork会创建一个子进程，并返回两次，父进程返回子进程ID，子进程返回0.
>
> 子进程会直接继承父进程的几乎所有信息。主要有：文件描述符、用户ID、进程组ID、信号屏蔽、环境、资源限制。



**exit(int status)**

> exit会使得进程停止执行，并释放资源包括内存和打开文件。status 的值会传给父进程，0表示正常退出，1表示非正常退出。父进程可以通过wait或者waitpid 回收子进程，并且获得其status



**exec(char* filename,argv)**

> exec系统调用会加载新的内存内容去替代掉原内存的内容，包括指令，数据等。这表示该程序会去执行新的文件，不会再return原内容。一般argv中的第一个参数内容就是filename。
>
> **为什么不把fork和exec 合并到一个函数？**：
>
> + 因为fork完不会直接复制所有父进程的内容，如果有不同的需要时，才会复制（**写回法**）。
> + 考虑到重定向需要在exec之前改变FD。如果直接使用 forkexec 会导致每个 forkexec中都需要写入重定向的信息，这会增加很大的工作量，复用性不好。
>
> **exec之后 打开的文件描述符表不会变，这使得I/O重定向变得容易**

~~~c
char* argv[3];
argv[0] = "echo";
argv[1] = "hello";
argv[2] = 0;
exec("/bin/echo", argv);
printf("exec error\n");
~~~





## 1.2 I/O和文件描述符

> Linux把很多东西都抽象成了文件，包括 ：文件、文件夹、管道、设备、socket等，他们的共同点是 一串bit流。这样就可以有一个统一的接口去实现他们的处理。
>
> 文件描述符是每个进程打开的文件所对应的int数值。每个进程都独自拥有一个描述符表。
>
> **子进程会继承父进程的一份描述符表，他们指向同一个文件表项（file对象），file对象中保存了引用次数和offset**
>
> 详情见 csapp/proxy-lab.md
>
> 默认 0、1、2分别表示标准输入、标准输出、标准错误输出

<img src="D:\截图\image-20210701143359106.png" alt="image-20210701143359106" style="zoom:67%;" />

### 1.2.1 Redirect



**read(fd,buf,n)**

> read 会从fd中读取最多n byte的数据到buf中，并返回读取的byte数。
>
> 每个fd所指的file中都有一个offset，read每次都是从这个offset开始读，所以如果子进程开始读也是从父进程的offset开始读的。



**write(fd,buf,n)**

> write从buf中复制n byte的数据到fd中，并返回写入的byte数。写入的起始点也是offset



**Example：cat**

~~~c
/* shell part */
char* argv[2];
argv[0] = "cat";
argv[1] = 0;
if(fork() == 0) {
close(0);
open("input.txt", O_RDONLY);
exec("cat", argv);
}

/* cat part */
void
cat(int fd)
{
  int n;

  while((n = read(fd, buf, sizeof(buf))) > 0) {
    if (write(1, buf, n) != n) {
      printf(1, "cat: write error\n");
      exit();
    }
  }
  if(n < 0){
    printf(1, "cat: read error\n");
    exit();
  }
}

int
main(int argc, char *argv[])
{
  int fd, i;

  if(argc <= 1){
    cat(0);
    exit();
  }

  for(i = 1; i < argc; i++){
    if((fd = open(argv[i], 0)) < 0){
      printf(1, "cat: cannot open %s\n", argv[i]);
      exit();
    }
    cat(fd);
    close(fd);
  }
  exit();
}
~~~

> shell 中的子程序把 FD = 0关闭了，然后用“input.txt"去填充 FD0.
>
> 子程序中cat函数从 FD = 0 中读数据然后输出到FD = 1（标准输出）直接表现在terminal上面。因为子进程的FD = 1继承的shell，所以他们是同一个文件描述符。





**dup(fd)**

> 复制fd，返回一个新的nfd，该nfd与fd共享同一个file对象。





## 1.3 Pipe

> pipe是一个小的内核buffer，对于进程来说是一对文件描述符，一个用作读一个用作写。

~~~c
int p[2];
char* argv[2];
argv[0] = "wc";
argv[1] = 0;
pipe(p);
if(fork() == 0) {
close(0);
dup(p[0]);
close(p[0]);
close(p[1]);
exec("/bin/wc", argv);
} else {
close(p[0]);
write(p[1], "hello world\n", 12);
close(p[1]);
}
~~~



**如果没有数据可读，在pipe上面read 会阻塞直到有数据写入，或者所有written相关的fd都被关闭了。后者会返回0。

**所以对于一个子进程来说关闭写端的pipe是非常重要的，否则可能会出现一直阻塞的情况**



看起来 pipe 和 临时文件好像没什么区别？

+ pipe会自动清理，临时文件需要手动删除
+ 临时文件需要足够的空间，pipe可以随意长度
+ pipe可以并行
+ pipe在blocking read中更有效率？



## 1.4 文件系统



~~~c
chdir("/a");
mkdir("/dir");

/* 将一个dev和两个数字进行关联 */
mknod("/console", 1, 1);
~~~



> 文件名相当于link，底层文件是inode，并不相同，一个inode 可以关联多个link。
>
> link包含文件名，和指向一个inode的引用
>
> inode包含了元数据：文件种类，长度，磁盘上的文职，其上的link数



**fstat*filename)**

> fstat从一个文件名去获得这个inode。

~~~c
struct stat {
int dev; // File system’s disk device
uint ino; // Inode number
short type; // Type of file
short nlink; // Number of links to file
uint64 size; // Size of file in bytes
};
~~~



**link(filename1,filename2)**

> 复制一个指向filename1引用的inode的引用到filename2这个link上
>
> unlink(filename1)：会把一个link删除



**PS：** 其他命令都是fork出的子程序做的，cd是改变shell本身的当前目录





# 2 LAB

## 2.1 sleep

> sleep 很简单，只是对操作的熟悉。
>
> 其实我是看的别人的代码。hint中说要去看sysproc中的sys_sleep，但是看不懂。而且我代码里调用的sleep的源代码 也不知道是哪个。

xv6中有两个sleep：一个是和wakeup一对的sleep，它的作用是让出cpu，进入SLEEPING状态，当被wakeup时，重新变为RUNNABLE，他们的代码在proc.c中

一个是本题中使用的：他的代码在sysproc.c中，他记录当前的ticks（这是一个全局变量，每产生一个定时中断，它就加一），如果过去的ticks没超过n，就调用第一个sleep,而trap中，每一个时钟中断，就增加一个tick，并且唤醒ticks（唤醒之后可能因为没超过而再次sleep，唤醒首先是为了让你检查而不是让你退出循环）， 直到ticks差值超过给定值

原文链接：https://blog.csdn.net/RedemptionC/article/details/106484363

~~~c
int main(int argv,char** argc){
    if(argv < 2){
        fprintf(2,"lack of argument!\n");
        exit(1);
    }
    int l = atoi(argc[1]);
    printf("(nothing happens for a little while) %d\n",l);
    sleep(l);
    exit(0);
}
~~~





## 2.2 pingpong

> 要求做一个pipe的简单应用，没什么难点。
>
> 第一个是使用pipe的时候，记得把不用的fd给关了。
>
> 第二个是程序结束记得退出。

~~~c
int main(){
    int p1[2],p2[2];
    pipe(p1);
    pipe(p2);
    int pid;
    if((pid = fork()) == 0){
        char buf[1];
        close(p1[1]);
        close(p2[0]);
        read(p1[0],buf,1);
        printf("%d: received ping\n",getpid());
        write(p2[1],buf,1);
    }
    else{
        char buf[1];
        close(p1[0]);
        close(p2[1]);
        write(p1[1],buf,1);
        read(p2[0],buf,1);
        printf("%d: received pong\n",getpid());
    }
    exit(0);
}
~~~





## 2.3 primes

> 就是通过提供的方法去晒出素数。做的时候遇到一些问题：
>
> + 是要处理fd，使得迭代中可以有一个统一的表示。
> + 为了避免打印出现混乱，有些子程序比父程序打印地还快，需要用到 wait 函数。
>
> + fork的传递，一开始想搞递归，但好像发现不太行，然后用的迭代法。其中我需要去确定每个进程的除数，一开始是考虑通过父进程传递，这会使得代码比较杂乱，不优雅。
> + 迭代的中，开头的情况是和后面有所不一样的，所以需要特殊处理。这也使得代码非常乱。
> + Solution 2 中通过在开头把他们统一，然后迭代的代码中也就是统一的了。

~~~c
int main(){
    int nowid = 2;
    int pid;

    for(int i = 2;i <= 35;i++){
        if(nowid != i)
            continue;
        if(i == 2){
            int p1[2];
            pipe(p1);
            if((pid = fork()) != 0){
                int nextid = nowid + 1;
                write(p1[1],&nextid,4);

                for(int j = 2;j <= 35;j++){
                    if(j == nowid){
                        printf("prime %d\n",j);
                    }
                    else if(j > nowid && (j % nowid != 0))
                        write(4,&j,4);
                }
                close(3);
                close(4);
                wait(0);
            }
            else{
                read(3,&nowid,4);
                close(4);
            }
        }
        else if(i == 3){
            int p2[2];
            pipe(p2);

            if((pid = fork()) != 0){
                int val;
                int nextid = nowid + 1;
                write(5,&nextid,4);

                while(read(3,&val,4) != 0){
                    if(val == nowid) {
                        printf("prime %d\n", val);
                    }
                    else if(val > nowid && (val % nowid != 0))
                        write(5,&val,4);
                }
                close(3);
                close(4);
                close(5);
                wait(0);
            }
            else{
                close(3);
                close(5);
                dup(4);
                close(4);
                read(3,&nowid,4);
            }

        }
        else{
            int p3[2];
            pipe(p3);

            if((pid = fork()) != 0){
                int val;
                int nextid = nowid + 1;
                write(5,&nextid,4);
                while(read(3,&val,4) != 0){
                    if(val == nowid){
                        printf("prime %d\n",val);
                    }
                    else if(val > nowid && (val % nowid != 0))
                        write(5,&val,4);
                }
                close(3);
                close(4);
                close(5);
                wait(0);
            }
            else{
                close(3);
                close(5);
                dup(4);
                close(4);
                read(3,&nowid,4);
            }
        }
    }
    exit(0);
}
~~~



> Solution 2 中我们的统一部分是，每次进入迭代的程序中，他的打开fd 分别为 3 4 5，其中 4 是该进程需要的读入fd，我们把 3 5 关闭，然后把 4 放到 3 的位置，就可以统一处理了。
>
> 还有一点是每次主程序在循环中只执行一次就会break跳出。

~~~c
int main(){
    int p[2];
    pipe(p);
    for(int i = 2;i <= 35;i++){
        write(4,&i,4);
    }

    dup(4);
    close(4);
    dup(3);

    for(int i = 2;i <= 35;i++){
        close(3);
        close(5);
        dup(4);
        close(4);

        int pt[2],tmp;
        pipe(pt);

        if(fork() != 0){
            while(read(3,&tmp,4) > 0){
                if(tmp == i)
                    printf("prime %d\n",i);
                else if(tmp > i && (tmp % i != 0))
                    write(5,&tmp,4);
            }
            close(5);
            // gurantee a process just exec one time
            break;
        }

    }
    wait(0);
    exit(0);
}
~~~





## 2.4 find

> find 需要对一些文件的数据结构有一定的了解，包括stat, dirent。
>
> 这个小结根据hint 主要参考了ls函数了解了如何处理 stat 和dirent。
>
> 遇到的问题有：
>
> + 照抄了ls的fmtname函数，发现他的bf是静态变量，而且填充时是用 ‘ ’填充，而不是常用的‘\0'。
>
> + 一开始看到 ls中 的dirent 会处理inum = 0 的情况，但是自己并没有去处理，导致de了很久的bug。
>
>   一般inode = 0 表示错误（我查阅资料，感觉 这里的inum 应该是inode。但不知道dirent为什么会读到 0.

~~~c
char*
fmtname(char *path)
{
    static char bf[DIRSIZ+1];
    char *p;

    // Find first character after last slash.
    for(p=path+strlen(path); p >= path && *p != '/'; p--)
        ;
    p++;

    // Return blank-padded name.
    if(strlen(p) >= DIRSIZ)
        return p;
    memmove(bf, p, strlen(p));
    memset(bf+strlen(p), ' ', DIRSIZ-strlen(p));
    return bf;
}

void find(char* filepath,char* name){
    char buf[512],*p;
    struct stat st;
    struct dirent dt;
    int fd;

    if((fd = open(filepath,0)) < 0){
        fprintf(2, "find: cannot open %s\n", filepath);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", filepath);
        close(fd);
        return;
    }


    strcpy(buf,filepath);
    p = buf + strlen(buf);
    *p++ = '/';


    switch (st.type) {
        case T_FILE:
            if(strcmp(fmtname(filepath), name) == 0)
                printf("%s\n", filepath);
            break;

        case T_DIR:
            while(read(fd,&dt,sizeof(dt)) == sizeof (dt)){
                if(dt.inum == 0 ||strcmp(dt.name,".") == 0 || strcmp(dt.name,"..") == 0)
                    continue;

                memmove(p,dt.name,DIRSIZ);
                p[DIRSIZ] = 0;
                find(buf,name);
            }
            break;
    }
    close(fd);
}

int main(int argc,char** argv){
    if(argc < 3)
    {
        fprintf(2,"too few arguments!\n");
        exit(1);
    }

    char name[DIRSIZ + 1];
    strcpy(name, fmtname(argv[2]));
    find(argv[1],name);
    exit(0);
}

~~~



~~~c
struct dirent {
  ushort inum;
  char name[DIRSIZ];
};

struct stat {
  int dev;     // File system's disk device
  uint ino;    // Inode number
  short type;  // Type of file
  short nlink; // Number of links to file
  uint64 size; // Size of file in bytes
};
~~~





## 2.5 xarg

> 不太熟悉这个命令 ，所以去网上查了一下。

xargs（英文全拼： eXtended ARGuments）是给命令传递参数的一个过滤器，也是组合多个命令的一个工具。

xargs 可以将管道或标准输入（stdin）数据转换成命令行参数，也能够从文件的输出中读取数据。

xargs 也可以将单行或多行文本输入转换为其他格式，例如多行变单行，单行变多行。

xargs 默认的命令是 echo，这意味着通过管道传递给 xargs 的输入将会包含换行和空白，不过通过 xargs 的处理，换行和空白将被空格取代。

xargs 是一个强有力的命令，它能够捕获一个命令的输出，然后传递给另外一个命令。

之所以能用到这个命令，关键是由于很多命令不支持|管道来传递参数，而日常工作中有有这个必要，所以就有了 xargs 命令，例如：

~~~c
find /sbin -perm +700 |ls -l       #这个命令是错误的
find /sbin -perm +700 |xargs ls -l   #这样才是正确的
~~~

-n 表示命令在执行的时候一次用的argument的个数，默认是用所有的。



> 首先要搞明白cmd 1中的输出是直接变为 cmd2的参数还是仅放在了标准输入中。
>
> 通过查看 shell的代码，可以知道只是留在了标准输入中，所以说有些命令是无法支持管道的。



**思路**：hint中说明了cmd1 中的输出以'\n'进行分割。所以一次一次读，读到一个\n 就执行一次fork。

~~~c
int main(int argc,char** argv){
    char* outArg[MAXARG];
    char buf[512];
    char* p=buf;

    /* Fixed arguments */
    for(int i = 0;i < argc - 1;i++){
        outArg[i] = argv[i + 1];
    }

    while(read(0,p,1) == 1){
        if(*p == '\n'){
            *p = '\0';
            outArg[argc - 1] = buf;
            if(fork() == 0){
                exec(outArg[0],outArg);
            } else{
                wait(0);
                p = buf;
            }
        }
        else{
            p++;
        }
    }

    exit(0);
}

~~~









## 2.6 shell符号



### 管道 |

把前者本来的标准输出放到管道的输入，cmd2 从管道中读取作为新的输入。



### 输入重定向 <

~~~shell
command   <  /path/to/file
~~~

shell把文件内容从标准输入中读入。



### 输出重定向 >

shell把标准输出的内容输出到文件中
