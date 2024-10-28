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
  height: "70vh"
  min_height: "50vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Linux-Kernel Concurrency Lock
---

## What is Lock

I have been thinking about this topic for a long time and I have been writing for a long time, but I am not very satisfied with it. Recently, I have been interviewing and I just got inspiration.

The first time I came into contact with the concept of "lock" was when I was exposed to multi-threaded programming. Lock is an essential tool to ensure data integrity when the same data is accessed by multiple concurrent logics at the same time.

If we ignore the implementation of the lock and only consider the lock primitives, we can imagine that in order to get the supplies in the room in an orderly manner. *Multiple people need to lock the door in sequence, take the things, and then unlock the door to let the next person in. In this way, no matter whether multiple people are lined up in a row or scattered at the door, only one person can enter the door at any time.*

As I got deeper into programming, I gradually came into contact with various locks, such as critical sections/read-write locks/spin locks in the Linux kernel, row lock, gap lock, next-key lock and intention lock in the database, ReentrantLock, StampedLock and Latch in Java, and even lock-free data structures in high-concurrency data structures.

I gradually got lost in the various locks and began to wonder.

**What exactly is a lock?**

**What are the underlying locks of the existing ones? Why do locks incur overhead? How to create new locks?**

## The Consensus in Synchronization

A lock in concurrency is first of all a consensus, which is a consensus on mutually exclusive access in a specific context.

To be more specific, in a scenario, 10 people want to drink a glass of water, but only one person can drink it. At this time, I stipulate that the person who first calculates ```23451*1987``` can drink it. In such a context, I stipulate that the time for each person to calculate ```23451*1987``` is different. Therefore, the calculation behavior "calculating ```23451*1987```" is a kind of lock in this scenario, because it can reach a consensus on mutually exclusive access in this context .

Back to the lock in our programming language, because the programming language itself needs to be converted into machine language, the locks in the programming language are based on the consensus on mutually exclusive access provided by the kernel context. To understand what a lock is, you need to understand what consensus on mutually exclusive access the kernel context provides.

## Lock-free Scenarios
The simplest scenario is it does not require locks or consensus on mutually exclusive access. In a multi-core environment, if the concurrent stream is a CPU, we set per-CPU variables; if the concurrent stream is multi-process, we set multi-process variables; if the concurrent stream is multi-threaded, we set per-thread variables. In this way, we do not need to consider the concept of locks.

The kernel sets per-CPU variables, which are simple and clear in meaning. The variables will not be executed by multiple CPUs at the same time. A well-known example is the runqueue per CPU in Linux kernel.

There are also thread local variables defined in user mode: *_thread int a*;  
It also means that different threads have different copies. Although the variable name is the same in definition, they do not execute the same memory in different thread memories.

After the user calls fork, the COW(copy-on-write) mechanism ensures that a deep copy is triggered when the data page is modified by either party. At this time, the parent and child processes operate on different resources, so there is no need for consensus on mutually exclusive access .

To further explain the kernel context's consensus concept for mutually exclusive access, let's look at a somewhat counterintuitive scenario.

### Kernel Preempion
Theoretically and by definition, data security can be guaranteed for each CPU variable. After all, no matter which logic flow is executed on other CPUs, the value of the CPU variable will not be accessed.

The tricky part is that the local CPU does not execute sequentially. In the scenario where kernel preemption is enabled, the kernel execution code is interrupted and the execution flow may be preempted by other processes. 

Consider the following scenario:
```
Kernel Mode CPU0:

1. CPU0 executes the system call kernel path of Process 1, accessing CPU0's per-CPU variable `runqueue[0]`.
2. It reads Process Descriptor A and is preparing to insert Process Descriptor B after A in the linked list, but the operation is only halfway done.
3. A clock interrupt occurs, and CPU0's execution flow is forcibly interrupted to execute the clock interrupt service routine.
4. The interrupt service routine detects that Process 2 has a higher priority. Before exiting the interrupt service routine, it switches CPU0 to the kernel path of Process 2.
5. The kernel path of Process 2 also reads Process Descriptor A, prepares to insert Process Descriptor C after A in the linked list, and completes the operation.
6. CPU0 schedules Process 1 to continue executing its previous kernel path and completes the remaining half of its operation.
```

It can be seen that due to the kernel preemption mechanism, the CPU0 runqueue[0] data is corrupted by the alternate execution logic.

