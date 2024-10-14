---
layout: post
title: Linux Kernel-Lock and Synchronization(Ongoing)
subtitle: picture from https://www.pexels.com/search/wild%20animals/
author: maxshuang
categories: Linux-Kernel
banner:
  image: /assets/images/post/linux-kernel-synchronization/pexels-david-selbert-15578852.jpg
  opacity: 0.618
  background: "#000"
  height: "55vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Linux-Kernel Concurrency Lock
---

## What is Lock

今天这个话题是我想了很长一段时间，它就是**锁与同步**。我写了很久，都不怎么满意，最近一段时间都在面试，刚好有了灵感。

我最早接触"锁"的概念是在接触多线程编程的时候，锁是保护同一份数据被多个并发逻辑同时访问时，保证数据完整的必备工具。

如果不考虑的锁的实现，只考虑锁的原语，我们可以这样想象，为了有序得获取房间中的物资，多个人需要依次顺序对门加锁，拿东西，然后解锁门以便让下一个人进入。这样不管多个人是排成一条队还是散乱得挤在门口，任何时刻都只能有一个人能进入门内。

随着编程得深入，我逐渐接触到各种各种的锁，比如 linux kernel 中的临界区/读写锁/自旋锁，数据库中的 row lock, gap lock, next-key lock 和 intension lock，java 中 ReentrantLock，StampedLock 和 Latch，甚至高并发数据结构中的无锁数据结构。

我逐渐迷失在各种各样的锁中，并且逐渐疑惑了起来，

**锁到底是什么？**

已经存在的这些锁底层是什么？为什么说锁会带来开销？如何创造新的锁？

## The Consensus in Synchronization

并发中的锁首先是一种*共识*，它是在特定 context 下对互斥访问的*共识*。

说得具体一点，在一个生活场景中，10 个人要喝一杯水，但要求只能有一个人能喝到，这个时候我约定先口算出 $19*19$ 的人就可以喝到。在这样的 context 中，我约定了每个人口算出 $19*19$ 的时间是不同的。所以计算行为 "口算出 $19*19$" 在这个场景下就是一种锁，因为它可以在 context 下对互斥访问达成*共识*。

回到我们编程语言中的锁，因为编程语言本身要转换成机器语言，所以编程语言中的锁是基于 kernel context 提供的对互斥访问的*共识*。要了解锁是什么，就要了解 kernel context 提供了哪些对互斥访问的*共识*。

## Lock-free Scenarios

最简单的场景就是不需要锁，不需要对互斥访问的*共识*。在多核环境下，如果并发流是 CPU，则我们设置每 CPU 变量；如果并发流是多线程，我们设置每线程变量；如果并发流是多进程，那我们设置多进程变量。这样我们就需要考虑锁的概念了。

内核中就设置了每 CPU 变量，意思简单明了，该变量不会被多个 CPU 同时执行，比较有名的例子就是每 CPU 的 runqueue。

还有就是用户态下定义的线程本地变量：
```
_thread int a;
```
表达的也是一个意思，不同线程拥有不同的副本，虽然在定义上是相同的变量名，但是在不同线程内存中它们执行的不是同一份 int 类型内存。

而用户在调用 fork 之后，copy on write 机制会保证数据页在任意一方修改时会触发 deep copy, 此时父子进程操作的是不同的资源，自然不需要对互斥访问的*共识*。 

为了再深入去解释 kernel context 对互斥访问的**共识**概念，我们看个有点反直觉的场景。

### Kernel preempt

理论和定义上看每 CPU 变量是可以保证数据安全的，毕竟其他 CPU 上不管执行哪个逻辑流，都不会访问本 CPU 变量的值。

tricky 的地方在于本地 CPU 不是顺序执行的，在开内核抢占的场景下，内核执行代码被中断打断可能被其他进程抢占执行流。考虑下面的场景：
```
内核态 CPU0：

1. CPU0 执行进程 1 的系统调用内核路径，访问 CPU0 的每 CPU 变量 runqueue[0]。
2. 读取了进程描述符 A，准备将进程描述符 B 插入到 A 的所在链表后面，只执行到一半操作。
3. 时钟中断到来，CPU0 执行流被强行打断去执行时钟中断的中断服务例程。
4. 中断服务例程发现进程 2 的优先级更高，在退出中断服务例程之前，把 CPU0 切换到进程 2 的内核路径中。
5. 进程 2 的内核路径也读取了进程描述符 A，准备将进程描述符 C 插入到 A 的所在链表后面，并执行完成。
6. CPU0 调度进程 1 继续执行之前的内核路径，把剩余一半的操作继续完成。
```
可以看到，由于内核抢占机制，CPU0 runqueue[0] 数据被交替执行的逻辑破坏了。

