---
layout: post
title: Simple-Paxos 
subtitle: picture from https://www.pexels.com/search/wild%20animals/
author: maxshuang
categories: Distributed-System
banner:
  image: /assets/images/post/replication-simple-paxos/pexels-dsd.jpg
  opacity: 0.618
  background: "#000"
  height: "55vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Distributed-System Replication
---

# Background

这篇博客主要讲下 Simple-Paxos 算法，它属于分布式系统多备份一致性(consistency)的领域, 这里的 consistency 说的是 recency requirement，即多备份场景下数据的 newest write 会被什么时候看到。

分布式系统多备份源于人们对于 fault-tolerant system 的需求。单份数据时代人们对 durability 的要求是数据能写入到 non-volatile 的介质中(HDD/SSD)，后来人们发现如果出现了磁盘故障无法恢复，机房火灾，地震等不可抗自然因素下，数据还是*无法恢复*。

于是，对数据 durability 的要求自然变成了=> 同一份数据是否可以保持*多备份*，复制到不同的机房，不同的物理区域，甚至是不同的国家，这样即使丢失部分备份，也可以使用*其他备份恢复系统*。

多备份自然会引入备份一致性的问题，这是一个数据一致性和效率的 tradeoff，*备份一致性越弱，写的效率越高*，因为不需要确认其他备份同样应用了 write operation。

一般我们会说一个系统的多备份是：
1. 强一致(strong consistency): 读写所有备份和读写单一数据一样，newest write 会**被立刻看到**。
2. 弱一致(week consistency): newest write 会**在一个时间限定范围内**被看到，比如系统提供保证 "newest write 在 5s 后一定会被所有用户看到"。
3. 最终一致性(eventual consistency): newest write **最终**会被看到，但没有限定时间范围。

强一致不代表每个 write 都要等待所有备份都确认，因为这样只要有一个备份无法访问，整个系统就无法 write 了。现在的强一致基本都是采取*多数派读写(majority read/write)*，majority read 涉及的节点集合和 majority write 的节点集合至少有 1 个是重合的，提高了系统 fault-tolerant 能力的同时，也能读到 newest write。

# Paxos consensus algorithm

Paxos 共识算法的目标是：   
在没有中心节点或者手动指定中心节点的场景下，多个节点可以对**同一个值**达成共识(consensus)(多数节点认可同一个 value)，比如 3 个人中至少有 2 个人认可 A 作为 leader。
同时，Paxos 算法可以保证在少于一半节点失败时系统仍然可以对同一个值达成共识。

> NOTE:   
> 在目标中要特别注意的是**同一个值**，Paxos 解决的是比如 *leader 是谁*这种问题，它无法直接用来做日志同步等场景。

Paxos 共识算法适用的场景是:   
1. Non-Byzantine 系统，即系统中不存在恶意欺骗，损坏的消息可以被检测出，丢弃或者恢复。
2. 分布式环境，节点和节点间沟通需要依赖不可靠的网络环境，可能出现消息丢失/消息重复/消息乱序。

Paxos 是我最喜欢的算法。在多个节点可以自由设置初始值的前提下，通过沟通交互就能对同一个值达成共识，有点不可思议，但是 Paxos 的整个推理过程又是那么得合理。

可以说是，**再一次在不可靠机制上实现了可靠的机制**。

我们跟随 [Paxos Make Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) 这篇论文的脚步看下 Paxos 推理和算法。

## Paxos 讨论
之所以一上来先介绍 Paxos 的推理，再到算法，除了和原始论文保持一致之外，还想要展现出该算法展现出的 logical thinking 能力，这是这个算法我觉得最吸引人的地方。

如果大家在没有学习过 Paxos 之前思考过多节点共识问题，就能体会到这个问题的难度，因为多节点在初始化阶段是可以自由设置初始化值的，所以采用哪个节点提交的值就是一个问题。