In this scenario, *is the pure pre-CPU variable a consensus for mutually exclusive access?*

**Not yet**, although it looks like it, but the way the kernel context handles interrupts determines that *pre-CPU variables + disabling kernel preemption + disabling interrupts* is a consensus on mutually exclusive access .

Through this example, we have a deeper understanding of how consensus is different in different contexts.

## Atomic Lock(Atomic Variable)

After looking at the lock-free scenario, let's take a small step forward. Assuming that the business **does** need to use multiple concurrent streams to access the same data, what is the simplest mutually exclusive access consensus we can provide?

**One idea is**:   
*Is it possible to stipulate a small block of memory, under some mechanism, that the kernel context can reach a consensus on its mutually exclusive access?*

For example: multiple threads and multiple processes can reach a consensus on a 4-byte int variable, stipulating that all +1/-1 instructions for int operations are atomic. The thread that sets the int variable to 1 first occupies the resources, otherwise it does not occupy the resources. When it does not occupy the resources, it returns an error to the upper layer, or the process is moved to the waiting queue.

**This is a feasible assumption, but it needs to be more specific**. Let's assume that all variables are only read and written in the main memory, there is no cache, and the processor provides CAS(Compare And Set) atomic instructions. In this way, when each concurrent flow executes $cas(variable, old value, new value)$, it can be guaranteed in the entire kernel context that no matter what the execution order of the concurrent flows is, it can be guaranteed that at any time, only one execution flow can occupy the resource. This is a consensus of mutually exclusive access .

This is our atomic variable, or what we call an atomic lock. Its *read-modify-write* operation is atomic and indivisible. In this way, concurrent execution flows can naturally operate serially on this memory block because it is the smallest indivisible operation unit (or primitive).

What's even better is that based on such a lock primitive, we can expand it, such as having concurrent streams that occupy resources do more things and then release the resources (set the int to 0).

**Are we reaching the end?**

**Wake up, we have just confirmed the right direction.**

The real kernel context is not what we assumed above, where all variables are only read and written in the main memory. The actual situation is that the hardware system has different levels of cache, such as the famous memory mountain structure. The access latency is in the order of registers, CPU-specific L1/L2 hardware cache, CPU-shared L3 hardware cache, and then shared physical memory.

