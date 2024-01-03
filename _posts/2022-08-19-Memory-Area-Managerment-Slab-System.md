---
layout: post
title: Linux Kernel-Slab System And Hardware Cache
subtitle: picture from https://www.pexels.com/search/wild%20animals/ 
author: maxshuang
categories: Linux-Kernel
banner:
  image: /assets/images/post/linux-kernel-memory-area-slab-system/pexels-mike-b-97824.jpg 
  opacity: 0.618
  background: "#000"
  height: "70vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Linux-Kernel Memory
---

## slab 对象内存分配器
前面一篇博客我们从比较大的方面讲内核是如何管理和利用物理内存的，这篇博客我们讲在一个连续的内存区间中，内存是如何被管理的。
其实细想下就可以理解，实际用户和内核很少会按照页的模式申请内存，反而是按照<font size=4> 对象的大小 </font> 申请内存，而一般的对象可能都在 100 bytes
以内，所以如何将一个连续的内存块进行切分，提高内存利用率的同时减少内部碎片就成了一个研究课题。我们今天讲的 slab system 还会
涉及到 L1/L2/L3 高速缓存硬件。

连续内存区间的内存管理其实我们一直都在接触，比如我们平时调用 New/Delete 和 malloc/free 的时候其实底层就是一个 ptmalloc
的内存池，算法大体上是先分配一块大的连续内存，然后以某种方式切分成更小的对象，比如按照之前说的 2^order 作为块大小，维护不同
size 内存块的链表。

所以 slab system 本质上也是一个关于将大的连续内存块切分成小对象的算法，但是 slab system 在设计需求上和常规内存分配还是有点不同的。 
* slab system 是为了满足内核关于小对象的分配需求，而内核需要分配内存的对象是非常有限的，常见的比如维护进程状态的 struct task,
虚拟内存相关的 struct mm 等，所以我们可以专门为固定大小的对象构建内存池。
* 常规内存池由于业务形态复杂，无法预估对象大小，所以一般按照 2^order 作为不同内存块的大小，这种做法通常会导致不同程度的内部碎片，因为我们在
分配合适的内存对象时总是向上取整的，导致有部分内存总是无法被利用的。而 slab system 是针对特定对象设计的，大小刚刚好。