转化成实际生产问题就是，如果系统是多主的(multi-master)，多主同时对一个数据进行操作，如何保证整个系统所有节点对该数据的认知是一致的，比如所有节点对该数据的查询都能返回一致的结果。

一开始的系统会尽量避免这个问题，因为 multi-master 的冲突很难解决，一般需要人工干预。所以系统会采用 single-master 架构，然后采用同步或者异步复制到 slave nodes。即使是现在跨城 multi-master 的架构，也只是用做异步容灾，只提供最终一致性，并且需要在数据上做逻辑隔离，比如 Datacenter A 数据只能给 City A 的居民读写。 

回到 Paxos 算法推理本身，在我看来，整个推理过程有个核心的点 => **多节点间逻辑时钟**的构建。

这个核心思想和 Lamport 的另外一些论文《[Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)》有密切关系，它讨论的问题就是多节点在不需要全局时钟的情况下如何构建全局次序。这个全局次序也叫全局逻辑时钟，它在分布式事务中非常重要，因为没有全局时钟就没法界定事务之间的顺序，就可能导致数据异常。

*全局逻辑时钟*的一个核心点就是通过*局部因果关系*(不是太准确的词，在考虑中)构建全局次序。举个例子：
```
node1 and node2 are two different nodes with different local clock in the system

initial:
node1(clock): 1 2 3 4 5 6 7 8 9 10
node2(clock): 1 5 10 15 20 25 30 35

operation:
node1: 1 2 3 4 5(r1) 6 7 8(w2) 9 10
node2: 1 5 10(w1) 15(r3) 20 25 30 35
In this case: 
1. node1 has two operations: r1 and w2。r1 happens at clock 5, w2 happens at clock 8.
2. node2 has two operations: w1 and r3。w1 happens at clock 10, r3 happens at clock 15.
we have no idea about the global sequence of r1,w2,w1,r3 because their clocks are not synchronized.


operation:
node1: 1 2 3 4(r1') => 11(r1) 12 13(w2') => 31(w2) 31 32
            /                   /
            /                   /
node2: 1 5 10(w1) 15(r3) 20 25 30 35

In this case: 对于 node1 的所有操作发起之前，都先对 node2 发起一次获取 node2 clock 操作，获取 node2 上最新的 clock，从而设置 node1 读写操作的 clock。通过这种方式，我们获得了以下可靠的跨节点逻辑次序：
w1(10) => r1(11)
w1(10) => r3(15) => w2(31)
原因很简单，比如对于 w2，它在操作之前看到了 node2 的时钟已经是 30，而 node2 自己的时钟是一直单调递增的，所以在 w2 发生之前，w1(10) 和 r3(15) 必定已经发生了，所以可以通过比较逻辑时钟 10<15<31 直接确定跨节点事件发生的先后次序。
```

Paxos 推理中也隐含了这种利用*局部因果关系*构建全局时钟的思想。

## Paxos 推理

### 算法性质
首先，Paxos 是一个 consensus 算法，它是系统中多节点(nodes)对单个值达成共识，即系统中 majority nodes 接受(accepted) 了值 v0，我们说值 v0 被整个系统选中(chosen)。

> NOTE:  
> accepted： 只有 node 才能 accepts value，同时节点也没有全局视角，不知道某个 value 是否被 chosen 了。  
> chosen: chosen 这个行为没有对应的实体执行，它是个全局视角下的认知，也就是当系统中 majority nodes accepted 相同的值 v0 后，这个值 v0 就自动成为 chosen value。

对于算法的正确性，我们需要定义算法的 safety requirements:
![alt text](/assets/images/post/replication-simple-paxos/image.png)

也就是：
1. 只有被提议的值(proposal)才可能会被系统选中(chosen)。  
简单说就是，chosen value 只能从 proposal value 中选择，不能从其他途经或者凭空捏造。  
2. 有且只有一个 value 会被选中 chosen。  
系统中不能出现 majority nodes accepted proposal value v1, 然后又出现 majority nodes accepted proposal value v2(v2!=v1)。
3. 只有一个值 v0 成为 chosen value 之后，观察者才可以认为值 v0 被 chosen 了。  
系统没有达到共识之前，观察者不能自己错误认为系统已经存在 chosen value。