在这个场景中，单纯的 pre-CPU 变量是一种互斥访问的*共识*吗？ 

**还不是**，虽然它看上去是，但是 kernel context 对中断的处理方式决定了，**pre-CPU 变量 + 禁止内核抢占** 才是一种对互斥访问的*共识*。 

我们通过这个例子更加深入体会了锁在不同 context 下是什么。

## Atomic Lock(Atomic Variable)

看完无锁场景之后，让我们再往前推进一小步。假设业务的确需要使用多个并发流访问同一份数据，那我们能提供的最简约的互斥访问*共识* 是什么？

一个的想法是：*是否有可能对一小块内存块规定，在某种机制下，kernel context 能对它的互斥访问达成共识。*

比如：多线程多进程能对一个 4 字节的 int 变量达成共识，规定所有对 int 操作的 +1/-1 指令都是原子的，哪个线程优先把 int 变量设置成 1 就说明占有资源，否则不占有资源，在不占有资源的时候给上层返回错误，或者进程被移动到等待队列等。

这是个可行的假设，但是还要再具体一点，我们再假设所有的变量都只在主存中读写操作，没有任何缓存，并且处理器提供 CAS(Compare And Set) 原子指令。这样每个并发流都执行 cas(variable，old value, new value) 的时候，就能在整个 kernel context 中保证，不管并发流执行顺序是什么样子的，都能保证在任意时候有且仅有一个执行流能占有资源，这就是一种互斥访问的*共识*。

这就是我们的原子变量，或者我们叫原子锁。它的`读-修改-写`操作是原子的，不可分割的。这样并发执行流就能天然在这个内存块上串行操作了，因为它是最小的不可分割的操作单元(或者叫原语 primitive)。

并且更妙的是，基于这样一个锁原语，我们可以进行拓展，比如占有资源的并发流再去做其他更多事情，做完之后把资源释放掉(将 int 设置成 0)。

*我们到达终点了吗？*

**醒醒啊，我们才刚确认了正确的方向而已。**

真正的 kernel context 可不是我们上面假设的那样所有的变量都只在主存中读写操作，真实情况是硬件系统存在不同级别的缓存，比如有名的存储山结构，按访问延迟依次是寄存器、CPU 专有的 L1/L2 硬件缓存，CPU 共享的 L3 硬件缓存，再到共享的物理内存。

