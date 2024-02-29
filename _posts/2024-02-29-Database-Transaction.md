---
layout: post
title: Transaction(Basic) 
subtitle: picture from https://www.pexels.com/search/wild%20animals/
author: maxshuang
categories: Database
banner:
  image: /assets/images/post/database-transaction/hippo.jpg
  opacity: 0.618
  background: "#000"
  height: "70vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Database Transaction
---

Transaction 是数据并发读写操作的一种抽象。这种读写操作的特点是：
1. 读写并发
2. 可以涉及 single-object 或者 multi-objects

在没有 transaction 抽象之前，就需要考虑很多边界问题，比如：
1. 操作 multi-objects 部分失败了，则成功的部分也会回滚，否则会出现数据不一致。
2. 并发写同时读取到旧值后，然后执行自增操作，导致更新丢失。
3. 并发读写导致读到了部分旧值，部分新值。
4. 写操作给用户返回成功了，系统 crash 或者 power cutoff 导致数据丢失。

等等这些问题都可以称为数据异常 data anomaly。

事务一般和 RDMS(Related Database Manager System) 一起出现，但是作为一种操作抽象，只要涉及到并发的 multi-objects 操作，事务都能提供一个简洁且强大的操作抽象。

# 基本特性
事务作为一种操作抽象，提供了 ACID 特性：
1. Atomicity
单个事务中的多个操作，要么全部成功，要么全部失败，用户无法看到中间状态。
2. Consistency
数据一致性，表示对数据的语义约束保持一致。这个更多是应用级别的约束，因为数据的语义是应用赋予的。
举个例子： 用户 A（\$100）给用户 B（\$50）转账 \$10，无论如何操作，从应用语义上看要求用户 A 和用户 B 的总额是 \$150。由于操作上的失误，用户A 转出 \$10 后系统 failover 了，恢复后变成了 A（\$90）, B (\$50)，此时我们说系统处于不一致状态。

同时，数据的一致性也和提供数据的系统有关，比如数据库在处理并发读写时读到了不一致的数据状态，即使数据本身是一致的，用户也可能基于读取到的不一致状态做出某些操作，导致底层数据不一致。

一些数据库本身提供了某些机制约束数据语义，比如 foreign key。
> NOTE:  
> 这里的 consistency 和 data replication consistency 是两个概念。
这里的 consistency 强调的是 data semantic requirement，比如两个数的总和保持固定值。 
而 data replication consistency 强调的是 recency requirement, 也就是多备份下 newest write 被什么时候看到。
3. Isolation
并发读写场景下数据的可见性问题。如果把事务在形式上看成多个操作的集合 tx=[op1, op2, ...], 那 isolation 描述的就是事务和事务之间数据可见性的问题，也是需要理解的难点。
4. Durability
持久性，事务提交后，即使系统崩溃也不会丢失数据。一般我们指写入本地 non-volatile 介质，比如 HDD/SSD。在一些分布式系统中可能有更高的持久性要求，比如强一致备份。

# Isolation
业务开发中，利用 SQL 读写数据库已经是个普遍的操作。许多初学者第一次学习数据库就是从如何使用 SQL 进行 CURD 开始的，甚至如果不是做后端开发相关工作，未来很长一段时间都可能在不了解 Isolation Level 的语义下使用 SQL。

数据库提供 transaction 能力，单条 SQL 其实已经被 implicitly declare 成了事务，如果需要在事务中执行多条操作，就需要显式声明事务的开始、提交和回滚，比如 MySQL 中的：
```
start transaction;
commit; (or rollback;)
```
事务拥有上面提及的 4 个特点，比如 atomicity 的要么全部成功，要么全部失败，但也导致了出现很多*迷思*。

>NOTE
>serializability 是系统提供的保证数据一致性的一种操作性质，表示并发操作的结果和单线程串行操作一致。
>serializable 是提供 serializability 的一个隔离级别, 也有其他一些隔离级别可以实现 serializability。

