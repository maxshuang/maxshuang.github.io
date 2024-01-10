---
layout: post
title: Linux Kernel-Manage Process
subtitle: picture from https://www.pexels.com/search/wild%20animals/
author: maxshuang
categories: Linux-Kernel
banner:
  image: /assets/images/post/linux-kernel-manage-process/pexels-pixabay.jpg
  opacity: 0.618
  background: "#000"
  height: "70vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Linux-Kernel CPU
---

在之前的博客中，我们一直在介绍内存相关的话题，因为我觉得内存是一切逻辑和数据的载体，<font size=3>先搞清楚事物的存在形式，再讨论事物的组织形式和运作形式，会更容易理解完整的流程</font>。
所以我们从最核心的内核加载出发，介绍到物理内存页的管理和小内存分配，再到最后内核给业务实体提供的抽象概念：进程线性地址空间。关于内存的一些核心概念我觉得大致上介绍完了，
注意实际实现还是很复杂的，需要考虑效率上的诸多细节。

这篇开始我们专注于 CPU 相关的概念，也就是事物是如何组织和运行的。

在开始之前，让我们再次在脑海中想象下操作系统这艘航母是如何轰鸣启动的。
1. 在硬件通电，用户按下电源键的一瞬间，CPU 硬件单元如同蒸汽机被灌入第一缕蒸汽一般开始跃动。首先它会读取 ROM 内存特定位置存放的
BIOS 工具箱。
2. 进而通过 BIOS 提供的磁盘操作能力开始加载操作系统文件映像中的 bootloader，并将内核加载并解压到物理内存中。然后，CPU 开始执行内核定义的初始化逻辑，通过感知周边硬件系统结构，比如
CPU 核数和物理内存大小，初始化对应的管理结构，并定义基础服务能力，比如物理页管理和小内存块分配管理等。
3. 最后，定义了基础能力之后，内核需要管理需要执行的逻辑流，其中比较重要的是定义一个内核自己的空闲执行流和一些内核管理工作相关的执行流，这样 CPU 就可以在不同的执行流之间轮流切换，
保证不同的执行流逻辑都在稳步推进。

这样整个操作系统就开始运行起来了，而执行流就保存在不同的进程中。

## 进程
进程是操作系统中独立实体抽象的概念，它包括一系列关系密切的所属资源(状态)和代码逻辑。所属资源包括申请的内存资源、文件资源和网络资源等，代码逻辑就是我们从 main 开始的业务逻辑。

为了进一步理解进程概念，让我们进一步关注进程的存在形式。业务编程的时候，我们关注的是从 main 开始的代码逻辑，关注的是堆栈内存使用以及数据结构算法的设计，看起来这些概念和进程概念有点关系不大，虽然
在定义上我们还是会牢记我们是在一个进程中，除非我们有意调用 fork 函数创建一个新的进程。

那从业务角度出发的相关概念和进程概念的契合点在哪里呢？

在于看不到的工具---编译器，链接器和装载器。

我们在编辑器中根据不同语言编译器的约定，使用固定的开始函数，比如 main，和符合编译器的语法实现我们的逻辑，编译器将我们编写的高级语言根据语言特性展开成了
CPU 可执行的机器代码。同时，更重要的是，将我们编写的逻辑按照可执行文件规范组织成了结构型的可执行文件。这些可执行文件的格式被内核识别，比如根据可执行文件头部的元信息可索引到从第 N 个字节开始的字节码就是
执行逻辑的开始。通过这种方式，当我们在 shell 中执行这个可执行文件时，内核可以动态分配出一个进程结构体 task struct，并关联可执行文件到这个结构体中，按需读取可执行文件的内容，比如读取硬盘中代码逻辑到内存的
task struct 中，以便 CPU 执行。

## 进程创建
进程创建的 2 大核心工作：
1. 内核分配 task struct 结构体
2. 关联对应的可执行文件，从而获取到代码段和数据段  
![linux-process-descriptor](/assets/images/post/linux-kernel-manage-process/linux-process-descriptor.png)