slab system 是管理内存小对象的算法，对于每种对象都会对应一个 slab 分配器。在实际的内存池设计中，增加了<font size=4>高速缓存</font>的概念(注意这是专指软件层面小对象使用的缓存)。
其实就是 slab 分配器的上层管理对象。一个高速缓存对应固定大小的对象，拥有多个 slab 分配器，一个 slab 分配器中有多个对象。高速缓存负责将完全空闲的 slab 
回收掉，释放底层的物理页框，当所有的 slab 分配器都没有空闲对象时则负责新建可用的 slab 分配器。  
![logical address](/assets/images/post/linux-kernel-memory-area-slab-system/cache-and-slab.png)  
[source](https://excalidraw.com/#json=R_YI9fKV39sZuRzAOUWCB,PjWo5b3s6I67S3HZ_J3LkQ)


slab 分配器在创建时被分配了连续的页框作为对象分配资源。slab 分配器自身的存储有两种模式，一种是存储在被分配的连续页框开头的位置，这种
方式需要内部碎片有足够的空间，这是可以提前计算出来的。  
举个例子：  
如果每个 slab 分配器都被分配了 2 个连续的物理页框，也就是 8K，假设这个高速缓存中所有的对象大小为 64 bytes，每个对象的对象描述符占用 4 bytes，则最后剩余的内部碎片为 (8\*1024)%(64+4) = 32 bytes。
另外一个方式就是存储在被分配页框之外的空间中。

对于对象大小，由于高速缓存硬件(注意和高速缓存是不同的概念) 以 <font size=4>cacheline</font> 作为缓存单元，而 cacheline 一般都是 64 bytes，所以需要根据对象大小
做内存对齐。比如如果小于 64 的一半，就按 64 的因子进行对齐，否则就按 64 的倍数进行缓存。

### slab 分配器元数据管理
slab 分配器将自己拥有的连续内存块划分成连续的多个对象，每个对象都有一个对象描述符。目前的对象描述符就是一个无符号整数，通过保存下一个空闲
对象的索引实现空闲对象链表，而 slab 分配器中则保存了空闲对象链表头对象的索引。所有的对象描述符和 slab 分配器存储在外部或者内部。当 slab 分配器
分配空闲时，总能以 O(1) 的复杂度知道没有空闲对象或者分配空闲对象。当释放对象时，直接在对象上调用指定的析构函数，将对象的索引设置成空闲对象
链表头，这样下次分配空闲对象时该对象可能还在高速缓存硬件中，这样 CPU 就不用再去内存中加载对象再更新。同时，如果新分配的空闲对象已经在高速缓存
硬件中，则对于 write back 这种写数据方式，可以直接更新高速缓存硬件中的数据即可。  
![logical address](/assets/images/post/linux-kernel-memory-area-slab-system/slab-object-descriptor.png)

### slab coloring 机制
接下来我们要讲的 slab coloring 也和高速缓存硬件有关，所以我们先讲下高速缓存硬件的缓存方式。首先 L1/L2/L3 高速缓存硬件的缓存单位是 cacheline，
一般是 64 字节大小。我们知道，计算机系统的中内存是个山型结构，
* 距离 CPU 最近的是寄存器，数量大概只有 16 个，大小一般是 16 bit 或者 32 bit，存取
时间几乎为 0 个时钟周期。
* 下一级是 CPU 独有的 L1/L2 硬件高速缓存，大小一般从 4K～100K 左右，访问时间为 0～10 个时钟周期；
* 再下面是更大的公用的 L3 硬件高速缓存，大小可能达到 1M～20M，访问时间为 30 个时钟周期左右。
* 再下一级就是物理内存，大小一般为 2G～8G，访问时间大概为 100 个时钟周期。  

对于不同的存储层次而言，每次存取数据的大小也不同，对于 L1/L2 硬件高速缓存而言一般是以 64 Bytes 为单位。存储层次越往下走，每次存取的数据量就越大，
这样就能通过均摊的形式降低长交互时间带来的数据延迟。这个类似于在高延迟网络交互中通过一次传输多个数据包提高吞吐，来降低单个数据包延迟的原理。

#### cacheline 硬件缓存机制
我们说 L1/L2 硬件高速缓存以 cacheline 作为缓存的单位，并不是说将 L1/L2 硬件高速缓存整个连续的内存空间直接划分成 64 bytes 为单位的小内存块，因为
我们是需要元数据去维护具体某个 64 bytes 的缓存是属于那个物理内存块的缓存。  
举个例子：  
假设 L1 只有 64K 的内存空间，而物理内存有 4G 的内存空间。规定 L1 按顺序映射所有内存，那么必然每间隔 64K 大小内存块就要复用一次 L1 相同位置的内存空间，
此时可以用 $address%64k 的方式确定具体某个字节在 L1 中的位置，但是还需要其他信息反向定位 L1 中具体某个位置对应的是内存中的第几个 64k 内存块的位置。所以需要在缓存数据的基础上增加元数据维护反向关系和缓存有效性。

实际一个通用的硬件高速缓存结构是这样的，首先分成 E 个组，然后每个组内有 S 个 cacheline，每个 cacheline 缓存 B 个字节，所以除了元数据之外，我们可以知道
整个硬件高速缓存中数据缓存的大小为 E\*S\*B 。在每个 cacheline 中，需要 1 bit 标识这个 cacheline 是否是有效内容，缓存策略是在分组内随机映射（找到一个空闲位置就行），
所以 (32-logE-logB) bit 作为标识符标记这个 64 bytes 是属于哪份内存块的。  
![logical address](/assets/images/post/linux-kernel-memory-area-slab-system/cacheline-structure.png)

我们用一个实际地址内存块的硬件高速缓存查找流程作为地址就比较容易理解这个过程。
假设 L1 数据缓存大小为 64K，每个 Cacheline 64 bytes，总共有 2^8 = 256 组，每组中有 2^2=4 个 cacheline。
给定一个物理地址 0x12003233, 需要确定这个物理地址开始的 4 字节是否在 L1 中。
物理地址的最低 6 bit 表示该地址在 64 bytes cacheline 中的偏移，紧接着的 8 bit 表示该地址被映射到组中，剩余的 (32-6-8)= 18 bit 唯一标识这个地址，
```
0x12003233: 0001 0010 0000 0000 0011 0010 0011 0011

000100100000000000 11001000 110011
------------------ -------- ------
    address flag    group   offset

address flag: 0x04800
group: 0xC8
offset: 0x33
```
此时硬件单元要检查的就是在 L1 硬件缓存的 0xC8 分组中是否存在 0x04800 标识，并且对应的 cacheline 是否是有效的，从而判断是否需要从下级内存单元中重新加载数据。

这里有个小细节，就是为什么我们要将中间的 8 bit 作为分组，而不是最高的 8 bit 作为分组。将中间的 8 bit 作为分组之后，我们可以看到，连续的 64 bytes 块
被映射到了不同的分组中，比如第一个 64 bytes 映射到分组1，第二个 64 bytes 被映射到分组2， 以此类推。也就是一个跨越了多个 cacheline 的连续内存块有可能都被缓存到了 L1 中。同时，使用中间 8 bit 分组后，可以看到每隔 2^(6+8) = 16K，
中间 8 bit 才会相同，由于程序访问内存的 locality 原理，此时替换掉之前的同一个组中的缓存也不会造成较高的缓存不命中。

如果我们使用高 8 bit 作为分组，则连续的 2^(32-8)=16M 内存空间都会被映射到一个组中，而一个组中只有 4 个 cacheline，就会造成连续的内存空间不断刷新 L1
缓存导致 CPU 需要频繁从物理内存中加载数据，每次加载数据起码都是 100 个时钟周期的延迟。

讲完了硬件高速缓存的缓存方式，我们就可以理解为什么 slab system 中的对象要根据 cacheline 大小进行对齐了，主要是保证<font size=4>一个 cacheline 中能保留多个对象</font>，
或者一个对象跨越多个 cacheline 时，也<font size=4>有可能都在硬件高速缓存中，可能横跨多个分组</font>。  
![logical address](/assets/images/post/linux-kernel-memory-area-slab-system/memory-cacheline-mapping.png)  
[source](https://excalidraw.com/#json=mjFFSsU0zuED_baRRjczu,eoJa57dZ_HgluYx-q3MHmw)

现在我们来看一种场景，一个高速缓存内有多个 slab 分配器，每个 slab 被分配了 4 个连续的物理页框。此时从硬件高速缓存的角度看，<font size=4/>每个 slab 分配器如果对象
的布局完全一样，则相同偏移的对象会被映射到同一个组中，此时就可能出现不同 slab 分配器对象不断刷新同一个组缓存的场景</font>。所以为了避免这个现象，slab 分配器
中加入 slab coloring 的机制。slab coloring 机制其实就是在连续内存块的头部加上不同的偏移，这个需要能跨过不同的组，也就是至少 2^6 bytes 才能将不同 slab
分配器中的对象岔开到不同的组中。

从原理上我们可以理解 coloring 机制存在的意义，但是我个人比较难理解的是不同的 slab 分配器中对象的释放和分配是随机或者存在某种分布规律，很难评判是否真的存在多个 slab 分配器对同位置对象缓存频繁操作的场景。

### 小细节
又来讲下鸡生蛋蛋生鸡的问题。slab system 依赖 cache 结构和 slab 结构作为元数据管理不同小对象的内存分配，也就是说这些元数据是需要根据新增的不同内核对象动态增加的，数量上也是不可预知的，所以每次都手动构造
比较费力。所以内核只手动构造了 cache 和 slab allocator 的 slab allocator, 有点绕口，就是说 cache 结构和 slab 结构也是从对应的 slab allocator 中构造出来的。因为  cache 结构和 slab 结构大小是
固定的，种类也只有 2 个，所以方便手动构造并初始化。

## 总结
1. slab system 用于为内核中小对象分配内存，内核中有限对象数量决定了 slab 分配器使用固定大小对象的特点。
2. 为了提高 L1/L2 缓存效率，slab system 中对象要进行内存对齐，同时使用 coloring 机制错开不同 slab 分配器中相同位置可能存在的缓存窃取。