一种普遍的迷思是怀着*朴素的可串行化 serializability* 的观点使用 SQL，说得更简单点就是认为数据库是以*单线程串行化*的方式在执行 SQL。举个例子：
```
initial: A=10

Transaction1 和 Transaction2 同时执行：
start transaction;
read： select A from table B;
check: if A > 9;
write: update A=A+10 in table B;
commit;
```
在并发操作下，如果认为数据库是以*单线程串行化*的方式在执行 SQL，那么就会认为最后的结果肯定是 A=30, 因为这两个并发的 transaction 在数据库中是串行有序执行的，类似如下执行时间序列：
| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction;|  |
| t1   | select A from table B; (A=10)| |
| t2   | if A > 9; update A=A+10 in table B; (A=20) |   |
| t3   | commit;     |   |
| t4   | |start transaction;|
| t5   | |select A from table B; (A=20)|
| t6   |    | if A > 9; update A=A+10 in table B; (A=30)  |
| t7   |    | commit; |

但是实际上，如果你使用默认的数据库隔离级别，在并发写的场景下大概率结果是 A=20，出现了更新丢失问题。

| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction;| start transaction; |
| t1   | select A from table B; (A=10)| |
| t2   |      |  select A from table B; (A=10)  |
| t3   | if A > 9; update A=A+10 in table B; (A=20)    |   |
| t4   |    | if A > 9; update A=A+10 in table B; (A=20)  |
| t5   | commit;   | commit; |

这里认知偏差的原因在于： 
*数据库为了提高事务并发读写性能，会默认不使用甚至不提供 serializability 这种最符合线性思维的事务并发读写保证。*

不理解 Isolation Level 就可能会在未来的某个时间节点触发数据异常，让我们跟随数据异常的脚步看下数据库已经为我们提供了什么样的约束。

我们使用 MySQL 提供的[隔离级别](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)作为例子，其他数据库需要去对应的官网中查看对应隔离级别的语义，因为这个和数据库的实现有关。

## Dirty Read/Write And Read Committed
只考虑 ACID 中的 ACD(因为 Isolation 是我们现在要讨论的)，我们基本只得到了单个事务的保证(不太准确)，没有说明多个事务同时读写相同数据时数据可见性是怎么规定的。

一个自然而然的问题是：
*事务 A 执行过程中，事务 B 是否可以读取事务 A 的变更？ 假如可以读取，但是后面事务 A 回滚了所有操作怎么办？*

这个数据异常就是 Dirty Read 和 Dirty Write，简单说就是读写了未提交的数据。
### Dirty Read
```
initial: A=10
isolation level: read uncommitted
```

| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction; | start transaction; |
| t1   | | update A=A+10 in table B; (A=20)|
| t2   | select A from table B; (A=20)|    |
| t3   |     |  rollback; |
| t4   | if(A>15) update A1;  |  |
| t5   | commit;  | (A=10) |


在这个并发读写的例子中，我们看到 Transaction1 读取了 Transaction2 还没有提交的 A=20 值，导致了 Transaction1 基于一个不存在的数据 A=20 进行了某些业务操作，甚至是后续的数据库读写操作。

### Dirty Write
```
initial: A=10
isolation level: read uncommitted
```

| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction; | start transaction; |
| t1   | update A=11; |  |
| t2   |  | udate A=12; |
| t3   |  | commit;  |
| t4   | rollback; |   |
| t5   |（A=10） |   |

在这个并发写的例子中，我们看到 Transaction1 和 Transaction2 并发更新 A 的值，在 Transaction2 提交之后，Transaction1 错误回滚了不属于自己的更新，所以叫 Dirty Write。

对于 Dirty Read 和 Dirty Write 这种数据异常，MySQL 提供 Read Committed 这种 isolation level 来解决这个问题，数据库内部可能是采用对 write 加排他锁阻塞 read或者多版本读取 old version data 方式保证只能看到 committed 的数据。

## Read Skew/Non-repeatable Read/Phantom Read And Repeatable Read
解决了读取 uncommitted 数据的数据异常之后，我们获取了一个数据操作 guarantee => 在 read committed 下看到的数据要么是数据提交前的，要么是提交后的。

在这个保证下，会出现的一个问题是：
*假如事务 A 在事务 B 开始前读取了一次 A 值，在事务 B 结束后又读取了一个 A 值，而事务 B 更新了 A 值，那前后两次读取的值就不一样了。这种算是数据异常吗？底层数据是一致的，但是读到的数据却是部分新部分旧的。*

