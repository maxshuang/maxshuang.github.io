---
layout: post
title: Linux Kernel-Lock and Synchronization (Ongoing)
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

I've been pondering this topic for a long time and have written about it extensively. However, I was never quite satisfied. Recently, during some interviews, I found new inspiration.

My first encounter with the concept of a "lock" was in multi-threaded programming. Locks are essential tools to ensure data integrity when multiple concurrent processes access the same data.

If we ignore the implementation details and focus on the **lock primitives**, we can imagine a scenario where:
> Multiple people need to access supplies in a room in an orderly manner. They must lock the door, take what they need, and then unlock the door for the next person. This ensures that only one person can enter the room at a time, regardless of how many people are waiting.

As I delved deeper into programming, I encountered various types of locks, such as:
- Critical sections, read-write locks, and spin locks in the Linux kernel
- Row locks, gap locks, next-key locks, and intention locks in databases
- ReentrantLock, StampedLock, and Latch in Java

I even explored *lock-free/wait-free data structures* in high-concurrency scenarios.

I began to wonder:

**What exactly is a lock?**

**What are the underlying principles of existing locks? Why do locks incur overhead? How can we create new locks?**

## The Consensus in Synchronization

A lock in concurrency is fundamentally a consensus on mutually exclusive access within a specific context.

For example, imagine 10 people wanting to drink a glass of water, but only one person can drink at a time. If we stipulate that the first person to calculate `23451*1987` can drink, **then the calculation itself becomes a lock**, as it ensures mutually exclusive access in this context.

In programming, **locks are based on the consensus provided by the kernel context**. To understand what a lock is, we need to understand the consensus on mutually exclusive access provided by the kernel.

## Lock-free Scenarios

The simplest scenario is one that **does not require** locks or consensus on mutually exclusive access. 

In a multi-core environment, we can use per-CPU variables for concurrent streams. If the concurrent stream is multi-process, we use per-process variables; if it's multi-threaded, we use per-thread variables. 

**This way, we don't need to consider locks.**

The kernel sets per-CPU variables, ensuring that variables are not accessed by multiple CPUs simultaneously. A well-known example is the per-CPU runqueue in the Linux kernel.

There are also thread-local variables in user mode: `_thread int a`. Different threads have different copies, so they don't access the same memory.

After a user calls fork, the COW (copy-on-write) mechanism ensures that a deep copy is triggered when the data page is modified. This way, the parent and child processes operate on different resources, eliminating the need for consensus on mutually exclusive access.

To further explain the kernel context's consensus on mutually exclusive access, let's consider a **counterintuitive** scenario.

### Kernel Preemption

Theoretically, data security can be guaranteed for each CPU variable. However, in a scenario where kernel preemption is enabled, the kernel execution code can be interrupted, and the execution flow may be **preempted** by other processes.

Consider the following scenario:
```
Kernel Mode CPU0:

1. CPU0 executes the system call kernel path of Process 1, accessing CPU0's per-CPU variable `runqueue[0]`.
2. It reads Process Descriptor A and begins inserting Process Descriptor B after A in the linked list, but the operation is only partially completed.
3. A clock interrupt occurs, forcibly interrupting CPU0's execution flow to handle the clock interrupt service routine.
4. The interrupt service routine detects that Process 2 has a higher priority. Before exiting, it switches CPU0 to the kernel path of Process 2.
5. The kernel path of Process 2 reads Process Descriptor A, inserts Process Descriptor C after A in the linked list, and completes the operation.
6. CPU0 then schedules Process 1 to resume its previous kernel path and finish the remaining part of its operation.
```

Due to the kernel preemption mechanism, the CPU0 `runqueue[0]` data is corrupted by the alternate execution logic.

*In this scenario, is the pure per-CPU variable a consensus on mutually exclusive access?*

**Not yet.** 

Although it seems like it, the way the kernel context handles interrupts determines that **per-CPU variables + disabling kernel preemption + disabling interrupts** is a consensus on mutually exclusive access.

This example deepens our understanding of how consensus varies in different contexts.

## Atomic Lock (Atomic Variable)

After exploring lock-free scenarios, let's take a small step forward. Assuming that multiple concurrent streams need to access the same data, 

**what is the simplest mutually exclusive access consensus we can provide?**

One idea is to **stipulate a small block of memory** that, under certain mechanisms, the kernel context can **reach a consensus** on its mutually exclusive access.