### 角色定义
系统存在 3 种角色(role)： proposer, acceptor 和 learner。
* proposer: 负责提出提案(proposal)，proposal 中包含一个 proposal value，表示 proposer *希望* 被系统 chosen 的值。
* acceptor: 负责接受(accept) proposal，相当于委员会成员。
* learner: 观察者，不影响算法流程，只是持续学习(learn) 系统的 chosen value。

node 和 role 是俩套概念，没有一对一关系，比如系统中存在 3 个 node，每个 node 都可以拥有 proposer/acceptor/learner 的 role，既可以自己提案 propose，也可以 accept proposal。

### 推理
为了获得满足上述 safety requirements 的 consensus algorithm, 论文开始从最简单的要求开始推理。

**推理1**： 为了达成共识，可以手动指定某个 acceptor 为 leader acceptor, 其他 acceptor 只需要被动获取 leader acceptor 的值就行，这样系统很快就可以达成 consensus。比如 leader acceptor 规定接收到的第一个 proposal value 为 chosen value。

**问题**： *假如 leader acceptor crash 了，系统就无法运行了，不符合分布式 fault-torelant 的要求*。

**推理2**： 为了解决上述的问题，提高系统 fault-torelant 能力，不需要 leader acceptor，any acceptors 都可以接受 any proposers' proposal。

由于网络的不可靠性，acceptor 不知道有多少 proposal 正在或者将要发给自己，所以要求 acceptor 不要等待，要直接 accept 第一个 received proposal。

我们称这个要求为 **P1**。
![alt text](/assets/images/post/replication-simple-paxos/image-1.png)

在 P1 要求下，我们假设每个 acceptor 只能 accept 一个 proposal。

**问题**： *因为无法控制 proposal value 的初始值，可能出现不同的 acceptor 会接受不同 value，导致系统中不存在某个被 majority acceptors accept 的值，比如：*
```
 acceptor0 accept proposal value v0;
 acceptor1 accept proposal value v1;
 acceptor2 accept proposal value v2;

 v0!=v1!=v2
```

**推理3**：为了解决上述问题，那就去掉*每个 acceptor 只能 accept 一个 proposal* 的约束，*允许每个 acceptor accept 多个 proposal*，从而保证系统最终有可能存在 chosen value。

此时为了区分不同的 proposal，需要给不同的 proprosal 指定唯一的 proposal number，使用递增数字来表达时序关系。

在允许*每个 acceptor accept 多个 proposal*的前提下，为了满足 safety requirement 2 => only one value is chosen，论文提出了一个要求以便满足 safety requirement 2。

要求系统中，一旦第 m 个 proposal => proposal-m value(v0) 被 chosen(也就是 majority acceptor has accepted v0), 之后所有被 chosen 的 proposal-n value(n>m) 都要等于 v0(是和等于在这里是一样的概念)，我们称这个要求为 **P2**。
![alt text](/assets/images/post/replication-simple-paxos/image-2.png)

这样就满足了不管 acceptor 如何规定 accept 多个 proposal 的方式，最终结果都是 only one value is chosen，即使整个系统多次达到了 chosen 状态，这些 chosen value 的值都是一样的。

> NOTE:  
> P2 只是一个对算法的要求，该要求中并没有规定 acceptor 要以什么规则 accept 多个 proposal，也没有规定 proposer 要以什么规则设置新的 proposal value。


**推理4**：记住我们的目标是要得到一个 consensus algorithm 在同一个值上达成共识。
- 我们从系统的不可靠性出发，要求这个最终算法要满足 P1。
- 为了满足算法的 safety requirement 2，需要在 P1 的基础上，继续满足 2 个条件：

```
1. acceptor 可以 accept 多个 proposal
2. 满足 P2
```

