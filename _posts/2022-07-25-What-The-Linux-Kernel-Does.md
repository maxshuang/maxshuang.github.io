---
layout: post
title: (Script)What The Linux Kernel Does
subtitle: pic from 
author: maxshuang
categories: Linux-Kernel
banner:
  image:  
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Linux-Kernel 
---

最近团队在开发一个数据迁移的统一资源调度平台 Tiflow Engine，以便统一抽象 Data Platform 团队多个产品的资源调度能力和 failover 能力。从 DP 
产品的功能上看，我们其实希望 Tiflow Engine 的最终形态是类似 Flink 这种流式计算框架，通过统一的流式算子抽象和 UDF 构建端到端的数据库同步流。
至于为什么不直接使用 Flink 而是重新造一个轮子，这里有技术栈的问题，也有已有业务的改造成本问题。我们希望通过制定 roadmap 逐步构建具备良好
流式抽象和 Cloud Native 特性的产品。

这个 Blog 的目的不是讨论 Tiflow Engine，而是我在思考我们要做的最终东西的过程中想到的关于 Linux Kernel 的问题。最开始从 main 函数开始写代码
的时候，觉得写的代码在逻辑上是一个很容易理解的事情，从 main 函数入口开始顺序执行，中间调用各种函数，直到退出 main 函数结束，我完成了我想要
做的事情，再复杂一点就是创建一些`并行`的线程，加上一些锁。然后这样写了两年代码之后，我发现了一些有点难以理解的事情，会有一个 c runtime lib
的东西和很多复杂的数据结构设计，比如 malloc 效率不高需要使用 tcmalloc 或者 jmalloc，深入里面看就是各种每线程变量 + 多级内存池。可以理解，但
不完全理解，我总觉得这些设计背后有些内容我没有 Get 到。

后来深入去找这些问题的答案的时候，我发现了原来这些问题的背后都是抽象和封装的问题，现代高级语言和操作系统提供了非常高级别的抽象和封装，以至
于我们使用 C/C++ 这类高级语言写逻辑时总认为我们处于一个纯粹简单的上下文环境中，我们的代码从头顺序得执行到结束。满足快速开发的需求，但是却容
易让人不理解这是个什么东西。从我们编写的高级语言到 CPU 真正执行的机器指令，中间隔了语言运行时、编译器、连接器、装载器、可执行文件格式、操作
系统进程描述符、地址空间映射、运行队列、进程切换，才到我们编写的 main 函数执行。原来我们写的东西经过了这么长的链路才能被真正执行，那如果我要
优化我写的程序，我就有点崩溃了，如果我不能真正了解整个链路，我要怎么确保我的优化是合理且有效的，在这么多设计中，我如何知道我的设计在原理上
是更加好的，我做到了最好了吗？

说得有点远了，