For example, multiple threads and processes can reach a consensus on a 4-byte int variable, stipulating that all +1/-1 instructions for int operations are atomic. The thread that sets the int variable to 1 first occupies the resources; otherwise, it does not. 

When it does not occupy the resources, it returns an error to the upper layer, or the process is moved to the waiting queue.

This is a feasible assumption, but it needs to be more specific. 

Let's assume that all variables are only read and written in the main memory, there is no cache, and the processor provides CAS (Compare And Set) atomic instructions. This way, when each concurrent flow executes `cas(variable, old value, new value)`, it can be guaranteed that only one execution flow can occupy the resource at any time. This is a consensus on mutually exclusive access.

This is our **atomic variable**, or **atomic lock**. Its *read-modify-write* operation is **atomic** and **indivisible**. Concurrent execution flows can naturally operate serially on this memory block because it is the **smallest indivisible operation unit** (or **primitive**).

Based on such a lock primitive, we can expand it, allowing concurrent streams that occupy resources to do more things and then release the resources (set the int to 0).

*Are we reaching the end?*

**Not yet.**

The real kernel context is more complex. The actual situation involves different levels of cache, such as the memory hierarchy. The hardware system-level support for atomic variable semantics is complex. We can imagine several inconsistent scenarios:

1. The variable memory is **not aligned to** the native word size, or the variable memory is not within the memory operation range of one instruction, requiring multiple instructions to operate the variable. Multiple CPUs may alternate between instructions.
2. The simplest state change `a++` requires **three operations**: reading the value of variable `a` into a register, changing the register, and writing the register value back to memory. Multiple CPUs may alternate between operations, causing updates to be lost.
3. In the memory hierarchy, some storage is **CPU-specific**, such as registers and L1/L2 cache, while some storage is CPU-shared, such as L3 cache and memory. For shared storage, we can serialize memory access by locking the memory bus, but for unique storage, there may be old variable values in the hardware cache, causing inconsistencies.

Therefore, for the implementation of atomic variables, the following mechanisms are required at the system level:
1. **Architecture-dependent implementations**: Variables need to be aligned to the native word size to implement single write or single read atomic operations.
2. **Read-modify-write operations**: Shared memory provides a way to lock the memory bus to serialize access from different CPUs.
3. **Read-modify-write operations**: Dedicated memory provides inter-processor mechanisms, such as the MESI protocol, to ensure cache consistency.