为了满足 P2，论文进一步提出了一个合理的要求 **P2a**，从而实现 P2。
![alt text](/assets/images/post/replication-simple-paxos/image-3.png)

P2a 表达的意思是： 一旦 proposal-m value v0 被 chosen, 那之后所有被 acceptor accept proposal-n value(n>m) 都要是 v0。

满足 P2a 就可以满足 P2, 因为 chosen v0 本质上是 majority acceptors accept v0, 现在规定在 n>m 时，any acceptors accept 的 any proposal-n value == v0, 自然满足 chosen value=v0。

可以说，P2a 是对 acceptor accept 行为的更强约束，因为要求的是 any acceptors, 不只是 majority acceptors。 

**问题**：*P2a 虽然可以满足 P2, 但是 P1 会导致 P2a 无法被实现，因为假如系统在已经存在 chosen value(v0) 的前提下，有个 new acceptor 和 new proposer 刚 failover 后上线，然后 new proposer 给 new acceptor 提交了一个 proposal value(v1)，根据 P1 的要求，new acceptor 要直接 accept v1，这样就违反了 P2a，因为 P2a 规定了 any acceptors 的行为*。

**推理5**： 为了解决 P1 和 P2a 的冲突，我们需要提出对 proposer 提交行为进行约束，不能允许 proposer 提交随意值，起码在系统已经存在 chosen value 时不能再提交随机值。所以论文进一步提出了 **P2b** 要求：
![alt text](/assets/images/post/replication-simple-paxos/image-4.png)

P2b 要求一旦 proposal-m value(v0) 被 chosen, 那 any proposers 提交的 proposal-n value(n>m) 都要等于 v0。

这个要求解决了 P1 和 P2a 的冲突，同时是个比 P2a 更强的约束，因为规定 any proposers 提交 v0, 自然 any acceptors 只可能 accept v0，没有其他可选项。

**推理6**： 为了实现 P2b 要求，一个直觉是： proposer 在决定 proposal value 之前是需要询问系统 majority acceptor，否则不可能知道系统已经有了 chosen value。

**但是要以什么样的方式询问呢？需要询问什么信息呢？**

接下来就是论文的**闪光点**了。

我们再复述一遍 P2b，当 proposal-m value v0 被 chosen 时，对于 n>m，any proposers 提出的 any proposal-n value == v0。

我们现在想要找到一种 Q 算法，使得最终的结论 any proposal-n value==v0 成立，从而满足整个推理链, **P2b => P2a => P2**。

为了找到这个算法 Q，需要绕一下，先假如已经存在某种算法 Q 使得 P2b 已经满足，成为一个定理。那就可以在某种算法 Q 下用数学归纳法对 P2b 进行证明。对应的数学归纳法如下：
```
initial:  proposal-m value v0 is chosen
last state： 对于 any proposal-(m, n-1]，都已经满足 proposal-(m, n-1] value==v0

求证： 在 Q 算法作用下，any proposal-n value==v0 成立。
```

为了找到 Q 算法，我们先分析已知条件下可以获得什么样的有用信息。
1. *proposal-m value v0 is chosen*
说明系统中存在某个 majority acceptors 集合 C，C 中 every acceptor accepts v0。 这是 chosen 的定义决定的。

2. *对于 any proposal-(m, n-1]，都已经满足 proposal-(m, n-1]==v0*
说明集合 C 中 every acceptor，至少 has accepted 其中一个 proposal-[m, n-1]，这个性质包含了上面的性质，起码最基本的 every acceptor in C has accepted proposal-m value v0。
另外一个性质就是，因为 any proposal-\(m, n-1\] value\==v0，所以 any acceptors accepted proposal-(m, n-1] value==v0, 因为没有其他可选项。

在上面的性质下，为了保证 any proposal-n value==v0 成立，论文提出了一个 Q 算法，也就是论文中的 **P2c invariant**：
![alt text](/assets/images/post/replication-simple-paxos/image-5.png)