在上面的进程描述符 task struct 中，我们可以看到几个核心的字段：
1. thread_info: 进程中执行流上下文，其实就是 CPU 执行时依赖的各种寄存器值。当 CPU 在不同执行流间切换时，本执行流的当前执行状态被保存在这里，用于后续恢复执行流。
2. run_list: 系统可运行队列链接字段，通过这个字段加入系统维护的可运行队列链表中。
3. mm_struct: 进程线性地址空间相关信息，包括抽象出来的各个线性区和进程页表。
4. real_parent/parent: 进程亲属关系，这里是父子关系，还有一些维护 sibling 关系的字段。
5. fs_struct/files_struct: 文件系统信息，包括打开了哪些文件，读写文件位置和权限等信息。
6. signal_struct: 信号机制相关信息

### 相关系统调用
进程描述符 task struct 是由内核维护的，所以进程创建也是通过内核提供的系统调用实现的，用户只需要将可执行文件的路径传递给系统调用即可，这也是我们平时在 shell 中执行程序时候指定相对路径或者绝对路径时做的事情。

内核提供 3 个系统调用给用户创建进程：
1. clone()
2. fork()
3. vfork()

#### 进程、轻量级进程和线程
在继续讲这几个系统调用之前，我们先插入关于 linux 中进程、轻量级进程和线程的概念。
* 进程：独立实体的资源集合，既包括执行逻辑也包括相关的资源。
* 线程：操作系统中最小的执行单元，多个线程共享进程资源。我们进一步了解线程的概念，每个进程中至少包括一个线程，因为进程也包括了执行逻辑。仔细想线程的概念，你会觉得为什么不可以直接使用进程的概念实现线程呢？毕竟线程
在定义上是进程的一个子集。答案是可以，但是独立出线程概念的原因是希望能共享大部分的进程资源，只独立出执行逻辑。这里是基于效率的考虑，如果直接用进程的概念替代线程，那进程 A 打开的文件，进程 B 无法从 A 读写的位置开始读写。
另外一点需要注意的是，线程在某些效率场景下也希望能有自己的少量的数据，避免数据竞争，这就是线程数据。
* 轻量级进程：进程和线程都是抽象概念，不同操作系统的实现不同。对于 linux，它使用轻量级进程来实现线程，这是一种实现方式，实现原理就是生成线程时，也创建新的进程描述符 task struct，但是新进程描述符直接 shadow copy 
原来进程描述符中的 mm_struct、fs_struct、files_struct 等的指针，不进一步拷贝指向的内容。通过这种方式，实现执行流隔离但是进程资源不隔离。

解释完 linux 轻量级进程的概念之后，回到系统调用中，其中 clone 是个通用的系统调用，通过传入标志控制创建新进程还是创建新的轻量级进程。
fork 和 vfork 都是调用 clone 实现的功能。fork 创建的是新子进程，指向父进程的状态但是不共享；vfork 创建的是新进程，一般和 exec 配合调用执行新的可执行文件。