Through these mechanisms, the kernel provides simple and intuitive atomic variable semantics, such as:

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
// lock memory bus+ invalidate the variables in other caches
lock addl $1, count(%rip)   
```

The kernel provides atomic operations such as:

```
atomic_read(v)                // Return *v
atomic_add(i,v)               // Add i to *v
atomic_sub_and_test(i, v)     // Subtract i from *v and return 1 if the result is zero; 0 otherwise
```

**Atomic variables provide reliable synchronization primitives**. 

All reads/writes/read-modify-writes on atomic variables are considered **instant** and **simultaneous** on multiprocessors. Thus, the operations of multiple processors on the atomic variable are synchronized, showing some order, such as:

```
define atomic_int v;
```

| ts      |  P1     |   P2   |    P3  |
|---------|---------|--------|--------|
| t1      | read v  |        |        |
| t2      |         |        | write v|
| t3      |         | r-m-w v|        |
| t4      | write v |        |        |

Computer experts have helped us take a significant step forward in the history of locks. They have reached a reliable consensus on mutually exclusive access in the kernel context. This consensus is based on hardware system instruction control, such as memory bus locks and the MESI cache consistency protocol. These hardware instruction details are encapsulated in the compiler based on the hardware system.

## The Implementation of Locks

Based on the reliable consensus provided by atomic variables, we can explore the true nature of various locks used in daily programming and address the issue of lock overhead.

### Exclusive Lock and Critical Section

If you have experience with multi-threaded development, you are likely familiar with mutex locks, which are often the starting point. The classic interface of a mutex lock is as follows:

```
Mutex mux;
mux.lock(); // or mux.tryLock();
...
mux.unlock();
```

But **what exactly is a mutex lock**?

When you ask this question, you might already feel uneasy about the operating system's black box.

To understand this, consider the evolution of computing. Initially, there was only a CPU executing instructions sequentially and continuous physical memory. To address CPU speed issues, we introduced multi-process multi-threading and time-sharing multiplexing. To handle memory locality, we introduced virtual memory and paging mechanisms.

With these complex concepts in place, we now need a mutual exclusion lock to ensure that different concurrent flows can reach a **consensus**: 

**only one execution flow can access resources or continue execution at a time.**

**How can we achieve this consensus?** 

=> The atomic lock, implemented by the kernel, provides this *consensus*, which is why we call it a primitive.

Atomic locks or atomic variables ensure that any operation on an atomic variable (essentially a small block of memory) is *instant* and *simultaneous* to all observers, regardless of the operating system's cache structure.

With this primitive, before concurrent execution flows access a large block of shared memory, they read and operate on an atomic variable, such as using `cas(varA, 0, 1)`. Because the `cas` operation is guaranteed to be atomic at the hardware level, only one execution flow can set `varA` to 1. This execution flow can then continue to operate.

**What happens to the concurrent execution flows that fail to set `varA` to 1?**

Atomic variables ensure that concurrent execution flows reach a consensus, but those that **do not** obtain permission need to take certain actions, which are implementation-dependent. 

In the Linux kernel, this action is **blocking**.

Blocking involves hardware context switching, where the context related to the execution flow is saved in memory, and the flow waits for the next scheduling. This creates the feeling that the execution flow is **blocked** and **waiting**.

When the blocking wait ends, the waiting execution flows compete for the mutex lock again. 

**What happens then?** 

Imagine that after the hardware context of all waiting execution flows is switched, their execution flow IDs are maintained in a queue or list. When the execution flow holding the lock releases it, these flows are **awakened**. 

Awakening means adding the execution flow from the waiting or sleeping queue to the runqueue, waiting to obtain CPU time for execution. Sometimes, this causes **herd shock**, where all waiting flows are awakened at once, leading to inefficiency. The kernel can use better algorithms, such as selecting the first waiting flow according to FIFO.

To summarize:
1. Atomic variables ensure that concurrent flows **serialize** their resource possession behavior. Only one execution flow can possess resources, and this behavior is immediately visible to other concurrent flows.
2. Execution flows that do not occupy resources are **suspended** by the kernel and placed in the lock waiting queue. They are awakened after the lock is released.

**The overhead of blocking locks** includes the synchronization protocol overhead of atomic variables in multi-level caches, process suspension, wake-up, and scheduling delays.

### Spin Lock

A spin lock involves waiting for the lock by polling when lock possession fails. Its classic interface is similar to a mutex lock:

```
SpinLock lk;
lk.lock(); // or lk.tryLock();
...
lk.unlock();
```

If you understand mutex locks at the kernel level, spin locks are straightforward. 
- First, atomic variables need to **reach a consensus**, 
- Then the execution flow that does not hold the lock will not be context-switched but will **execute some nop instructions** and check the atomic variables again.

Spin locks have **faster scheduling speeds**, possibly in the ns or us range, and do not require sleep waiting for wake-up. 

However, long-term instruction idling wastes CPU cycles. Spin locks are suitable for scenarios where atomic variables are already in caches, the protected code area is short, and concurrency conflicts are minimal, making them common in the kernel.

User-mode spin locks, which encapsulate *atomic variables, nop, and while operations*, may not be very meaningful in terms of performance. Kernel spin locks disable kernel preemption and shield interrupts to prevent execution flow switching. User-mode spin locks cannot avoid being switched out by time-sharing systems, leading to ms-level scheduling delays if the execution flow is switched.

### Condition Variable

Condition variables are another crucial primitive provided by the kernel. They enable the development of locks with **higher-level semantics**, such as semaphores, read-write locks, and fence latches. Unlike mutex locks, condition variables make **waiting, condition checking, and notification explicit**. The classic interface is as follows:

```
ConditionVariable cv;
Lock lk;
cv.wait(lk, condition);
...
cv.notifyOne(); // or cv.notifyAll();
```

With condition variables, multiple execution flows can call `cv.wait(condition)` simultaneously, explicitly waiting for a specific condition to be met. When the condition is met, the execution flows continue; otherwise, they wait until `notifyOne()` or `notifyAll()` is called to wake them up.

Essentially, a condition variable is a consensus mechanism combined with **condition checking and explicit notification**. 

The mutex lock provides the consensus part. After acquiring the mutex lock, the condition is checked. If the condition is met, execution continues; if not, the execution flow is added to the waiting queue, the lock is released, and it waits to be awakened by `notifyOne()` or `notifyAll()`.

Using custom conditions, we can construct *rich synchronization semantics*. For example, if the lock check condition is `count > 0` and `count` is initialized to 6, then when calling `wait()`, any execution flow that satisfies `count > 0` can proceed. This is the **semantics of a Semaphore**. The general code implementation is as follows:

```
int counter = 6;
std::mutex mtx;
std::condition_variable cond_var;