翻译过来就是：在 proposer 决定 proposal-n value 之前，向某个 majority acceptors 集合 S 询问他们当前的 accepted value，并从中选择小于 n 的 highest proposal number 对应的 proposal value 作为自己的 proposal-n value，这样就能保证 proposal-n value==v0。

**为什么 P2c 就可以作出这种保证**？

1. 假如 highest proposal number 在范围 (m, n-1] 内，则可以确定对于对应的 proposal value==v0，因为这个范围的所有 proposal value 都是 v0，这是个已经条件。
2. 假如 highest proposal number 在范围 (0, m] 内，因为任何 majority acceptors 集合 S 都会和集合 C 至少有一个 overlap acceptors，所以 highest proposal number == m，accepted value=v0, 这个也是已知条件。

到这里可以看出只要满足 P2c 的要求，就可以证明数学归纳法，自然满足 P2b 要求。

**问题**： *要实现 P2c 还有个实际的问题需要解决，就是选择这个 highest proposal number 在分布式环境下是很难做到的，因为 P2c 是在一个理想的前提条件下(已经达到了 chosen 状态，已经经过了 proposal-m), 然后按照 P2c 的要求确定 proposal-n value 就行*。

**实际的场景不是这样的，因为每个 proposer/acceptor 都在独立得决定应该 propose/accept 什么样的 proposal。proposer 在获得 proposal number n 的时候，系统可能还没有开始 proposal-m 的提交，同时它也不可能知道 proposal-m value 一定会成为 chosen value**。

*所以说 P2c 中的 highest proposal number 指的是，在 proposer n 开始询问时，在 (0,n] 范围内 any acceptor **可能获取** 到的 highest proposal number，而不是在 (0,n] 范围内 any acceptor **已经获取** 到的 highest proposal number。*

**这是一种微妙的差别**。

举个简单的例子：
```
future chosen proposal number: m=5
future chosen value=v0
n=10

Current State(<proposal number, value>):
acceptor0: <1, v0>
acceptor1: <2, v1>
acceptor2: <3, v2>

proposer0: 
    current proposal number=5
    1. ask...
    2. response...
    3. select...
    4. construct... 


proposer1: 
    current proposal number=10
    1. ask majority acceptors: acceptor0, acceptor1
    2. get response: acceptor0 <1, v0>, acceptor1 <2, v1>
    3. select the response with highest proposal number: acceptor1 <2, v1>
    4. construct new proposal: <10, v1>  (BINGO!!!WRONG!!!VIOLATE P2c!!!)
```

在这个例子中，系统提交 proposal number==5 时会出现一个 chosen v0，但是因为 proposer0 和 proposer1 独立运行，在 proposer1 开始 proposal number=10 的 proposal 时 proposer0 也正在进行，而此时系统还没有任何 chosen value。

此时，即使 proposer0 使得系统存在 chosen value=v0 状态，proposer1 的行为也直接违反了 P2c invariant，导致推理链条上的 P2c => P2b => P2a => P2 全部不成立，则无法满足 consensus algorithm 的 safety requirement 2。

**推理7**： 上诉提到的问题的核心点在于，分布式环境下，proposer 在询问 acceptor 当前状态之后，是无法预测是否存在区间 (current highest proposal number of acceptor, my proposal number) 内的 proposal 会被对应的 acceptor accept，也就是无法预测未来发生的事情。比如：

```
proposer1: 
    current proposal number=10
    1. ask majority acceptors: acceptor0, acceptor1
    2. get response: acceptor0 <1, v0>, acceptor1 <2, v1>
    ################################################
        acceptor1 accepts a new proposal <5, v0>
    ################################################
    3. select the response with highest proposal number: acceptor1 <2, v1>
    4. construct new proposal: <10, v1>  (WRONG!!!)

```

于是，论文提出的解决方案是:  
![alt text](/assets/images/post/replication-simple-paxos/image-6.png)