![memory-hierarchy](/assets//images/post/linux-kernel-synchronization/memory-hierarchy.png)

The hardware system-level support for the operational semantics of atomic variables is complex. We can imagine several inconsistent scenarios:

1. The variable memory is *not aligned to* the native word size, or the variable memory is not within the memory operation range of one instruction, and multiple instructions are required to operate the variable. Multiple CPUs may alternate between instructions, for example, the first 8 bytes and the last 8 bytes of a variable may not be in one atomic operation.
2. The simplest state change a++ *requires not one operation*, but three operations: reading the value of variable a in memory into a register, changing the register, and writing the register value back to memory. Multiple CPUs may alternate between operations, for example, causing updates to be lost.
3. In the memory mountain structure, *some storage is CPU-specific*, such as registers L1/L2, and some storage is CPU-shared, such as L3/memory. For shared storage, we can serialize memory access to specific areas by locking the memory bus, but for unique storage, there may be old variable values ​​in the hardware cache L1/L2, and the CPU may mistakenly believe that it has obtained the latest value.

Therefore, for the implementation of atomic variables, the following mechanisms are required at the system level:
1. **In architecture-dependent implementations**, variables need to be aligned to the native word size to implement single write or single read variable atomic operations. For example, an 8-byte variable needs the lower 3 bits of the address to be 0.
In architecture-dependent implementations, ensure that the operation can be translated into a single instruction for execution.
2. **For read-modify-write operations**, shared memory provides a way to lock the memory bus to serialize access from different CPUs, such as the lock instruction.
3. **For read-modify-write operations**, dedicated memory provides inter-processor mechanisms, such as the MESI protocol, to ensure that different cache states of the same variable can affect each other, such as marking the cache of other CPUs as invalid.
Through the above mechanism, the kernel provides very simple and intuitive atomic variable semantics to the outside world, such as:

```
struct atomic_int {
    int v;  // a small and aligned memory
};

define global struct atomic_int count;

int main() {
    atomic_inc(&count);
}
```

Possible corresponding compilation:

```
lock addl	$1, count(%rip)   // lock memory bus+ invalidate the variables in other caches
```

The kernel provides some atomic operations as follows:

```
atomic_read(v)                // Return *v
atomic_add(i,v)               // Add i to *v
atomic_sub_and_test(i, v)     // Subtract i from *v and return 1 if the result is zero; 0 otherwise
```

Atomic variables provide very reliable synchronization primitives, all reads/writes/read-modify-writes on atomic variables are considered instant and simutaneous on multiprocessors. Thus, the operations of multiple processors on the atomic variable are synchronized, and finally some order is shown, such as:

```
define atomic_int v;
```

| ts      |  P1     |   P2   |    P3  |
|---------|---------|--------|--------|
| t1      | read v  |        |        |
| t2      |         |        |write v |
| t3      |         |r-m-w v |        |
| t4      | write v |        |        | 	

Well, computer experts have finally helped us take a big step forward in the history of locks, and we have reached a reliable consensus on mutually exclusive access in the kernel context. 

This consensus is more based on the instruction control of the hardware system, such as the memory bus lock and the MESI cache consistency protocol. These hardware instruction details are encapsulated in the compiler based on the hardware system.

## The implementation of Lock
Based on this reliable consensus on atomic variables, we can begin to look at the true nature of various locks used in daily programming, as well as the lock overhead issue we often mention.

### Exclusive Lock and Critical Section
If you have done multi-threaded development before, then you must have understood the concept of mutex locks, which is almost a standard starting point. The classic interface of a mutex lock is as follows:

```
Mutex mux;
mux.lock(); // or mux.tryLock();
...
mux.unlock();
```

In fact, what is more interesting is: **what is a mutex lock**.

When you ask this question, you actually already have a certain insecurity in your mind and feel uneasy about the black box of the operating system.

We can understand this problem in this way. If you have read my previous [blog](https://maxshuang.github.io/linux-kernel/2022/07/25/What-The-Linux-Kernel-Does.html), you should know that we did not have such a complex operating system and the concept of locks at the beginning. There was only a CPU that could execute instructions sequentially and a continuous physical memory. In order to solve the problem of CPU speed, we introduced the concepts of multi-process multi-threading and time-sharing multiplexing. In order to solve the problem of hot and cold memory and locality, we introduced virtual memory and paging mechanism.

Under the premise of introducing these complex concepts, we now require a concept of mutual exclusion lock so that different concurrent flows can reach a **consensus**. This consensus is that only one execution flow can obtain resources or continue execution at a time.

**So how can we reach this consensus**? Coincidentally, the atomic lock that the kernel has worked so hard to implement can provide this *consensus*, which is why we call the atomic lock => a primitive.

Atomic locks or atomic variables give us such semantics that any operation on an atomic variable (essentially a small block of memory) is instant and simutaneous to all observers, regardless of the cache structure of the operating system.

With this primitive, before concurrent execution flows access a large block of shared memory, they read and operate on an atomic variable, such as using $cas(varA, 0, 1)$. Because the cas operation is guaranteed to be atomic at the hardware level, there is only one execution flow that can set varA to 1. This execution flow becomes the one that can continue to operate.

The question is: **what happened to the concurrent execution flows that failed?**

Good question! Atomic variables only allow concurrent execution flows to reach a consensus, but those execution flows that think they do not have permission to operate need to take certain actions, which is implementation-dependent. In the Linux kernel, this action is **blocking**.

I use the word **blocking** because it is very vivid and gives people a feeling of waiting, which is the hardware context switching in the Linux kernel. We introduced this concept in the [process switching blog](https://maxshuang.github.io/linux-kernel/2024/01/11/Linux-Kernel-Process-Scheduler.html). In essence, it is to save the context related to the execution flow in the memory and wait for the next scheduling, which creates a feeling that the execution flow is blocked and waiting.

When the blocking wait ends, the waiting execution flows compete for the mutex lock again. **What's going on?** We can imagine that after the hardware context of all waiting execution flows is switched, their execution flow IDs are uniformly maintained in a queue or list. When the execution flow that holds the lock releases the lock, these execution flows will be **awakened** in some way. The specific meaning of *awakening* is to add the structure of the execution flow from the waiting or sleeping queue to the runqueue, waiting to obtain the CPU time slice for execution. Sometimes we hear a phenomenon called **herd shock**, which is essentially about whether to wake up all the waiting execution flows at once and let them run the logic of operating atomic variables again. Of course, this will lose efficiency, and the kernel can use better algorithms, such as selecting the first waiting execution flow according to FIFO.

At this point we have finished talking about the basic principles of mutex locks in the kernel, let's summarize:
1. Atomic variables ensure that concurrent flows **serialize** their resource possession behavior. Only one execution flow can possess resources, and the behavior of possessing resources will be immediately seen by other concurrent flows.
2. The execution flow that does not occupy resources is **suspended** by the kernel and placed in the lock waiting queue. The waiting concurrent execution flow is awakened after the lock is released.

From the implementation mechanism, we can see that the overhead of this type of blocking lock lies in the synchronization protocol overhead of atomic variables in multi-level caches, the process suspension wake-up and the process wake-up waiting for scheduling delay .

Process suspension and wakeup This was mentioned in the previous [process scheduling blog](https://maxshuang.github.io/linux-kernel/2024/01/11/Linux-Kernel-Process-Scheduler.html). In addition to being interrupted by asynchronous hardware events to enter the kernel, the kernel is also interrupted by *periodic clock interrupts* (usually 1ms) to perform priority updates and process scheduling in the kernel. Therefore, there may be a delay of ms when a process is woken up after being suspended.

### Spin Lock
Spin lock, semantically means waiting for the lock by executing stream polling when lock possession fails. Its classic interface is very similar to mutex lock, because there is not much difference at the semantic level.

```
SpinLock lk;
lk.lock(); // or lk.tryLock();
...
lk.unlock();
```

If you understand the implementation mechanism of mutex locks at the Linux kernel level, it is not difficult to understand this. First, the atomic variables need to **reach a consensus**, and then the execution flow that does not hold the lock will not be context-switched, but will **execute some nop instructions** and then check the atomic variables again.

From the implementation mechanism, we can see that spin lock has a *faster scheduling speed*, which may be ns or us level scheduling (related to the process time slice rotation), and does not require sleep waiting for wake-up. Of course, the disadvantage is also obvious, that is, long-term instruction idling will waste CPU instruction cycles. Its usage scenario is that atomic variables are already in caches at all levels, and the protected code area is very short, and concurrency conflicts are not strong, so it is often used in the kernel.

Here is an interesting thing. I have used user-mode spin lock before, which is actually **encapsulating atomic variables + nop + while operations**. Now I think it may not be *very meaningful* in terms of performance, because the kernel spin lock needs to turn off kernel preemption and shield interrupts to prevent the execution flow from being switched out. User-mode spin locks cannot use kernel-level preemption and other mechanisms, so they cannot avoid being switched out by time-sharing systems. Once the execution flow is switched, there will be at least ms-level scheduling delay.

### Condition Variable
Condition variables are another very important primitive provided by the kernel. Based on this primitive, we can further develop locks with **higher-level semantics**, such as semaphores, read-write locks, and fence latches. The semantics of condition variables are different from the mutex locks mentioned above. Its classic interface is as follows:

```
ConditionVariable cv;
Lock lk;
cv.wait(lk, condition);
...
cv.notifyOne(); // cv.notifyAll();
```

Different from the implicit waiting and waking logic of mutexes, condition variables make **waiting + condition checking + notification explicit**. This means that multiple execution flows call $cv.wait(condition)$ at the same time, explicitly waiting for a certain condition to be met. Multiple execution flows that meet the condition continue to execute, and those that do not meet the condition have to wait. The condition is not judged until the execution flow that has obtained permission **explicitly calls notifyOne()** to notify a waiting execution flow, or **uses notifyAll()** to wake up all waiting flows.

Compared with the mutex lock, the mutex lock has no conditional judgment, and the unlock operation is equivalent to notifyOne or notifyAll.

So **what is a condition variable essentially** ?   
It is essentially a **consensus + condition check + explicit notification**.

The consensus part is provided by the mutex lock (used with the condition variable). After obtaining the mutex lock, **check whether the condition is met**. If it is met, continue to execute. If not, add the execution flow to the waiting queue, release the lock, and wait to be awakened from the waiting queue by $notifyOne()$ or $notifyAll()$.

Based on custom conditions, we can construct *very rich synchronization semantics*. For example, if the lock check condition is $count>0$, I initialize count to 6, so that when calling $wait()$, as long as the execution flow satisfies $count>0$, it can be executed. This is **the semantics of Semaphore**. The general code implementation is as follows:

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

Semaphore semantically allows multiple execution flows to occupy resources at the same time, and the execution flow that has not obtained the resource needs to wait. Its classic interface is as follows:

```
counting_semaphore semaphore;
semaphore.acquire();  // or wait()
semaphore.release();  // or signal()
```

This type of lock uses two mechanisms in its implementation:
1. The atomic variable count = n ensures that there are only n execution flows that can occupy resources, and the behavior of occupying resources will be immediately seen by other concurrent flows.
2. Execution flows that do not hold resources are suspended.

You may be curious, **why do we need to provide semaphore primitives when we can implement semaphores directly through condition variables?**   

There are several reasons:  
1. *The design purposes are different*: condition variables are aimed at synchronization **under complex conditional judgments**, and require mutex locks to protect the condition checks; semaphores are used to control concurrent access to limited resources. Semaphores **essentially only rely on an atomic counter** and do not need to check complex conditions.
2. *The requirements for locks are different*: condition variables **rely on locks** to make complex conditional judgments safe. Separating the locks allows the use of more flexible lock mechanisms, such as shared locks. Semaphores do not actually require locks, they **only require atomic counters** and kernel wait wake-up mechanism support.
3. *Different performance and efficiency*: condition variables must be locked each time the condition is checked, which involves switching between user state and kernel state; semaphores do not need this. POSIX semaphores use **atomic counters** in user state. When $counter>0$, they will not switch to kernel state. They will enter kernel state only when $counter==0$ to implement waiting and waking.

Therefore, for concurrent access to limited resources, such as the producer-consumer model, **semaphores are more efficient**, while condition variables sacrifice performance to achieve more flexible synchronization strategies.

### Advanced Lock
Let's take a look at some advanced locking mechanisms designed based on synchronization primitives such as atomic locks/condition variables/semaphores.

**Latch**  

Latch is another name for fence. Its synchronization semantics vividly describes a synchronization scenario where a concurrent flow A **waits for multiple concurrent flows** B, C, D to reach a synchronization point before executing B, C, D=>A, similar to a fence used to enclose sheep. Its classic interface is as follows:

```
latch lt(5);
lt.count_down(); // internal --counter;
lt.wait();       // wait for counter==0
```

We can imagine a latch implementation like this, with a counter counter=3, concurrent streams B, C, and D are all waiting for the scenario where $counter>0$, and when they finish, they will $–counter$ and $notifyAll()$. Concurrent stream A is waiting for the scenario where $counter==0$, and it needs to wait for concurrent streams B, C, and D to finish before it can be awakened and executed.

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

### Read-Write Lock
Read-write lock is a commonly used type of lock to improve read-write concurrency. Semantically, there is no read-read conflict, but there is **read-write/write-read/write-write** conflict. It is suitable for scenarios with more reads than writes.

We can imagine the implementation of the read-write lock like this: there is a $runningReaderCounter$ that maintains the number of readers currently running, a $waitingWriterCounter$ that maintains the number of writers waiting, and an $isWriterRunning$ that maintains whether there is a writer currently running.
1. The condition for the reader to run is that there is currently no writer running and no waiting writer: ```isWriterRunning==false && waitingWriterCounter==0```.
2. The condition for the writer to run is that there is neither a running reader nor a running writer: ```isWriterRunning==false && runningReaderCounter==0```.

**Write-priority read-write lock**  

The following is a write-priority read-write lock implementation, which means that for the concurrent sequence of request sequence - ```read 0-read 1-write 0-read 2-read 3-write 1```, if write 1 waits before write 0 is finished, write 1 will be scheduled first, and the execution sequence is - ```read 0-read 1-write 0-write 1-read 2-read 3```.

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

**Read-priority read-write lock**  

Let's implement a read-priority read-write lock. In this case, the reader's waiting condition becomes that it can be executed as long as there is no running writer: ```cond.wait(lock, this { return !writerActive; })```. 

For this condition, the most intuitive result is that after the writer finishes running, both the waiting writer and the waiting reader have the opportunity to run. **Once the reader gets the opportunity to run, the waiting writer must wait for all readers to finish running before running**, so we call it *read priority*.

For example, for a concurrent sequence such as the request sequence ```read 0-write 0-read 1-write 1-read 2-read 3-write 2```, its execution sequence may be ```read 0-read 1-read 2-read 3-write 1-write 0-write 2```.

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

**Fair read-write locks**  

Fair read-write locks balance the ratio of read and write requests, such as using a flipper to **alternately select** read threads and write threads, or putting readers and writers into a FIFO queue in the order of requests, and selecting whether to execute a read thread or a write thread based on the queue head element.

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

**Fair read-write locks scheduled in request order**:

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

### Distributed Lock
Let's continue to look at a more advanced lock - distributed lock. This lock is no longer limited to the Linux kernel and a single machine, but provides a lock service for entities in a distributed environment.

Interestingly, sometimes distributed locks can help us see the **semantics of lock primitives**. We said before that locks are actually **a kind of consensus**. For example, atomic variables are implemented as a lock primitive at the kernel level through various complex hardware and software mechanisms .

In a distributed environment, the lock primitive becomes clearer, because for entities in a distributed environment, the only way they interact with the lock service is *through network interaction*, that is, under network interaction, as long as the lock service can provide a **consensus**, it will be fine. This consensus can be ```a==2``` or ```set a=1 if a==0```. In a broader sense, for the semantics of mutual exclusion locks, it only needs to ensure that only one entity holds the result of a successful network request at any time.

This is why high-performance caches such as redis are often used to implement distributed locks. Because its internal execution is single-threaded, as long as the version is maintained as an optimistic lock, the CAS interface can be implemented based on the version, and the lock primitives mentioned above can be implemented naturally .

Aother example is, you can use the database's row lock + where mechanism to implement lock primitives too, such as "update a=1 where a=0", because there is only one updated affected row that is 1. Or you can add a version column to your own records to achieve similar capabilities.

There are also fair distributed locks and unfair distributed locks in the implementation of distributed locks. I am not sure if this is a standard classification. In short:
1. **Fair distributed lock**: Obtain locks in the order of request, such as the distributed lock implemented by zookeeper/etcd, by queuing in the directory, the first element in the queue is considered to have obtained the distributed lock. By maintaining the session, it is ensured that after the first element in the queue is disconnected, the next element becomes the first element in the queue and obtains the lock.
2. **Unfair distributed locks**: All entities compete for a certain value through the CAS mechanism. For example, the entity that successfully changes 0 to 1 after performing an auto-increment operation obtains the lock. This competition method depends on the network latency/CPU and other conditions of the entities in the distributed environment. The best one wins, so I call it an unfair distributed lock. This unfair distributed lock is still very advantageous in some scenarios because it is itself a kind of feedback from the real distributed environment. For example, when updating the cache, it is a better choice to let the entity with lower latency at that time obtain the distributed lock and then update the cache.

Let's diverge a little further. How do we implement a distributed write priority lock? If you understand the previous content, then this is just a matter of combining different implementations.

### Optimistic Lock

We mentioned optimistic locking before. Pessimistic locking and optimistic locking are often mentioned in the database field. The pessimistic locking mechanism is that the execution flow that has not obtained the lock needs to wait, while the optimistic locking does not wait. It relies on the version mechanism to determine whether the data version has changed during the execution process before the execution ends. If the data version is found to have changed, the entire operation needs to be rolled back and redone, otherwise it can be terminated directly.

Optimistic locking is more lightweight and is usually implemented using some kind of version mechanism, which is suitable for scenarios with few concurrency conflicts.

This version mechanism has more extensive and profound applications. For example, Google Spanner's percolator , which is used to implement distributed transactions , uses the version mechanism on the primary key to lock multiple records in a transaction range. Another example is that in the Paxos prepare-accept phase, the highest proposal number is used to reject other proposals, which is itself a locking mechanism.

## Conclusion
The interesting thing about discussing "Lock and Synchronization" is that you start to sense the idea of "Primitives". One thing that really impressed me was during an internal session at Tencent, where the presenter mentioned that much of the work in software development involves designing proper primitives as you gain experience. At the time, I didn't fully grasp the significance, but now, reflecting on my career, it makes perfect sense.

Starting with locks, we first implement a basic primitive—an "Atomic lock". This primitive can then be expanded into various specialized locks for different scenarios. As you connect all these related concepts, it becomes quite enlightening.

Cool! Let's wrap up here. Another closely related topic is "Lock and Memory Order". This concept is fundamental to creating a highly useful data structure known as a "Lock-free data structure".

Cheers! See ya!