void request_resource() {
    std::unique_lock<std::mutex> lock(mtx);
    // wait for counter > 0
    cond_var.wait(lock, []{ return counter > 0; });
    // get resource, decrease counter
    --counter;
}

void release_resource() {
    std::unique_lock<std::mutex> lock(mtx);
    // release resource, increase counter
    ++counter;
    // notify the waiting thread
    cond_var.notify_one(); // or use notify_all()
}
```

### Semaphore

A semaphore allows multiple execution flows to occupy resources simultaneously, and those that cannot obtain the resource must wait. Its classic interface is:

```
counting_semaphore semaphore;
semaphore.acquire();  // or wait()
semaphore.release();  // or signal()
```

Semaphores use two mechanisms:
1. An atomic variable `count = n` ensures that only `n` execution flows can occupy resources, and this behavior is immediately visible to other concurrent flows.
2. Execution flows that do not hold resources are suspended.

**Why provide semaphore primitives when semaphores can be implemented using condition variables?**

1. *Different design purposes*: Condition variables are for synchronization **under complex conditions** and require mutex locks to protect condition checks. Semaphores control concurrent access to limited resources and **rely only on an atomic counter** without needing complex condition checks.
2. *Different lock requirements*: Condition variables **rely on locks** for safe condition checks, allowing flexible lock mechanisms like shared locks. Semaphores **only need atomic counters** and kernel wait-wake mechanisms.
3. *Performance and efficiency*: Condition variables must lock each time the condition is checked, involving user-kernel state switching. Semaphores use **atomic counters** in user state and switch to kernel state only when `counter == 0`.

For concurrent access to limited resources, such as in the producer-consumer model, **semaphores are more efficient**, while condition variables offer more flexible synchronization strategies at the cost of performance.

### Advanced Lock

Let's explore advanced locking mechanisms based on synchronization primitives like atomic locks, condition variables, and semaphores.

#### Latch

A latch, or fence, describes a synchronization scenario where a concurrent flow A **waits for multiple concurrent flows** B, C, and D to reach a synchronization point before executing. Its classic interface is:

```
latch lt(5);
lt.count_down(); // internal --counter;
lt.wait();       // wait for counter==0
```

Imagine a latch implementation with `counter = 3`. Concurrent streams B, C, and D wait for `counter > 0`, decrement the counter, and call `notifyAll()` when finished. Concurrent stream A waits for `counter == 0` and is awakened to execute once B, C, and D are done.

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

A read-write lock is a commonly used synchronization mechanism to improve read-write concurrency. Semantically, there is no conflict between read operations, but there are conflicts between read-write, write-read, and write-write operations. It is suitable for scenarios with more reads than writes.

We can imagine the implementation of a read-write lock like this: there is a `runningReaderCounter` that maintains the number of readers currently running, a `waitingWriterCounter` that maintains the number of writers waiting, and an `isWriterRunning` that maintains whether there is a writer currently running.
1. The condition for a reader to run is that there is currently no writer running and no waiting writer: `isWriterRunning == false && waitingWriterCounter == 0`.
2. The condition for a writer to run is that there are neither running readers nor running writers: `isWriterRunning == false && runningReaderCounter == 0`.

**Write-Priority Read-Write Lock**

The following is a write-priority read-write lock implementation, which means that for the concurrent sequence of requests - `read 0-read 1-write 0-read 2-read 3-write 1`, if write 1 waits before write 0 is finished, write 1 will be scheduled first, and the execution sequence is - `read 0-read 1-write 0-write 1-read 2-read 3`.

```cpp
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

**Read-Priority Read-Write Lock**

Let's implement a read-priority read-write lock. In this case, the reader's waiting condition becomes that it can be executed as long as there is no running writer: `cond.wait(lock, [this]() { return !writerActive; })`.

For this condition, the most intuitive result is that after the writer finishes running, both the waiting writer and the waiting reader have the opportunity to run. **Once the reader gets the opportunity to run, the waiting writer must wait for all readers to finish running before running**, so we call it *read priority*.

For example, for a concurrent sequence such as the request sequence `read 0-write 0-read 1-write 1-read 2-read 3-write 2`, its execution sequence may be `read 0-read 1-read 2-read 3-write 1-write 0-write 2`.