翻译过来就是：一旦 acceptor 收到了 proposal-n 的查询请求，acceptor 就要作出 promise：拒绝 accept 所有小于 n 的 proposal，把 **可能** 发生的所有 accepted proposal 等价转变成 **已经** 发生的所有 accepted proposal。

从而保证查询请求拿到的是 acceptor 在这段时间内的 accepted highest proposal number proposal，这样从所有查询结果中获取的 highest proposal number 自然也是可能发生的 highest proposal number。

到这里整个算法的推导就结束了，同时在推导过程中通过解决所有问题，也为 proposer 和 acceptor 制定了相应的行为约束，从而保证推导链的成立。

## Paxos 算法流程

上面的推导流程涉及的行为约束大致为：
1. proposer 在正式提案之前，需要做 majority read 查询获取系统历史信息，然后再提交 proposal。
2. acceptor 在 receive 查询时，需要对 proposer 作出 promise，可以 accept 所有不在 promise 范围内的 proposal。

正式的算法流程如下：

![alt text](/assets/images/post/replication-simple-paxos/image-7.png)
![alt text](/assets/images/post/replication-simple-paxos/image-8.png)

这个算法过程分成 Prepare 阶段和 Accept 阶段：
1. Prepare 阶段:

```
Proposer: 获取一个 new proposal number n, 然后发送 prepare request (prepare-n) 到 majority acceptors。

Acceptor: receives prepare-n
    * if 已经 received proposal number 大于 n 的  prepare-t(t>n), 则丢弃或者回应告知无效 prepare，要求对应的 proposer 重新获取 new proposal number。
    * else 回应 accepted <highest proposal number, proposal value>，语义上保证不会再 accept 比 n 小的 proposal。
```

2. Accept 阶段

```
Proposer: 
    * if 超时获取不到 majority acceptors 的正确应答或者 received 到一个无效 prepare 应答，就重新开始 Prepare 阶段。
    * else 获取所有 prepare 应答中 highest proposal number 对应的 proposal value v, 提交 accept request <n, v>。

Acceptor： receives accept request <n, v>
    * if 已经 received proposal number 大于 n 的  prepare-t(t>n)，则丢弃或者回应告知无效 proposal，要求对应的 proposer 重新获取 new proposal number。
    * else accepted proposal <n, v>，回应成功。
```

## Paxos 算法终止

由于 Paxos 算法保证了系统中 only one value is chosen，所以只要通过 majority read 发现存在被 majority acceptors accepted 的值，则认为算法运行结束。

learner 有多种方式可以获取到最终的 chosen value:
1. 每个 learner 周期做 majority read 判断。
2. 每次 acceptor accepted 了某个 proposal，就把值发给所有 learner，每个 learner 自行判断即可。
3. 每次 acceptor accepted 了某个 proposal，就把值发给部分 learner，等 learner 确定 chosen value 存在后，再通知其他 learner。

## Paxos 算法 Liveness

上面在 Paxos 算法推导中只考虑了 safety requirement，没考虑 liveness，即算法需要能一直推进。

从 Paxos 的 Prepare 和 Accept 阶段可以推断出，可能存在 2 个 proposer 不断在 Prepare 使对方失效，从而再次获取更大的 proposal number 进行 prepare 的循环场景，最终算法无法推进。

对于这个问题，推荐的比较好的做法是：根据冲突的严重情况，采用 random backoff 延迟下一次 prepare 时间，保证其他 proposal 有充足的时间可以被 accept，从而快速达到 chosen 状态。

## 总结
Paxos 算法本身只能对一个值达成共识，比如3个节点中选主，它无法直接用于实际场景中的日志同步。可以认为它只是对日志中的一个写操作达成了共识，日志同步需要用到 multi-paxos 构建日志状态机，原始论文《[The Part-Time Parliament](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf)》有对这方面的描述, 具体工程方面的实践也很复杂。

[simple paxos demo](https://github.com/maxshuang/simple-paxos) 写的一个可以模拟多节点和网络丢包环境的 demo，不算特别完善，可以玩一下。