### Read Skew/Non-repeatable Read
这就是我们要说的 Read Skew/Non-repeatable Read 的数据异常。 

Read Skew 简单说就是 *Partial Read*, 也就是读取到了部分旧值，读取到了部分新值。
Non-repeatable Read 也可以认为是一种 read skew，它是指在一个事务中对同一份数据，前后两次读取的值不同，前面读取到了旧值，后面读取到了新值。
```
initial: A1=10, A2=10;
isolation level: read committed
```

| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction; | start transaction;|
| t1   | select A1 from table B; (A1=10) | |
| t2   |  |  update A1=A1-5, A2=A2+5 in table B; (A1=5, A2=15)  |
| t3   |  | commit; |
| t4   | select A2 from table B; (A2=15) | |
| t5   | commit; (A1=10;A2=15) | (A1=5;A2=15) |

从 Transaction1 获取的结果上看，它读取了部分旧数据 A1=10, 读取了部分新数据 A2=15, 但是并没有违反读取 committed data 的规则，同时存储的数据状态也是一致的 A1=5, A2=15。

这个数据异常会比较微妙，从数据的语义约束上看，我们希望数据是从一个一致性状态过渡到另一个一致性状态，比如转账场景下从转账前的一致性状态过渡到转账后的一致性状态。在这个场景下，数据变更的确满足一致性要求，但是读取操作看到的数据不满足一致性要求。

### Phantom Read
同一个事务中除了 partial read 之外，还可能出现前后读取到不同的数据。比如：
```
initial: Column A has two rows(A1=10, A2=10)
isolation level: read committed
```

| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction; | start transaction;|
| t1   | select A from table B where A=10; (A1=10;A2=10) | |
| t2   |  |  insert A3=10 in table B;  |
| t3   |  | commit; |
| t4   | select A from table B where A=10;; (A1=10;A2=10;A3=10) | |
| t5   | commit;|  |

可以看到 Transaction1 在同一个事务中读到的数据是不一样，后面一次 read 读取到了更多数据。

这些数据不一致在一些 long transaction 中出现的概率会比较大，比如数据全量备份和分析型查询。

MySQL 提供 Repeatable Read(RR) 的 isolation level 解决这种数据异常，有些数据库称为 Snapshot Isolation(SI)。实现技术上大致为：
1. 对于并发读写，使用 MVCC(Multi-Version Concurrent Control) 技术，每份数据维护多个版本，读者只能读取到读取操作之前已经提交的数据，无法读取正在提交或者未来提交的数据，通过单调递增的 version 表达时序关系。
再通俗理解就是： 读的是历史数据，所以不会被写阻塞。
2. 对于并发写写，采用 2PL(2 Phase Lock)的形式串行化数据访问，只有事务提交才会释放自己持有的所有锁。

>NOTE
>MySQL 的 RR 和 ANSI SQL 基于锁定义的 RR 是不同的概念，它在实现上更加靠近 SI 的 MVCC 机制，所而 ANSI SQL 中 RR 是通过在 read 时持有 shared lock 直到事务结束实现 RR。
>
>另外一个不同的地方在与，ANSI SQL RR 的实现方式决定了无法解决 Phantom Read 的问题，而 MySQL RR 可以解决，因为 new insert data 的 commit id 更大，所以是不会被看到的。

对于上面的例子，由于 Transaction1 开始读时 Transaction2 的事务还没有结束，所以 Transaction1 看不到 Transaction2 做的所有变更，从而保证了读到的数据满足一致性要求。
```
initial: A1=10, A2=10;
isolation level: repeatable read/snapshot isolation
```

| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction; | start transaction;|
| t1   | select A1 from table B; (A1=10) | |
| t2   |  |  update A1=A1-5, A2=A2+5 in table B;  |
| t3   |  | commit; |
| t4   | select A2 from table B; (A2=10) | |
| t5   | commit; (A1=10;A2=10) | (A1=5;A2=15) |

可以看到 Transaction1 看到的是上一个一致状态(A1=10;A2=10)，底层是新的一致性状态(A1=5;A2=15)。