```cpp
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

**Fair Read-Write Locks**

Fair read-write locks balance the ratio of read and write requests, such as using a flipper to **alternately select** read threads and write threads, or putting readers and writers into a FIFO queue in the order of requests, and selecting whether to execute a read thread or a write thread based on the queue head element.

```cpp
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
        preferWriter = !preferWriter;
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
        preferWriter = !preferWriter;
    }

    // Writer unlock (exit write mode)
    void unlockWrite() {
        std::unique_lock<std::mutex> lock(mtx);
        writerActive = false;
        cond.notify_all();  // Notify all readers and writers waiting
    }
};
```

**Fair Read-Write Locks Scheduled in Request Order**

```cpp
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

Let's explore a more advanced concept: the distributed lock. Unlike traditional locks limited to the Linux kernel and a single machine, distributed locks provide locking services for entities in a distributed environment.

Interestingly, distributed locks can help us better understand the **semantics of lock primitives**. Previously, we mentioned that locks are essentially **a form of consensus**. For instance, atomic variables are implemented as lock primitives at the kernel level through various complex hardware and software mechanisms.

In a distributed environment, the lock primitive becomes clearer because entities interact with the lock service *through network interactions*. As long as the lock service can provide a **consensus** under network interactions, it suffices. This consensus can be as simple as ```a==2``` or ```set a=1 if a==0```. Essentially, for mutual exclusion locks, it only needs to ensure that only one entity holds the result of a successful network request at any time.

This is why high-performance caches like Redis are often used to implement distributed locks. Redis operates in a single-threaded manner, and by maintaining a version as an optimistic lock, it can naturally implement the CAS interface and the lock primitives mentioned earlier.

Another example is using a database's row lock combined with a where clause to implement lock primitives, such as "update a=1 where a=0". This works because only one updated row will be affected. Alternatively, you can add a version column to your records to achieve similar capabilities.

Distributed locks can be categorized into fair and unfair distributed locks. Although this may not be a standard classification, it helps to understand the differences:
1. **Fair distributed lock**: Locks are obtained in the order of requests. For example, distributed locks implemented by Zookeeper or etcd use a queuing mechanism. The first element in the queue is considered to have obtained the distributed lock. By maintaining the session, it ensures that if the first element disconnects, the next element in the queue becomes the first and obtains the lock.
2. **Unfair distributed lock**: All entities compete for a certain value using the CAS mechanism. For instance, the entity that successfully changes 0 to 1 after performing an auto-increment operation obtains the lock. This competition depends on network latency, CPU, and other conditions of the entities in the distributed environment. The best-performing entity wins, making it an unfair distributed lock. This type of lock can be advantageous in certain scenarios because it reflects the real conditions of the distributed environment. For example, when updating the cache, it is better to let the entity with the lowest latency at that moment obtain the distributed lock and update the cache.

To further illustrate, how would we implement a distributed write-priority lock? If you understand the previous content, it is simply a matter of combining different implementations.

#### Optimistic Lock

We mentioned optimistic locking before. Pessimistic locking and optimistic locking are often mentioned in the database field. The pessimistic locking mechanism is that the execution flow that has not obtained the lock needs to wait, while the optimistic locking does not wait. It relies on the version mechanism to determine whether the data version has changed during the execution process before the execution ends. If the data version is found to have changed, the entire operation needs to be rolled back and redone, otherwise it can be terminated directly.

Optimistic locking is more lightweight and is usually implemented using some kind of version mechanism, which is suitable for scenarios with few concurrency conflicts.

This version mechanism has more extensive and profound applications. For example, Google Spanner's percolator , which is used to implement distributed transactions , uses the version mechanism on the primary key to lock multiple records in a transaction range. Another example is that in the Paxos prepare-accept phase, the highest proposal number is used to reject other proposals, which is itself a locking mechanism.

## Conclusion

The interesting thing about discussing "Lock and Synchronization" is that you start to sense the idea of "Primitives". One thing that really impressed me was during an internal session at Tencent, where the presenter mentioned that much of the work in software development involves designing proper primitives as you gain experience. At the time, I didn't fully grasp the significance, but now, reflecting on my career, it makes perfect sense.

Starting with locks, we first implement a basic primitive—an "Atomic lock". This primitive can then be expanded into various specialized locks for different scenarios. As you connect all these related concepts, it becomes quite enlightening.

Cool! Let's wrap up here. Another closely related topic is "Lock and Memory Order". This concept is fundamental to creating a highly useful data structure known as a "Lock-free data structure".

Cheers! See ya!