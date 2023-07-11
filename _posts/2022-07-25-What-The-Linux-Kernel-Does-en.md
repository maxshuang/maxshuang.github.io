---
layout: post
title: What The Linux Kernel Does
subtitle: picture from https://www.pexels.com/search/wild%20animals/ 
author: maxshuang
categories: Linux-Kernel
banner:
  image: assets/images/post/what-linux-kernel-does/pexels-thomas-b-814898.jpg
  opacity: 0.618
  background: "#000"
  height: "95vh"
  min_height: "50vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Linux-Kernel Preface 
---

> In order to better design the resource management capabilities of the [Tiflow engine](https://github.com/pingcap/tiflow/tree/master/engine), I have read "Understanding The Linux Kernel" and have been thinking about the Linux Kernel's abstractions regarding resource management.
> 1. The more we want to improve resource utilization, the more we need to divide resources into smaller units.
> 2. The more we want to enhance resource management capabilities, the more we need to provide a holistic view.

## Background
Recently, our team has been developing a unified resource scheduling platform called Tiflow Engine for data migration. The platform aims to abstract the resource scheduling and failover capabilities of multiple products within the Data Platform team. From the perspective of DP product capabilities, what we actually hope for Tiflow Engine is a final form similar to stream processing frameworks like Flink. It should provide an end-to-end database synchronization flow through unified stream operators and user-defined functions (UDFs). As for why we are not directly using Flink and instead building a new platform, there are technical stack considerations and existing business transformation costs. We hope to gradually build a product with good stream abstractions and Cloud Native features by defining a roadmap. The purpose of this blog is not to discuss Tiflow Engine specifically, but rather the questions I have encountered while contemplating the ultimate goal of what we want to build, related to the Linux Kernel.

## Simplicity behind abstractions and encapsulation
Initially, when I started writing code from the main function, I thought the code was logically easy to understand. It executed sequentially starting from the main function entry, called various functions in between, and finally exited the main function when I achieved what I wanted to do. Adding some complexity involved creating parallel threads and introducing locks. However, after writing code like this for two years, I realized that there were some things that were difficult to understand. There was a thing called C Runtime Lib and complex data structure designs, such as the need for efficient memory allocation using tcmalloc or jmalloc, which involved per-thread variables and multi-level memory pools. I could understand them to some extent, but not completely. I always felt that there was something behind these designs that I couldn't fully grasp.
![simple-main](/assets/images/post/what-linux-kernel-does/linux-kernel-does-2022-08-06-2237.png)
[source](https://excalidraw.com/#json=5isqDNfuTgIXKWxAbZRHN,JzhT_W-8X1R3uscwcOkj8w)

Later, when I delved deeper into finding answers to these questions, I discovered that the underlying issue was abstraction and encapsulation. Modern high-level languages and operating systems provide very advanced abstractions and encapsulations to the point where when we write logic using high-level languages like C/C++, we always feel like we are in a purely simple context (our code executes sequentially from start to finish). This meets the demand for rapid development but makes it difficult to understand what exactly it is.

From the high-level language we write to the machine instructions that the CPU actually executes, there are several layers in between: the language runtime, compiler, linker, loader, executable file format, operating system process management, address space mapping, run queues, process switching, and only then can our main function be executed. It turns out that our code goes through such a long chain before it can be truly executed, which leaves me feeling overwhelmed when I need to optimize my program.

* If I don't truly understand the entire chain, how can I ensure that my optimizations are reasonable and effective?
* Among all these designs, how do I know if my design is better in principle? Have I achieved the best possible outcome?

I'm digressing a bit here, but in the case of linker and executable file formats, there are more conventions and module reuse behind them (the linker already involves mapping files into a linear address space). In this series of blog posts, we mainly discuss resource abstractions in the Linux Kernel.

## Memory/CPU abstractions in the Linux Kernel
For those who have experience with high-level languages, besides understanding the syntax provided by the language, another important concept to grasp is the idea of stacks and heaps in the context of processes—stacks for function calls, small space for stacks, and heaps for large space. Going deeper, we ask:

* Why do we abstract the concepts of stacks and heaps?
* What are the implementations behind stacks and heaps?
* Why is it that when we have an address in a 4GB address space, it can be different depending on whether it refers to the stack or the heap?

This is where we encounter the abstractions in the Linux Kernel concerning memory resources. On the other hand, from the perspective of writing code, the code is executed sequentially without interruption, and when it encounters blocking I/O, the execution flow waits and then resumes. It fits well with linear thinking habits. A good execution flow abstraction should achieve this level of simplicity. However, when we dive deeper into thinking:

* What does it mean for `the execution flow to wait and then resume`?
* What does the specific low-level implementation look like?

This is where we encounter the abstractions in the Linux Kernel regarding CPU resources. There is also the abstraction related to I/O, which we will cover later.

## What does the Linux Kernel do?
To understand the abstractions in the Linux Kernel regarding CPU and memory resources, I read the book "Understanding The Linux Kernel." The ingenious data structures, efficient cache designs, and register utilization in the kernel always make me exclaim with admiration.

Later, I asked myself, what is the essence of what the kernel really does behind all these complex designs? If I were to implement a kernel myself, what would it look like? This question is a bit daunting. What do I have at hand? I have:

* A CPU that can be driven by clock signals to continuously fetch and execute instructions in sequential order.
* A physical memory hardware with a linear address space ranging from 0 to N.
This feels somewhat similar to solving an algorithm problem. Given a memory address space from 0 to N, how would I implement an advanced and efficient memory allocator from scratch? How would I design the data structures to store metadata for the memory allocator? How would I organize the available memory space? It turns out that I, as well as the kernel, face the same simple hardware structure. However, the kernel provides super-efficient resource utilization abstractions and encapsulations, such as CPU time-sharing and linear address space, on top of this simple and comprehensible hardware. Even without knowing about these abstractions, programmers can accomplish simple tasks. (source: "what-linux-kernel-does")
![what-linux-kernel-does](/assets/images/post/what-linux-kernel-does/linux-does-2022-08-06-2346.png)
[source](https://excalidraw.com/#json=ZGufgOYmBZXjHklUQaw4x,r4fqGYMiBdpT3as5QoTxAg)

Thinking about this, the abstractions in the Linux Kernel regarding resource utilization are truly admirable.

## Summary and Reflections
A profound realization regarding the Linux Kernel's abstractions in resource management is:

1. The more we want to improve resource utilization, the more we need to divide resources into smaller units.
* For example, in the case of CPU time-sharing, for non-CPU-intensive tasks, dividing CPU time into small time slices and switching to other tasks when a task is waiting for resources improves the system's utilization of CPU resources.
* For example, in the case of process linear address space and operating system paging management, dividing the hardware memory space into small memory pages and only allocating physical memory pages when a process actually uses memory, while swapping cold data to disk, improves the system's utilization of memory resources.
2. The more we want to enhance resource management capabilities, the more we need to provide a holistic view.
* For example, in the management of process linear address space, exposing the operating system's memory paging management to users would require them to deal with page-level management of numerous small memory blocks with various permissions and attributes, which would be a disaster for users. Therefore, the process address space introduces a larger holistic view, such as a continuous linear address space of the entire 4GB, with different types of linear segments, such as read

## Recommended Reading
If you want to learn more about related topics, please read directly:
1. 《Understanding The Linux Kernel》Daniel P.Bovet & Marco Cesati  
2. 《Modern Operating System》Andrew S.Tanenbaum & Herbert Bos
3.  https://www.kernel.org/

Translated by ChatGPT.