我们仔细看下 clone 系统调用对应的库函数 [clone](https://www.man7.org/linux/man-pages/man2/clone.2.html), 有几个细节值得我们注意：
```
/* Prototype for the glibc wrapper function */

#define _GNU_SOURCE
#include <sched.h>

int clone(int (*fn)(void *_Nullable), void *stack, int flags,
         void *_Nullable arg, ...  /* pid_t *_Nullable parent_tid,
                                      void *_Nullable tls,
                                      pid_t *_Nullable child_tid */ );

/* For the prototype of the raw clone() system call, see NOTES */

#include <linux/sched.h>    /* Definition of struct clone_args */
#include <sched.h>          /* Definition of CLONE_* constants */
#include <sys/syscall.h>    /* Definition of SYS_* constants */
#include <unistd.h>

long syscall(SYS_clone3, struct clone_args *cl_args, size_t size);

Note: glibc provides no wrapper for clone3(), necessitating the
use of syscall(2).
```

1. fn 函数指针和 stack 用户态堆栈  
创建的进程或者轻量级进程的执行逻辑都从 fn 函数开始，fn 函数返回时执行流结束，fn 函数返回值是子进程的退出码。  
不管是何种类型子进程，由于是独立的执行流，所以要通过 stack 指定新的用户栈，而新进程的进程内核栈(概念在后一篇用户态和内核态的文章中介绍)在内核初始化新 task struct 再分配。

2. flags 创建标志  
4 字节标志字段，最低字节表示子进程结束时给父进程发送的信号。高 3 字节用作其他标志，有几个创建标志对理解不同的概念有帮助，如下：  

| Name         | Meaning |
|--------------|---------|
| CLONE_VFORK  | 设置后，调用进程会被挂起直到子进程通过结束自己 exit 或者执行一个新程序 exec 释放虚拟内存空间|
| CLONE_VM     | 设置后，父子进程共享线性地址空间，是个轻量级子进程；不设置则是创建子进程|
| CLONE_FILES  | 文件描述符资源共享。设置后，父子进程操作文件描述符会互相影响 |
| CLONE_FS     | 文件系统信息共享，包括工作目录等。设置后，父子进程操作文件系统信息会互相影响 |
| CLONE_PARENT | 不设置则生成一个子进程；设置后，生成子进程的父进程设置成调用进程的父进程，相当于生成了自己的 sibling 进程|       |

* [fork](https://www.man7.org/linux/man-pages/man2/fork.2.html)
创建子进程。内部调用 clone，flags 标志只指定最低直接是 SIGCHILD, 高 3 字节全部清 0。子进程使用写时复制技术全部浅拷贝所有父进程的指针引用，真正写时再申请内存块拷贝原始内容。

* [vfork](https://www.man7.org/linux/man-pages/man2/vfork.2.html)
创建子进程。内部调用 clone，flags 标志只指定最低直接是 SIGCHILD, 高 3 字节设置 CLONE_VM|CLONE_VFORK。语义上共享父进程的内存空间，挂起父进程直到子进程调用 exit 或者 exec。
由于会挂起父进程，所以一般在调用 fork 之后调用 exec 执行新的可执行文件，语义上就是创建一个全新的进程。

* [exec](https://man7.org/linux/man-pages/man3/exec.3.html)
额外讲下 exec 库函数，主要是在当前进程空间下执行一个新的进程映像，相当于创建了一个完全不相关的新进程。

最后讲下线程的创建，有一些符合 POSIX 标志的系统库可以创建线程，比如 pthread_create。在内部实现上，调用 clone 指定 CLONE_VM 即可创建轻量级进程，通过共享父进程内存资源的形式实现了线程的概念。

#### TLS(Thread Local Segment)
线程是独立的执行流，为了满足执行流要求，需要拥有自己的用户态栈、进程内核栈和 TCB(Thread Control Block)，存放函数调用参数、临时数据和线程执行时硬件上下文。linux 使用轻量级进程实现线程，
其中共享了主进程的内存资源。

为了避免数据竞争，一类经典的场景是线程也拥有自己的数据，不同线程更新自己的数据备份，如下：
```
#include <stdio.h>
#include <threads.h>  // For C11 thread support

// Define a thread-local variable
_Thread_local int tls_var;

void threadFunc() {
    tls_var = 11;
    printf("Thread-local variable value: %d\n", tls_var);
}

int main() {
    // Set the value of the thread-local variable
    tls_var = 10;

    // create a new thread
    pthread_create(threadfunc, ...);

    // sleep and wait the other thread runs, access and print the value of the thread-local variable
    sleep(3);
    printf("Thread-local variable value: %d\n", tls_var);
    
    ... 
    
    return 0;
}
```
在这个 demo 中，我们使用 _Thread_local 定义了一个线程本地变量，然后我们在主线程和子线程中分别更新 tls_var 的值。
可以发现主线程打印出来的 tls_var 值还是 10，线程本地变量 tls_var 在不同线程对应不一样的副本。

另一个有名的线程本地例子是 errno 的改造，原来在定义的是全部变量，到了多线程环境下返回的是 thread local errno。

线程本地变量的机制和 TLS 有关，不同架构下的实现也不同，其原理涉及编译器、链接器、动态链接器、内核和语言运行时。有兴趣的同学深入的同学可以参考以下博客，  
[A Deep dive into (implicit) Thread Local Storage](https://chao-tic.github.io/blog/2018/12/25/tls)  
[all-about-thread-local-storage](https://maskray.me/blog/2021-02-14-all-about-thread-local-storage)

### 进程标识和进程状态
识别不同的进程就如果区分不同的人，进程描述符 task struct 通过一个 4 字节的 pid 值区分不同进程。可以看到，这种分配方式下，内核中可维护的进程个数上限是 2^32(65535)。内核递增得为新进程分配 pid 号，
如果超过了上限，就再从头找空闲的 pid 号。为了快速找到可用的 pid 号，内核维护了一个 32k bit 的 bitmap 表示可用关系，只要对应位置上的 pid 号被使用了就设置成1，否则是 0。

进程按照其运行状态被标识成不同的状态：
* 可运行状态：TASK_RUNNING，表示可以被执行。可能正在被执行或者在 running 队列中等待被调度执行。
* 暂停状态：TASK_STOPPED，表示进程被暂停，进程描述符挂入暂停队列（有问题描述）。通过给进程发送 SIGSTOP、SIGTSTP 等信号让进程进入暂停状态，通过给进程发送 SIGCONT 信号可以唤醒暂停进程进入 TASK_RUNNING 状态。
* 可中断的等待状态：TASK_INTERRUPTIBLE，表示进程处于可中断的休眠状态，挂入对应事件的等待队列中。当系统资源可用或者被传递一个信号时可被唤醒，状态转移到 TASK_RUNNING。
* 不可中断的等待状态：TASK_UNINTERRUPTIBLE，和 TASK_INTERRUPTIBLE 类似，区别在于信号无法唤醒。挂入对应事件的等待队列中。
* 僵死状态：EXIT_ZOMBIE，表示子进程终止了，但是子进程资源还没有被父进程回收。
* 僵死撤销状态：EXIT_DEAD，表示子进程终止且被父进程回收，防止回收子进程资源时出现多个线程同时回收子进程资源的数据竞争场景。

更复杂更细节的进程状态转移可以关注 [geeksforgeeks](https://www.geeksforgeeks.org/process-states-and-transitions-in-a-unix-process/) 上的相关内容。  
![process-state-transitions](/assets/images/post/linux-kernel-manage-process/process-state-transitions.jpg)

### 进程资源限制
进程是内核给用户提供的资源集合的抽象，所以需要通过限制进程使用资源的方式保护整个操作系统，避免个别进程恶意申请资源导致其他正常进程无法正常申请资源。

进程资源限制存放在 task struct->signal->rlim 字段中，是一个 rlimit struct 的数组。
```
struct rlimit {
    unsigned long rlim_cur; // 字段当前值
    unsigned long rlim_max; // 字段允许的最大值
}
```

几个常见的资源类型如下：

| Name         | Meaning |
|--------------|---------|
| RLIMIT_CORE  | 进程 core dump 之后的转储文件大小 |
| RLIMIT_CPU   | 进程使用 CPU 最长时间 |
| RLIMIT_DATA  | 堆大小最大值 |
| RLIMIT_STACK | 栈大小最大值 |
| RLIMIT_NOFILE| 打开文件描述符的最大数 |

### 内核线程和内核代码段
除了进程、线程和轻量级线程这些概念之后，内核还有个内核线程的概念。内核线程只运行在内核态，只涉及操作内核相关的资源，只访问高于 3G 的线性地址空间。在实现上是个进程，但是没有用户线性地址空间相关的 mm_struct。  
内核中比较知名的内核线程有：
* 进程 0：也称为 idle 进程或者 wrapper 进程，是内核的第一个进程，也是所有进程的祖先。进程 0 的进程描述符是静态分配在内核并随内核一起初始化。进程 0 的核心逻辑是 cpu_idle 函数，循环执行 hlt 指令，暂停 CPU 直到下个外部中断。
当系统中没有任何 TASK_RUNNING 状态的进程时选择进程 0 执行。每个 CPU 都有一个进程 0。
* 进程 1：也称为 init 进程，由进程 0 创建，负责内核初始化。
* ksoftirqd: 运行 tasklet，处理软中断，下篇文章讲内核态切换会涉及到。
* kswapd: 执行内存回收，在内存紧张时讲长久不访问的物理内存页内容写入磁盘页，释放可用物理内存页。

和内核线程类似的概念就是内核代码段，与内核线程这种拥有资源和生命周期的实体对比起来，内核代码段就只是一段代码，执行完之后就结束了，内核不需要创建 task struct 维护其相关资源和生命周期，比较典型的例子就是中断处理例程。

## 进程管理
进程创建后，就需要维护进程的不同关系，比如如何维护所有进程，如何维护可运行进程，当给线程组的领头进程发终止信号时如何通知线程组的其他线程等等场景。
### 进程链表
所有进程描述符都通过 task_struct 中的 list_head 类型字段形成了双向链表，从而关联操作系统中所有的进程。  
进程链表的头是上面提到的进程 0 的进程描述符。  
![process-list](/assets/images/post/linux-kernel-manage-process/process-list.png)

### 运行队列
为了快速获取可运行的进程，内核还涉及了可运行队列，将所有处于 TASK_RUNNING 状态的进程描述符都链接进可运行队列。每个 CPU 都有自己的 runqueue，并且一个进程描述符只属于一个 runqueue。  
runqueue 中核心的字段是 prio_array_t 结构体，里面维护了 140 个优先级的运行队列，以便 CPU 能在常数时间内获取到最佳的可运行进程。
```
struct prio_array_t {
    int nr_active;                  // 进程描述符数量
    unsigned long[5] bitmap;        // 优先权位图，当对应优先级队列不为空时设置成 1
    struct list_head[140] queue;    // 140 个优先队列头节点
}
```
由于优先级队列的内容我们放到进程切换和进程调度的文章中讲，这里只简单提及进程拥有不同的优先级，放入了不同的优先级队列中维护。

内核没有为 TASK_STOPPED、EXIT_ZOMBIE 和 EXIT_DEAD 状态维护专门的队列，因为他们要么数量少，要么可以通过父子关系等索引到。

### 等待队列
等待队列是除了运行队列之外的核心队列，在中断处理、进程同步(部分是锁机制，在锁相关文章中介绍)和定时场景下使用。当进程等待某些资源时，比如等到定时间隔到期，等待文件网络可写等，内核就把进程描述符挂入对应的等待队列中，让出 CPU 使用权，
等待特定事件发生时再把进程描述符从等待队列中移动到运行队列中。  
等待队列实现上也是个双向链表，
```
struct _ _wait_queue_head {
    spinlock_t lock;            // 自旋锁，保护等待队列
    struct list_head task_list; // 双向链表头
};

// 等待队列中的单个实体
struct _ _wait_queue {
 unsigned int flags;
 struct task_struct * task;   // 关联的进程描述符，标识具体等待事件的进程
 wait_queue_func_t func;      // 特定事件发生唤醒操作
 struct list_head task_list;  // 双向链表节点
};
```
为了避免过于深入等待队列的细节，我们用一个简单的例子解释下进程进入等待队列和出等待队列的流程，
```
// 当前进程在某个事件上休眠
void sleep_on(wait_queue_head_t *wq)
{
    wait_queue_t wait;
    init_waitqueue_entry(&wait, current); // 初始化 wait queue 中单个实体
    current->state = TASK_UNINTERRUPTIBLE;  // 设置进程状态
    add_wait_queue(wq,&wait);               // 加入等待队列
    schedule( );    // 进程切换，因为当前进程还在等待事件。从这里开始 CPU 已经去执行其他进程逻辑了
    remove_wait_queue(wq, &wait);   // 进程切换回来，所以等待的事件已到达，被内核唤醒了，将自己从等待队列中删除。
}

// 事件到达是内核使用 wakeup 唤醒等待进程
void wake_up(wait_queue_head_t *q)
{
    struct list_head *tmp;
    wait_queue_t *curr;
    // 这是一个非互斥等待队列，唤醒全部的进程
    list_for_each(tmp, &q->task_list) {
        curr = list_entry(tmp, wait_queue_t, task_list);
        if (curr->func(curr, TASK_INTERRUPTIBLE|TASK_UNINTERRUPTIBLE,
            0, NULL) && curr->flags)  // 这里将进程状态修改成 TASK_RUNNING, 并加入运行队列，等待被调度执行
        break;
    }
}
```
理解进程等待和唤醒的流程就要理解 schedule, 进程等待事件未达到就不能直接返回用户态，要将自己注册到等待队列中，然后主动让出 CPU 时间。当等待事件发生时，内核会负责将进程设置成可运行状态，并加入运行队列中再次运行。
也就是说，schedule 返回时大部分情况下意味着等待的事件已经到达了，内核可以继续执行一些操作，然后返回用户态了。

从上面的例子也可以注意到，内核对于唤醒函数 wakeup 可以有不同的设计，对于互斥进程可以只唤醒一个，对于非互斥进程可以全部唤醒。这个就和我们平时会接触到的惊群效应有关系了。

### 进程关系
从进程组织的角度看，进程间存在着亲属关系和非亲属关系：
1. 父子关系：进程 A 调用 clone/fork 创建了进程 B，则进程 A 是进程 B 的父亲进程，进程 B 是进程 A 的子进程。进程描述符中 real_parent、parent 和 children 指向这样的关系。
2. 兄弟关系：同一个父进程创建的所有子进程都是兄弟关系，进程描述符中 sibling 作为双向链表节点指向这样关系。
3. 线程组 leader
4. 进程组 leader
5. 登录会话组 leader

* 对于 1 和 2 这种亲属关系，其关系图如下：  
![process-parenthood](/assets/images/post/linux-kernel-manage-process/process-parenthood.png)
* 对于线程组，很好理解。我们在一个进程中创建了多个子线程，多个子线程和主线程就组成了线程组，子线程在 linux 中都是轻量级进程的概念。线程组的意义在于，对线程组中其中一个子线程的操作可能影响整个线程组。比如子线程 A 受到 SIGSEG 的信号，此时整个线程组中的轻量级进程都要退出。
进程组 leader 是 main 进程。
* 对于进程组，它的概念源于 job。一个 job 中含有多个独立逻辑的进程，比如 shell 的前后台进程。我们希望通过进程组的概念能控制整个进程组的销毁等操作。
* 对于登录会话组，它的概念源于多用户。用户登录进 linux 后会创建一个对应的登录会话组，该用户创建的所有的进程都会关联到该登录会话组，当用户退出或者用户注销时可以将相关的所有进程销毁。登录会话组另外一个重要作用是限制用户使用资源，
通过这种方式避免单个用户挤占系统资源。

用户态下，一个经典需求是使用 kill($pid) 函数给指定进程发送信号，使用进程 pid 号标识进程。如果通过遍历进程链表的方式获取 pid 号对应的进程描述符 task struct 或者获取进程组 leader 是 pid 的所有进程，效率上就太低了。

内核使用 hash table 保存 pid 和进程描述符的关系。同样的，相同进程组的进程或者相同线程组的轻量级进程会映射到 hash table 的相同位置，并通过搜索冲突链表的方式找到 leader pid 相同的节点，形成双向链表。具体如下图所示：  
![process-pid-hash](/assets/images/post/linux-kernel-manage-process/process-pid-hash2.png)  
[source](https://excalidraw.com/#json=C3kc-JYQEl8zNDG-mrlYT,zh_M9dPVCZeUi9NENACikw)

上图中有几点是我们需要关注的：
1. pid_hash 结构体：该结构体中包含了 4 个 hash table，分别为 pid，pgid，tgid 和 sgid 提供 pid 到 task struct 的索引。
2. 对于每个 hash table，不同的 pid 可能映射到 hash table 的相同 slot，使用冲突链表法处理冲突，相同位置映射通过单向链表互相关联。冲突链表的节点字段是 pid_chain。
3. 对于 pgid、tgid 和 sgid hash table，它们的相似点在于除了通过 pid 索引进程描述符之外，还要找过所有具有相同值的进程描述符，比如线程组 leader pid 都是 4351 的所有进程。所以我们可以看到，hash table 除了冲突链表之外，还存在其他链表, 相同线程组的双向链表节点字段是 pid_list。

以上就是内核中进程关系相关的核心概念。

## 销毁进程
销毁进程是个自然的概念，在用户逻辑执行完之后进程就会被销毁，这是我们在写一些 demo 时遇到的场景。另外一些重要场景是进程执行过程中遇到了不可恢复的异常或者用户主动调用了 exit，此时进程也会被销毁。

销毁进程对于内核而言，就是将进程描述符从进程链表中移除，然后回收进程描述符关联的系统资源，比如关闭打开文件等，最后就是回收分配给进程描述符本身的小内存块。至此，不管进程中用户逻辑执行到哪个位置，进程都已经被销毁了，因为进程实体被销毁了。

用户可以显示调用 exit() 函数终止进程，C 编译器也会在每个 main 函数结尾插入 exit() 触发内核回收进程资源。

Linux 2.6 提供了 2 个相关的内核调用：
1. exit_group：终止整个线程组的所有进程，exit() C 函数内部调用了这个系统调用。这个符合我们的语气，主线程如果被销毁了，子线程也会被销毁。
2. _exit: 终止单个进程，也可以是单个线程和单个轻量级进程。pthread_exit C 函数内部调用的这个系统调用。

## 总结
这篇文章通过深入部分细节的方式描述了内核中进程是如何管理的，包括创建、组织和销毁，希望通过对细节的描述能让读者对从写用户逻辑到用户逻辑被内核管理这个流程有个直观感受。