![memory-hierarchy](/assets//images/post/linux-kernel-synchronization/memory-hierarchy.png)

硬件系统层面对原子变量的操作语义支持是复杂的，我们可以想象的几个不一致的场景：
1. 变量内存不对齐 native word size，或者变量内存不在一次指令内存操作范围内，需要使用多个指令才能操作变量。多个 CPU 可能在指令之间交替执行，比如变量的前 8 字节和后 8 字节可能不在一个原子操作中。
2. 最简单的状态变更 a++ 需要的不是一个操作，而是 3 个操作，读取内存中 a 变量的值到寄存器，改寄存器，将寄存器的值回写内存。多个 CPU 可能在操作之间交替执行，比如导致更新丢失。
3. 在存储山结构中，部分存储是 CPU 专有的，比如寄存器 L1/L2，部分存储是 CPU 共享的，比如 L3/memory。对于共享存储，我们可以通过锁内存总线的方式串行化特定区域的内存访问，但是对于特有的存储，就可能存在有旧的变量值在硬件缓存 L1/L2 中，CPU 误认为自己获取了最新的值。

因此，对于原子变量的实现，在系统层面上需要以下几种机制保证：  
1. 架构相关实现下，变量需要对齐 native word size 实现单个`读` 或`写`变量的原子操作，比如 8 bytes 变量需要地址低 3 bit 都为 0。  
架构相关实现下，保证操作可以翻译成单个指令执行。  
2. 对于`读-修改-写`操作，共享型内存提供锁内存总线的方式串行化不同 CPU 的访问，比如 lock 指令。
3. 对于`读-修改-写`操作，独享型内存提供处理器间机制，比如 MESI protocol，保证同一变量的不同缓存状态能互相影响，比如将其他 CPU 的高速缓存标记成失效状态。

通过以上机制，内核对外提供了非常简单直观的原子变量语义，比如：
```
struct atomic_int {
    int v;  // 一小块内存对齐的内存
};

define global struct atomic_int count;

int main() {
    atomic_inc(&count);
}
```

可能对应的汇编：
```
lock addl	$1, count(%rip)   // 锁总线+标记其他高速缓存失效
```

内核提供一些原子操作如下：
```
atomic_read(v)                // Return *v
atomic_add(i,v)               // Add i to *v
atomic_sub_and_test(i, v)     // Subtract i from *v and return 1 if the result is zero; 0 otherwise
```

原子变量提供了非常可靠的同步原语，所有的`读`/`写`/`读-修改-写`在多处理器上被认为是 instantly 和 simutaneously 的。这样对于同一个原子变量，多处理在该原子变量上的操作被同步了，最终表现出 some order，比如：

define atomic_int v;

| ts      |  P1     |   P2   |    P3  |
|---------|---------|--------|--------|
| t1      | read v  |        |        |
| t2      |         |        |write v |
| t3      |         |r-m-w v |        |
| t4      | write v |        |        |

好了，计算机大牛们终于帮我们迈过了锁历史上的一大步，我们得到了 kernel context 下对互斥访问的一个可靠*共识*。这种共识更多的是基于硬件系统的指令控制，比如内存总线锁和 MESI 缓存一致性协议，这些硬件指令细节都封装到了基于硬件系统的编译器中了。

## The implementation of Lock

基于原子变量这个可靠*共识*，我们可以开始看下在日常编程中使用的各种锁的真面目了，以及我们经常提及锁开销问题。

### Exclusive Lock and Critical Section 

如果你做过正式的多线程开发，如果你不太在乎多线程资源竞争的性能损耗，那你一定有了解互斥锁的概念，这几乎是一种标准的起手式。互斥锁的经典节点如下：
```
Mutex mux;
mux.lock(); // or mux.tryLock();
...
mux.unlock();
```

其实更有意思的是：**互斥锁是什么**。

当你问出这个问题的时候，你其实内心已经有某种不安全感，对操作系统这个黑盒子感到不安。

我们可以这样去理解这个问题，如果你看到我前面的 blog，就应该知道我们其实在一开始的时候并不存在这么复杂的操作系统和锁的概念，存在的只是一个能顺序执行指令的 CPU 和一块连续的物理内存。为了解决 CPU 的快慢问题我们引入了多进程多线程和分时复用的概念，为了解决内存的冷热和 locality 问题，我们引入虚拟内存和分页机制。

在引入这些复杂概念的前提下，现在我们提出的要求是，需要一个*互斥锁* 的概念，让不同的并发流能达成*共识*，这个共识是每次只能由一个执行流获得资源或者继续执行。

*那如何能达成这种共识* ？巧了，之前 kernel 废了老大劲实现的原子锁刚好可以提供这种 *共识*。这也是我们称原子锁为一种原语(primitive)的原语。

原子锁或者原子变量给我们这样一种语义，对于原子变量(本质上是一小块内存)的任何操作，不管操作系统存在什么样的缓存结构，对所有观察者都是 instant 且 simutaneous 的。

有了这种 primitive，并发执行流要访问一大块共享内存之前，都去读取和操作一个原子变量，比如使用 cas(varA，0, 1)。因为 cas 的操作在硬件层面是保证原子的，所以有且只有一个执行流可以将 varA 设置成 1。这个执行流成为了可以继续操作的 the one。

问题来了：**操作失败的那些并发执行流做什么去了？**。

好问题！！原子变量只是让并发执行流都达成了共识，但是认为自己没有获得操作权限的那些执行流是需要做某些动作的，这个是和实现相关的。在 linux kernel 中，这个动作就是**阻塞**。

我用**阻塞**这个词是因为它很形象，给人一种等待的感觉，这种感觉在 linux kernel 中就是硬件上下文切换。我们在进程切换 [blog](https://maxshuang.github.io/linux-kernel/2024/01/11/Linux-Kernel-Process-Scheduler.html) 中介绍过这个概念，本质上就是终端执行流，将相关的 context 保存在内存中，等待下次被调度，通过这种方式营造出一种执行流正在阻塞等待的感觉。

阻塞等待结束时，等待的执行流重新获得互斥锁执行，**这又是怎么回事？**。我们可以想象，所有等待的执行流被切换了硬件上下文之后，它们的执行流 ID 被统一维护在某种队列或者列表中。当占有锁的执行流释放锁时，会以某种方式**唤醒**这些执行流，而**唤醒**的具体意思是将执行流的结构体从等待或者休眠队列中加入到 runqueue，等待获取 CPU 时间片执行。我们有时候会听到一个叫**惊群**的现象，它本质上就是在说是不是一次唤醒所有等待的执行流，让他们再次运行操作原子变量的逻辑。当然，这样会损失效率，内核可以使用更好的算法，比如按照 FIFO 选择第一个等待的执行流。

讲到这里内核中的互斥锁的基本原理我们已经讲完了，总结下：
1. 原子变量保证串行化并发流对资源的占有行为，有且只有一个执行流可以占有资源，并且占有资源的行为会立即被其他并发流看到。
2. 不占有资源的执行流被 kernel 挂起，放入锁等待队列中，等锁释放后再唤醒等待的并发执行流。

从实现机制上，我们可以看到这类阻塞锁的开销在于*原子变量在多级缓存中的同步协议开销*，*进程挂起唤醒* 和 *进程被唤醒等待调度延迟*。

*进程挂起唤醒* 这个在前面[进程调度 blog](https://maxshuang.github.io/linux-kernel/2024/01/11/Linux-Kernel-Process-Scheduler.html)中提到，内核除了会被异步硬件事件打断进入内核之外，还会被周期的时钟中断(通常 1ms) 打断在内核中进行优先级更新和进程调度，所以进程被挂起后唤醒是可能有 ms 级别的延迟的。

### Spin Lock

自旋锁，语义上是指锁占有失败时通过执行流轮询的方式等待锁。它的经典接口和互斥锁很像，因为在语义层面没有太大区别。
```
SpinLock lk;
lk.lock(); // or lk.tryLock();
...
lk.unlock();
```

如果你理解了互斥锁在 linux kernel 层面的实现机制，理解这个就不难。首先需要原子变量达到共识，然后没有占有锁的执行流不会被切换上下文，而是执行一些 nop 空指令然后再次检查原子变量。

从实现机制上，可以看到 spin lock 具有*更快的调度速度*，可能是 ns 或者 us 级别的调度(和进程时间片轮转还有关)，不需要休眠等待唤醒，当然缺点也很明显，就是长时间的指令空转会浪费 CPU 指令周期。它的使用场景是，原子变量已经在各级缓存中，并且保护的代码区域非常短，并发冲突不强，内核中就经常使用。

这里有个有意思的事情，我之前用过用户态的 spin lock，其实就是封装了原子变量+nop+while操作。现在想起来其实可能在性能方面的意义不大，因为内核的 spin lock 是需要关闭内核抢占和屏蔽中断的，通过这样方式避免执行流被切换出去。而用户态的自旋锁因为没法使用内核级别的原语，从而没法避免被分时系统等切换出去，一旦执行流切换就会至少有 ms 级别的调度延迟。

### Condition Variable

条件变量是内核提供的另外一个非常重要的原语，基于这个原语可以进一步拓展出信号量(Semaphore)/读写锁(Read-Write Lock)/栅栏 Latch 等更高级语义的锁。条件变量的语义和前面说的互斥锁就不一样的，他的经典接口如下：
```
ConditionVariable cv;
Lock lk;
cv.wait(lk, condition);
...
cv.notifyOne(); // cv.notifyAll();
```

不同于互斥锁那种是隐式等待和唤醒的逻辑，条件变量将等待+条件检查+通知显式化了。意思就是说，有多个执行流同时调用 cv.wait(condition)，显示等待某个条件成立。有且只有一个执行流可以满足条件继续执行，其他的都要等待。直到获得权限的执行流显式调用 notifyOne 通知一个等待执行流，或者使用 notifyAll() 唤醒所有等待流之后再进行竞争。

对比互斥锁，互斥锁没有条件判断，unlock 操作相当于 notifyOne 或者 notifyAll。

所以**条件变量本质上是什么呢**？它本质上是一种**共识**+**条件检查**+**显式通知**。

*共识* 部分由互斥锁提供(搭配条件变量使用)，获得互斥锁之后检查*条件*是否满足，满足则继续执行，不满足则将该执行流加入到等待队列中，释放锁，等待被 notifyOne 或者 notifyAll 从等待队列中唤醒。

基于自定义的条件，我们可以构造出非常丰富的同步语义。举个例子，比如锁检查的条件是 count>0，我将 count 初始化成 6，这样在调用 wait 时只要满足 count>0 的执行流都可以执行，这就是 Semaphore 的语义。大致的代码实现如下：

```
int counter = 6;
std::mutex mtx;
std::condition_variable cond_var;

void request_resource() {
    std::unique_lock<std::mutex> lock(mtx);
    // wait for counter > 0
    cond_var.wait(lock, []{ return counter > 0; });
    // get resource，decrease counter
    --counter;
}

void release_resource() {
    std::unique_lock<std::mutex> lock(mtx);
    // release resource，increase counter
    ++counter;
    // notify the waiting thread
    cond_var.notify_one(); // or use notify_all()
}
```

### Semaphore

信号量，语义上允许在同一个时刻有多个执行流占用资源，而其他资源需要等待。他的经典接口如下：
```
counting_semaphore semaphore;
semaphore.acquire();  // or wait()
semaphore.release();  // or signal()
```

这类锁在实现上利用了 2 种机制：
1. 原子变量 count=n 保证有且只有 n 个执行流可以占有资源，并且占有资源的行为会立即被其他并发流看到。
2. 不占有资源的执行流会被挂起。

也许你会好奇，为什么我们可以直接通过条件变量实现信号量，还要提供信号量的原语？有几个原因：
1. *设计的目的不同*：条件变量针对的是复杂条件判断下的同步，需要互斥锁保护条件的检查；信号量则用于控制有限资源的并发访问，它本质上只依赖一个原子的计数器，不用检查复杂条件。
2. *对锁的需求不同*：条件变量依赖锁做复杂的条件判断安全，将锁分离出来可以使用更加灵活的锁机制，比如共享锁；信号量其实不需要锁，它只需要原子的计数器和内核等待唤醒机制支持。
3. *性能和效率不同*：条件变量每次检查条件都要加锁，这里涉及到用户态和内核态的切换；而信号量不用，posix 信号量使用用户态的原子计数器，当 counter>0 时不会切换内核态，当counter==0 时才会进入内核态实现等待和唤醒。

所以对于有限资源的并发访问，比如生产者-消费者模式，信号量更加高效。而条件变量牺牲性能实现更加灵活的同步策略。

### Advanced Lock

我们看下基于原子锁/条件变量/信号量这些同步原语，设计出来的一些高级锁机制。

#### Latch

latch 中文名栅栏，它的同步语义形象描述了某个并发流 A 等待其他多个并发流 B,C,D 到达某个同步点之后再执行的同步场景 B,C,D=>A，类似用来圈羊的栅栏。他的经典接口如下：

```
latch lt(5);
lt.count_down(); // internal --counter;
lt.wait();       // wait for counter==0
```

我们可以想象这样一种 latch 实现，有个计数器 counter=3，并发流 B,C,D 都在等待 counter>0 的场景，并且在结束时会 --counter 并且 notifyOne。并发流 A 则在等待 counter==0 的场景，它需要等待并发流 B,C,D 都结束时才可能被唤醒执行。

```
class latch {
public:
    explicit latch(int count) : counter(count) {}

    void count_down() {
        std::unique_lock<std::mutex> lock(mtx);
        if (--counter == 0) {
            cv.notify_all();
        }
    }

    void wait() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this]() { return counter == 0; });
    }

private:
    int counter;
    std::mutex mtx;
    std::condition_variable cv;
};
```

#### Read-Write Lock

读写锁，提高读写并发的常用的一类锁，语义上读读不冲突，读写/写读/写写冲突，它适用于读多写少的场景。

我们可以这样想象读写锁的实现，有个 runningReaderCounter 维护当前正在运行的 reader 个数，有个 waitingWriterCounter 维护正在等待的 writer 个数，有个 isWriterRunning 维护当前是否有 writer 在运行。
* reader 运行的条件为当前没有 writer 正在运行和没有等待的 writer: isWriterRunning==false && waitingWriterCounter==0。
* writer 运行的条件为当前既没有正在运行的 reader，也没有正在运行的 writer: isWriterRunning==false && runningReaderCounter==0。

1. 写优先的读写锁
下面是一个写优先的读写锁实现，它意味着对于请求序列-"读0-读1-写0-读2-读3-写1" 这种并发序列，如果写1在写0未结束时就 wait，则在调度时会优先调度写1，执行序列为-"读0-读1-写0-写1-读2-读3"。

```
// This implementation is essentially write-priority (with more reads and fewer writes), meaning as long as there is a writer running or waiting, it will block readers. Therefore, we don't maintain `waitingReaders`
class ReadWriteLock {
private:
    std::mutex mtx;
    std::condition_variable cond;   // Readers and writers share one condition variable
    int readerCount = 0;            // Number of active readers, essentially `runningReader`
    bool writerActive = false;      // Is there an active writer? Essentially `runningWriter`
    int waitingWriters = 0;         // Number of waiting writers (for priority)

public:
    // Reader lock (enter read mode)
    void lockRead() {
        std::unique_lock<std::mutex> lock(mtx);
        // Wait until all writers are done; this is write-priority
        cond.wait(lock, [this]() { return !writerActive && waitingWriters == 0; });
        ++readerCount;  // Increment reader count
    }

    // Reader unlock (exit read mode)
    void unlockRead() {
        std::unique_lock<std::mutex> lock(mtx);
        --readerCount;
        if (readerCount == 0) {
            cond.notify_all();  // Notify writers and readers waiting, but readers will go back to sleep
        }
    }

    // Writer lock (enter write mode)
    void lockWrite() {
        std::unique_lock<std::mutex> lock(mtx);
        ++waitingWriters;  // Increment the number of waiting writers
        // Wait until there are no writers or active readers
        cond.wait(lock, [this]() { return !writerActive && readerCount == 0; });
        --waitingWriters;  // Decrement waiting writers as writer is now active
        writerActive = true;
    }

    // Writer unlock (exit write mode)
    void unlockWrite() {
        std::unique_lock<std::mutex> lock(mtx);
        writerActive = false;
        cond.notify_all();  // Notify all readers and writers waiting
    }
};
```

2. 读优先的读写锁
我们再实现一个读优先的读写锁，此时 reader 等待的条件变成只要不存在正在运行的 writer 就可执行：cond.wait(lock, [this]() { return !writerActive; })。对于这种条件，最直观的结果就是，writer 运行完后，waiting writer 和 waiting reader 都有机会获得运行机会。一旦 reader 获得运行机会，则 waiting writer 要等待所有 reader 运行完才能运行，所以我们称它为读优先。

举个例子，对于请求序列-"读0-写0-读1-写1-读2-读3-写2" 这种并发序列，它的执行序列可能是-"读0-读1-读2-读3-写1-写0-写2"。

```
class ReadWriteLock {
private:
    std::mutex mtx;
    std::condition_variable cond;   // Readers and writers share one condition variable
    int readerCount = 0;            // Number of active readers, essentially `runningReader`
    bool writerActive = false;      // Is there an active writer? Essentially `runningWriter`
    int waitingWriters = 0;         // Number of waiting writers (for priority)

public:
    // Reader lock (enter read mode)
    void lockRead() {
        std::unique_lock<std::mutex> lock(mtx);
        // Read-priority: as long as there is no running writer, readers have a chance to be awakened; 
        // heavy reading can cause writer starvation
        cond.wait(lock, [this]() { return !writerActive; });
        ++readerCount;  // Increment reader count
    }

    // Reader unlock (exit read mode)
    void unlockRead() {
        std::unique_lock<std::mutex> lock(mtx);
        --readerCount;
        if (readerCount == 0) {
            cond.notify_all();  // Notify writers and readers waiting, but readers will go back to sleep
        }
    }

    // Writer lock (enter write mode)
    void lockWrite() {
        std::unique_lock<std::mutex> lock(mtx);
        ++waitingWriters;  // Increment the number of waiting writers
        // Wait until there are no writers or active readers
        cond.wait(lock, [this]() { return !writerActive && readerCount == 0; });
        --waitingWriters;  // Decrement waiting writers as writer is now active
        writerActive = true;
    }

    // Writer unlock (exit write mode)
    void unlockWrite() {
        std::unique_lock<std::mutex> lock(mtx);
        writerActive = false;
        cond.notify_all();  // Notify all readers and writers waiting
    }
};
```

3. 公平的读写锁

公平的读写锁会均衡读写请求的比例，比如利用一个 flipper 交替选择读线程和写线程，或者将读者和写者按照请求顺序放入 FIFO 队列中，通过队列头元素选择执行读线程还是写线程。

```
class ReadWriteLock {
private:
    std::mutex mtx;
    std::condition_variable cond;   // Readers and writers share one condition variable
    int readerCount = 0;            // Number of active readers, essentially `runningReader`
    bool writerActive = false;      // Is there an active writer? Essentially `runningWriter`
    int waitingWriters = 0;         // Number of waiting writers (for priority)
    bool preferWriter = false;      // flipper

public:
    // Reader lock (enter read mode)
    void lockRead() {
        std::unique_lock<std::mutex> lock(mtx);
        // Fair scheduling:
        cond.wait(lock, [this]() { return !writerActive && (!preferWriter || waitingWriters == 0); });
        ++readerCount;  // Increment reader count
        preferWriter=!preferWriter;
    }

    // Reader unlock (exit read mode)
    void unlockRead() {
        std::unique_lock<std::mutex> lock(mtx);
        --readerCount;
        if (readerCount == 0) {
            cond.notify_all();  // Notify writers and readers waiting, but readers will go back to sleep
        }
    }

    // Writer lock (enter write mode)
    void lockWrite() {
        std::unique_lock<std::mutex> lock(mtx);
        ++waitingWriters;  // Increment the number of waiting writers
        // Wait until there are no writers or active readers
        cond.wait(lock, [this]() { return !writerActive && preferWriter && readerCount == 0; });
        --waitingWriters;  // Decrement waiting writers as writer is now active
        writerActive = true;
        preferWriter=!preferWriter;
    }

    // Writer unlock (exit write mode)
    void unlockWrite() {
        std::unique_lock<std::mutex> lock(mtx);
        writerActive = false;
        cond.notify_all();  // Notify all readers and writers waiting
    }
};
```

按请求顺序调度的公平读写锁：
```
enum class ThreadType { Reader, Writer };

struct ThreadInfo {
    std::thread::id threadID;  // Unique thread identifier
    ThreadType type;           // Thread type (Reader or Writer)
};

class ReadWriteLock {
private:
    std::mutex mtx;
    std::condition_variable cond;
    int readerCount = 0;
    bool writerActive = false;
    std::queue<ThreadInfo> waitQueue;  // FIFO queue for managing threads
    std::unordered_map<std::thread::id, bool> ready;  // Tracks whether a thread can proceed

public:
    // Reader thread acquires the lock
    void lockRead() {
        std::unique_lock<std::mutex> lock(mtx);
        std::thread::id thisID = std::this_thread::get_id();
        waitQueue.push({thisID, ThreadType::Reader});
        ready[thisID] = false;  // Initialize the thread's ready state to false

        // Wait until this thread is the first in the queue and no writer thread is active
        cond.wait(lock, [this, thisID]() {
            return !writerActive && (waitQueue.front().threadID == thisID);
        });
        ++readerCount;
        waitQueue.pop();  // The reader thread acquires the lock, remove it from the queue
    }

    // Reader thread releases the lock
    void unlockRead() {
        std::unique_lock<std::mutex> lock(mtx);
        --readerCount;
        if (readerCount == 0) {
            cond.notify_all();  // Notify all threads, allowing writers to acquire the lock
        }
    }

    // Writer thread acquires the lock
    void lockWrite() {
        std::unique_lock<std::mutex> lock(mtx);
        std::thread::id thisID = std::this_thread::get_id();
        waitQueue.push({thisID, ThreadType::Writer});
        ready[thisID] = false;  // Initialize the thread's ready state to false

        // Wait until this thread is the first in the queue and no other readers or writers are running
        cond.wait(lock, [this, thisID]() {
            return readerCount == 0 && !writerActive && (waitQueue.front().threadID == thisID);
        });
        writerActive = true;
        waitQueue.pop();  // The writer thread acquires the lock, remove it from the queue
    }

    // Writer thread releases the lock
    void unlockWrite() {
        std::unique_lock<std::mutex> lock(mtx);
        writerActive = false;
        cond.notify_all();  // Notify all waiting threads
    }
};
```

#### Distributed Lock

我们继续看一个更高级的锁-分布式锁，这种锁不再局限于 linux kernel 和单机，而是为分布式环境中的实体提供一个锁服务。

有意思的是，有时候分布式锁可以帮我们看清楚**锁原语**的语义。我们之前说**锁其实是一种共识**，比如原子变量就是通过各种复杂的硬件和软件机制在内核层面实现了一个**锁原语**。

在分布式环境下，**锁原语**反而变得更加清晰起来，因为对于分布式环境中的实体，他们和锁服务的交互方式只有网络交互，也就是说在网络交互下，只要锁服务自己能提供一个*共识* 就行。这种 *共识* 可以是 "a==2" 或者 "set a=1 if a>0"，更广泛的说法是对于互斥锁语义，它只要能保证在任何时间段都只让一个实体持有网络请求成功的结果就行。

这也是为什么 redis 这种高性能缓存经常被用来实现分布式锁的原因，因为它内部执行是单线程的，只要维护 version 作为乐观锁，就天然能基于 version 实现 CAS 接口，自然能实现上面我们说的**锁原语**。

比如利用数据库的 row lock + where 机制也可以实现**锁原语**，比如 update a=1 where a=0，因为有且只有一个 update 的 affected row 为 1。或者可以在自己的记录中增加 version column 实现类似的能力也行。

分布式锁的实现方式中，还有公平的分布式锁和不公平的分布锁，我不确定这个是不是我自己命名的。简单来说：
* *公平的分布式锁*: 按照请求顺序依次获得锁，比如 zookeeper/etcd 实现的分布式锁，通过在目录下排队，队首元素认为获得了分布式锁。通过维护 session 保证队首掉线后，下一个元素称为队首，获得锁。
* *不公平的分布式锁*: 所有实体通过 CAS 机制竞争某个值，比如执行自增操作，成功将 0 变成 1 的实体获得锁。这种竞争方式取决于分布式环境中实体的网络延迟/CPU 等条件，属于优者获胜，所以我称为不公平的分布式锁。这种不公平的分布式锁在某些场景下还是很有优势的，因为它本身就是真实分布式环境的一种反馈，比如做缓存更新时，让当时延迟更低的实体更新缓存是个更快的选择。

我们再发散下，如何实现一个分布式的写优先锁？如果你理解了前面的内容，那到了这里只是一个组合不同实现的问题而已。

#### Optimistic Lock

乐观锁我们之前提到一下，悲观锁和乐观锁在数据库领域比较经常被提到。悲观锁机制就是没有获得锁的执行流需要等待，而乐观锁则不等待，它依赖版本机制在执行结束前判断在执行过程中是否发生了其他执行流变更了数据版本，如果发现数据版本变更了，需要回滚和重做整个操作，否则可以直接结果。

乐观锁更轻量级，实现上一般只是某种版本机制，适用于并发冲突不多的场景。

这种版本机制有着更加广泛且深刻的应用，比如 google spanner 用于实现分布式事务的 [percolator](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36726.pdf)，利用了 primary key 上的版本机制锁定了一个事务范围的多个记录；又比如 Paxos prepare-accept 两阶段中，利用 highest proposal number 拒绝其他 proposal，这个本身就是一种锁机制。

## Conclusion

这篇博客从一个锁的疑问出发，分析各种同步原语和锁在 linux kernel 中的实现，并且拓展到了更广泛意义上的锁原语。读者可以更加深入理解锁的原理机制，并可以根据实际的业务场景自由拓展锁的语义。