## Write Skew/Phantom And Serializable
Repeatable Read（RR)/Snapshot Isolation(SI) 可以极大得提高数据库的事务读写并发，并且可以保证读事务可以一直获取到一个一致状态(虽然可能是旧的)，所以在非常多的业务场景下都会使用 RR 或者 SI 作为隔离级别。

但是 RR 和 SI 就可以实现一般用户心目中的 serializability 了吗？ *还不是*。

这里出现问题的关键就在于：
*RR 和 SI 读取到的都是历史一致状态，也就是说用户如果基于一个旧的状态做一些更新，同时这个更新是和旧状态相关的，那最后可能导致数据语义上的约束被破坏。*

表述得有点抽象，我们举个具体的例子说明一下，分析的过程中要记住*基于历史状态做判断* 这个出问题的点。

### Update Lost
RR 和 SI 距离 serializability 最远的距离我认为是 *Update Lost*，这是一种非常符合直觉但是却总是得到错误结果的操作，也就是 read-modify-update。举个例子：
```
initial: A=10
isolation level: rr or si
```

| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction; | start transaction; |
| t1   | select A from table B;(A=10) |  |
| t2   |  | select A from table B;(A=10) |
| t3   | update A=11 in table B;(set A=A+1) |  |
| t4   |commit; |   |
| t5   |  | update A=11 in table B;(set A=A+1)   |
| t6   | | commit;  |

这里在 rr or SI 隔离级别下读取了 A 的旧值（A=10），业务语义上是对值进行自增，serializability 下期望 A=12，但是由于*基于历史状态做判断*，导致了 Transaction1 和 Transaction2 都设置了 A=11, 出现了更新丢失。

这种场景下，想要规避 update lost，可以有 2 中做法：
1. 在 read 时对所有的数据加上 exclusive lock，避免被其他事务读写，从而串行化不同事务。缺点是并发读写效率低。
2. 乐观处理机制，在每次 read 时记录下 version，在 commit 时检查所有 read 过的数据 version 是否变更过，变更过就回滚或者重启事务。缺点是热点读写数据可能造成频繁事务重启。 

### Write Skew
*基于历史状态做判断* 导致数据异常的另外一个有名例子是 write skew，它是一种 read-check-write 操作模式下，write 导致 check 不再成立的一种违反数据语义的异常。举个例子：
```
initial:  A1=1, A2=1
isolation level: rr or si
```
| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction; | start transaction;|
| t1   | select A1,A2 from table B; (A1=1;A2=1) | |
| t2   |  |  select A1,A2 from table B; (A1=1;A2=1)  |
| t3   | if(A1+A2==2) update A1=0 in table B; |  |
| t4   |  | if(A1+A2==2) update A2=0 in table B; |
| t5   | commit; (A1=0;A2=1) |  |
| t6   | |commit; (A1=0;A2=0) |

在这个例子中，我们可以看到，Transaction1 和 Transaction2 都希望在 A1+A2\==2 的前提下，将其中一个值设置成0, 从而保证最后 A1+A2\==1，但是实际的运行结果是 A1+A2\==0。
>NOTE
>如果想象成医院值班医生就会比较有意义。每天需要一名值班医生，结果原本值班的两个医生查看状态后都认为对方会值班，都请假了，关键是请假都通过了，导致最后没有医生值班。

也就是说在 RR 和 SI 隔离级别下，基于历史状态下做一些数据变更，如果最后影响了对应的历史状态的话，就会出现这种 write skew 的*违反数据语义约束*的数据异常。

这种 write skew 的现象不只是出现在 2 个事务中，也可能出现在多个事务的*级联操作*中，导致难以排查的数据异常。

解决这个问题的关键还是*基于历史状态做判断*，只要每次读取的时候都读取*最新的状态*，并禁止其他读写，就可以部分解决这个问题(见 Phantoms 小节)，但是这样会降低数据库读写并发的性能。

另一种解决方案是： 业务级别的周期性 recheck，因为数据库不提供这种自动化 recheck 语义约束的 feature。(foreign key 可以认为是一种业务级别的语义约束)。

这种 write skew 的违反语义约束是实际开发过程中*难以察觉，难以排查*的数据异常。

