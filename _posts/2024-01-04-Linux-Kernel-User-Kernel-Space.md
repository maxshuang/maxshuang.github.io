---
layout: post
title: Linux Kernel-From User Space To Kernel Space
subtitle: picture from https://www.pexels.com/search/wild%20animals/
author: maxshuang
categories: Linux-Kernel
banner:
    image: /assets/images/post/linux-kernel-user-kernel-space/pexels-lucas-pezeta.jpg
    opacity: 0.618
    background: "#000"
    height: "70vh"
    min_height: "38vh"
    heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
    subheading_style: "color: gold"
tags: Linux-Kernel CPU
---

## 用户态到内核态切换
在之前的博客中，我们一直在介绍内存相关的话题，因为我觉得内存在一切逻辑和数据的载体，<font size=4>需要先搞清楚事物的存在形式，再讨论事物的组织形式和运作形式</font>。
所以我们从最核心的内核加载出发，介绍到物理内存页的管理和小内存分配，再到最后的内核给业务实体提供的抽象概念：进程线性地址空间。关于内存的一些核心概念我觉得大致上介绍完了，
注意实际实现还是很复杂的，需要考虑效率上的诸多细节。

这篇开始我们专注于 CPU 相关的概念，也就是事物是如何运行的。

## 用户态和内核态
这两个概念对于做后台开发的同学而言很熟悉，最常听见的说法是：要减少用户态到内核态的切换，避免不必要的开销。要说清楚这两个概念，我们也从几个问题开始：
1. 用户态和内核态是什么？
2. 用户态和内核态的存在形式是什么？
3. 用户态、内核态和内核的关系是什么？
4. 用户态切换内核态，内核态切换用户态都有哪些时机？