### Phantom
另一类特殊的 write skew 是幻象 Phantom，在 RR 中我们讨论过 Phantom Read，那个问题已经被 RR 和 SI 解决了，因为 new insert 有更大的 transaction id，所以不会被看到。

这里的 write skew phantom 是一个*更加微妙*的场景。在前面的分析中，我们知道 write skew 的核心问题是 *基于历史状态做判断*， 其中一个解决方案是：
只要每次读取的时候都读取*最新的状态*，并禁止其他读写，就可以部分解决这个问题。

基于这个原则，我们把上面的例子修改下：
```
initial:  A1=1, A2=1
isolation level: rr or si
```
| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0  | start transaction; | start transaction;|
| t1  | select A1,A2 from table B for update;(lock) (A1=1;A2=1) | |
| t2  |  |  select A1,A2 from table B; (wait)  |
| t3  | if(A1+A2==2) update A1=0 in table B;(lock) |  |
| t4  | commit; (A1=0;A2=1) |  |
| t5  | | select A1,A2 from table B for update; (continue)(A1=0;A2=1) |
| t6  | |if(A1+A2==2) update A2=0 in table B;(do nothing)|
| t7  | |commit; (A1=0;A2=1) |

可以看到 Transaction1 在 read A1,A2 的时候加了一个排他锁，导致除了本事务之外其他所有的 read/write 都要等到 Transaction1 事务提交后才能继续运行，从而串行化了事务执行，规避了 write skew。

这里有个小问题出现了：
*我们是通过对已存在的记录加锁规避的 write skew，那对于两个记录之间的 gap 而言，由于没有锁串行化，那怎么防止用户操作呢？*

我们用下面的例子解释这个问题：
```
initial:  Column A has two rows(A1=1,A2=1)
isolation level: rr or si
```
| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction; | start transaction;|
| t1   | select A from table B where A>=1 and A<=3 for update; (A1=1;A2=1) | |
| t2   |  |  insert A3=2 in table B;  |
| t3   | |commit; (A1=1;A2=1;A3=2;) |
| t4   | if(sum(A)==2) do something;  |  |
| t5   | commit; |  |

在这个例子中，我们可以看到，虽然 Transaction1 为了读取到最新的数据并且防止其他事务读写该数据，加了排他锁，但是还是没有能够防止 Transaction2 insert 新的数据，导致 if(sum(A)==2) 也变成了*历史状态*，就可能*违反数据语义约束*。

对于这类数据异常，一种方法是除了 record lock 之后，还要加上 *gap lock*，比如 MySQL 的 next-key lock(record lock+gap lock)。

我们在上面所讲的数据异常，是从最简单的 read uncommitted 开始，一步一步通过技术约束数据可见性，从而提供更强大的 guarantee。到了 RR 和 SI 隔离级别时，明显可以发现想要解决 Update Lost 和 Write Skew 就需要对读数据进行加锁或者追踪 version 变更，这些都是开销很大的操作。

解决了所有的数据异常问题，就可以认为数据库提供了 serializability 的隔离级别。目前看能实现 Serializability 且性能较好的是 *[Serializable Snapshot Isolation](https://dl.acm.org/doi/10.1145/1620585.1620587)*，这是 PgSQL 用来实现 Serializability 的算法。

# 总结
不同隔离级别都是对数据一致性和性能之间的 tradeoff。

本文首先提及业务对事务的迷思---串行化的事务执行(serializability)，然后从 weak isolation level 开始逐步分析可能出现的 data anomaly:
1. Read Uncommitted(RU) => Dirty Read, Dirty Write
2. Read Committed(RC) => Read Skew, Non-repeatable Read, Phantom Read
3. Repeatable Read(RR)/SnapShot Isolation(SI) => Write Skew, Phantom
4. Serializable Snapshot Isolation(SSI)/Serializable

本文还提及了一些和 MySQL 相关的实现 isolation level 的并发控制技术，也就是 MVCC + 2PL，还有其他一些不同的并发控制技术，比如：
1. Timestamp Order
2. Commitment Order
3. Serialization Graph checking

读者有兴趣可以执行查找相关的专业数据库资料。 