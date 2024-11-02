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
  height: "55vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Database Transaction
---

# Background

Transaction is an abstraction of concurrent read and write operations on data. The characteristics of these operations are:
1. Concurrent reads and writes
2. Can involve single-object or multi-objects

Before the abstraction of transactions, many boundary issues needed to be considered, such as:
1. If part of the operations on multi-objects fail, the successful part will also roll back, otherwise, data inconsistency will occur.
2. Concurrent writes read the old value simultaneously and then perform increment operations, resulting in lost updates.
3. Concurrent reads and writes lead to reading part of the old value and part of the new value.
4. The write operation returns success to the user, but system crash or power cutoff leads to data loss.

All these issues can be referred to as **data anomalies**.

Transactions generally appear with RDMS (Related Database Manager System), but as an operational abstraction, transactions can provide a simple and powerful operational abstraction as long as concurrent multi-object operations are involved.

## Basic Characteristics

As an operational abstraction, transactions provide ACID characteristics:
1. Atomicity  
Multiple operations within a single transaction either all succeed or all fail, and users cannot see intermediate states.

2. Consistency  
Data consistency means maintaining the semantic constraints of the data. This is more of an application-level constraint because the semantics of the data are given by the application. For example:  

``` 
User A ($100) transfers $10 to user B ($50). No matter how the operation is performed, from the application semantics, the total amount of user A and user B should be $150.  
Due to operational errors, user A transferred $10, and then the system failed over. After recovery, it became A ($90), B ($50). At this point, we say the system is in an inconsistent state.
```

At the same time, data consistency is also related to the system providing the data. For example, when the database handles concurrent reads and writes, it reads an inconsistent data state. Even if the data itself is consistent, users may perform certain operations based on the inconsistent state they read, leading to underlying data inconsistency.

Some databases provide mechanisms to constrain data semantics, such as foreign keys.
> NOTE:  
> The consistency here is different from data replication consistency.  
> The consistency here emphasizes data semantic requirements, such as keeping the sum of two numbers fixed.   
> Data replication consistency emphasizes recency requirements, that is, when the newest write is seen under multiple backups.

3. Isolation  
The visibility of data in concurrent read and write scenarios. If transactions are viewed as a collection of multiple operations tx=[op1, op2, ...], isolation describes the visibility of data between transactions, which is a difficult point to understand.

4. Durability  
Durability means that once a transaction is committed, the data will not be lost even if the system crashes, usually referring to writing to local non-volatile media such as HDD/SSD. In some distributed systems, there may be higher durability requirements, such as strong consistent backups.

## Isolation

In business development, using SQL to read and write databases is already a common operation. Many beginners first learn about databases from how to use SQL for CRUD operations. Even if they do not engage in backend development-related work, they may use SQL for a long time without understanding the semantics of Isolation Levels.

Databases provide transaction capabilities, and a single SQL statement is implicitly declared as a transaction. If multiple operations need to be executed within a transaction, the transaction needs to be explicitly declared to start, commit, and roll back, such as in MySQL:

```
start transaction;
commit; (or rollback;)
```

Transactions have the four characteristics mentioned above, such as atomicity, where all operations either succeed or fail, but this also leads to many *misconceptions*.

> NOTE:  
> Serializability is a property of concurrent transaction scheduling, indicating that the result of concurrent operations is consistent with single-threaded serial operations.  
> Serializable is an isolation level that provides serializability, and there are other isolation levels that can achieve serializability.

A common misconception is using SQL with the **naive view of serializability**, which means thinking that the database executes SQL in a **single-threaded serial** manner. For example:

```
initial: A=10

Transaction1 and Transaction2 execute simultaneously:
start transaction;
read: select A from table B;
check: if A > 9;
write: update A=A+10 in table B;
commit;
```

In concurrent operations, if you think the database executes SQL in a *single-threaded serial* manner, you would expect the final result to be A=30 because these two concurrent transactions are executed serially in the database, similar to the following execution timeline:

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

But in reality, if you use the default database isolation level, the result is likely to be A=20 in concurrent write scenarios, resulting in lost updates.

| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction;| start transaction; |
| t1   | select A from table B; (A=10)| |
| t2   |      |  select A from table B; (A=10)  |
| t3   | if A > 9; update A=A+10 in table B; (A=20)    |   |
| t4   |    | if A > 9; update A=A+10 in table B; (A=20)  |
| t5   | commit;   | commit; |

The reason for this cognitive bias is: 
**To improve the performance of concurrent read and write transactions, the database will by default not use or even provide serializability, which is the most consistent guarantee with linear thinking for concurrent read and write transactions.**

Not understanding Isolation Levels may trigger data anomalies at some point in the future. Let's follow the footsteps of data anomalies to see what constraints the database has already provided for us.

We use the [isolation levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html) provided by MySQL as an example. For other databases, you need to check the corresponding official website for the semantics of the corresponding isolation levels, as this is related to the implementation of the database.

### Dirty Read/Write And Read Committed

Considering only ACD from ACID (since Isolation is the topic of discussion here), we mostly only get guarantees for a single transaction (not entirely accurate) and do not describe the visibility rules when multiple transactions read and write the same data simultaneously.

A natural question arises:
*During the execution of Transaction A, can Transaction B read the changes made by Transaction A? If it can, what happens if Transaction A rolls back all its operations afterward?*

This type of data anomaly is called Dirty Read and Dirty Write, which means reading or writing uncommitted data.

#### Dirty Read

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


In this concurrent read-write example, we see that Transaction1 read the uncommitted value A=20 from Transaction2, which led Transaction1 to perform business logic based on non-existent data (A=20), possibly impacting further database operations.

#### Dirty Write

```
initial: A=10
isolation level: read uncommitted
```

| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction; | start transaction; |
| t1   | update A=11; |  |
| t2   |  | update A=12; |
| t3   |  | commit;  |
| t4   | rollback; |   |
| t5   |（A=10） |   |

In this concurrent write example, Transaction1 and Transaction2 update the value of A simultaneously. After Transaction2 commits, Transaction1 erroneously rolls back updates that do not belong to it, resulting in Dirty Write.

To handle these data anomalies, MySQL offers the Read Committed isolation level. Internally, it may use mechanisms such as exclusive locks on writes to block reads or multi-version reads (old version data) to ensure only committed data is visible.

### Read Skew/Non-repeatable Read/Phantom Read And Repeatable Read

Once the issue of reading uncommitted data is resolved, we achieve a guarantee for data operations where:
*Under read committed, the data seen is either before or after a commit.*

With this guarantee, a new issue arises:
*What if Transaction A reads a value before Transaction B starts, then reads it again after Transaction B updates it? Is this data anomaly? The underlying data is consistent, but the read data is a mix of new and old values.*

#### Read Skew/Non-repeatable Read

This data anomaly is known as Read Skew/Non-repeatable Read.

Read Skew refers to *Partial Read*, where some old and some new values are read. Non-repeatable Read is a type of read skew, where a transaction reads different values for the same data at different times.

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

Transaction1 read old data (A1=10) and new data (A2=15) without violating the rule of reading committed data. However, the data consistency requirement during the read is not met, even though the underlying data is consistent (A1=5, A2=15).

#### Phantom Read

Besides partial reads, a transaction may read different data sets at different times. For example:

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

You can see that Transaction1 reads different data within the same transaction, with the latter read retrieving more data.

These data inconsistencies are more likely to occur in long transactions, such as full data backups and analytical queries.

MySQL provides the Repeatable Read (RR) isolation level to address such data anomalies, which some databases refer to as Snapshot Isolation (SI). The implementation techniques are roughly as follows:
1. For concurrent reads and writes, the MVCC (Multi-Version Concurrent Control) technique is used. Each piece of data maintains multiple versions, and readers can only read data that has been committed before the read operation, not data that is being committed or will be committed in the future. The temporal relationship is expressed through monotonically increasing versions.
In simpler terms: reading historical data, so it won't be blocked by writes.
2. For concurrent writes, the 2PL (2 Phase Lock) method is used to serialize data access. Only when a transaction commits will it release all the locks it holds.

> NOTE:  
> MySQL's RR is different from the ANSI SQL RR defined based on locks. MySQL RR is more akin to SI (MVCC mechanism) in implementation, while ANSI SQL RR achieves RR by holding shared locks during reads until the transaction ends (prohibiting other writes).  
> Another difference is that the ANSI SQL RR implementation cannot solve the Phantom Read problem because simple row locks cannot prevent gap inserts. MySQL RR uses the MVCC mechanism, and since the commit ID of newly inserted data is larger, it won't be seen.

In the above example, since Transaction1 starts reading before Transaction2's transaction ends, Transaction1 cannot see all the changes made by Transaction2, thus ensuring the consistency of the read data.

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

You can see that Transaction1 sees the previous consistent state (A1=10; A2=10), while the underlying state is the new consistent state (A1=5; A2=15).

> NOTE:  
> Besides MVCC, MySQL's RR also uses gap locks. It might seem counterintuitive that the XID of newly inserted/deleted records should be greater than the XID of the read transaction, so MVCC can achieve consistent reads. Why do we still need gap locks?  
> Because MVCC solves the phantom read problem for snapshot reads but does not solve the phantom read problem for current reads. Current reads need to read the latest data version, so they require a read lock. Without gap locks, new inserted data could be accessed by the same read transaction, causing issues.

### Write Skew/Phantom And Serializable
Repeatable Read (RR) / Snapshot Isolation (SI) can greatly improve the concurrency of database transaction reads and writes and ensure that read transactions can always obtain a consistent state (although it may be old). Therefore, RR or SI is used as the isolation level in many business scenarios.

But does RR and SI achieve the serializability that most users expect? *Not yet*.

The key issue here is:
*RR and SI read historical consistent states, meaning that if users make updates based on an old state, and this update is related to the old state, it may break the semantic constraints of the data.*

This might sound abstract, so let's illustrate with a specific example. Remember the point of **making judgments based on historical states** during the analysis.

#### Update Lost
The furthest distance between RR and SI and serializability, in my opinion, is **Update Lost**. This is a very intuitive operation that often yields incorrect results, known as read-modify-update. For example:

```
initial: A=10
isolation level: rr or si
```

| timestamp | Transaction1 |  Transaction2 |
|------|-------|----|
| t0   | start transaction; | start transaction; |
| t1   | select A from table B; (A=10) |  |
| t2   |  | select A from table B; (A=10) |
| t3   | update A=11 in table B; (set A=A+1) |  |
| t4   | commit; |   |
| t5   |  | update A=11 in table B; (set A=A+1)   |
| t6   | | commit;  |

Here, under the rr or SI isolation level, the old value of A (A=10) is read. The business semantics are to increment the value. Under serializability, we expect A=12, but due to *making judgments based on historical states*, both Transaction1 and Transaction2 set A=11, resulting in an update lost.

To avoid update lost in this scenario, there are two approaches:
1. Add an exclusive lock to all data during the read to prevent other transactions from reading or writing, thus serializing different transactions. The downside is low concurrent read/write efficiency.
2. Use an optimistic approach, recording the version during each read and checking if the version of all read data has changed during commit. If it has changed, rollback or restart the transaction. The downside is that hot read/write data may cause frequent transaction restarts.

#### Write Skew
Another famous example of data anomalies caused by *making judgments based on historical states* is write skew. It is an anomaly that violates data semantics in a read-check-write operation mode, where the write causes the check to no longer hold. For example:

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

In this example, we can see that both Transaction1 and Transaction2 aim to set one of the values to 0 under the condition that A1+A2==2, ensuring that the final result is A1+A2==1. However, the actual result is A1+A2==0.
> NOTE:  
> It makes more sense if you imagine it as doctors on duty in a hospital. Every day, one doctor needs to be on duty. Both doctors originally on duty check the status and think the other will be on duty, so they both take leave. The key point is that both leaves are approved, resulting in no doctor on duty.

In other words, under the RR and SI isolation levels, making data changes based on historical states can lead to write skew, a data anomaly that violates data semantic constraints.

This write skew phenomenon does not only occur in two transactions but can also appear in cascading operations of multiple transactions, leading to hard-to-detect data anomalies.

The key to solving this problem is **making judgments based on historical states**. As long as the latest state is read each time and other reads/writes are prohibited, this problem can be partially solved (see the Phantoms section). However, this will reduce the concurrent read/write performance of the database.

Another solution is periodic recheck at the business level because the database does not provide this automated recheck semantic constraint feature (foreign keys can be considered a business-level semantic constraint).

This write skew violation of semantic constraints is a data anomaly that is *difficult to detect and troubleshoot* in actual development.

#### Phantom
Another special type of write skew is the Phantom. In RR, we discussed Phantom Read, which has been solved by RR and SI because new inserts have larger transaction IDs and thus won't be seen.

Here, the write skew phantom is a *more subtle* scenario. From the previous analysis, we know that the core issue of write skew is *making judgments based on historical states*. One solution is:
As long as the latest state is read each time and other reads/writes are prohibited, this problem can be partially solved.

Based on this principle, let's modify the above example:
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

We can see that Transaction1 adds an exclusive lock when reading A1 and A2, causing all other reads/writes to wait until Transaction1 commits, thus serializing the transaction execution and avoiding write skew.

Here, a small problem arises:
*We avoid write skew by locking existing records. How do we prevent user operations for gaps between two records since there is no lock serialization?*

Let's explain this problem with the following example:
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
| t4   | if(sum(A)==2) { /* perform some business logic based on the sum of A */ };  |  |
| t5   | commit; |  |

In this example, we can see that although Transaction1 adds an exclusive lock to read the latest data and prevent other transactions from reading/writing it, it still cannot prevent Transaction2 from inserting new data, causing if(sum(A)==2) to become a *historical state*, potentially **violating data semantic constraints**.

To handle such data anomalies, one method is to add a *gap lock* in addition to the record lock, such as MySQL's next-key lock (record lock + gap lock). 

A gap lock is a lock on the gap between index records, which prevents other transactions from inserting new rows into the gap. For example, if you have records with values 1 and 5, a gap lock on the range (1, 5) would prevent any new records with values between 1 and 5 from being inserted. This ensures that no new records can be added that would violate the constraints checked by the current transaction.

Example:

The data anomalies discussed above start from the simplest read uncommitted and gradually constrain data visibility through technical means to provide stronger guarantees. At the RR and SI isolation levels, it is evident that solving Update Lost and Write Skew requires locking read data or tracking version changes, which are costly operations.

Solving all data anomaly problems means the database provides a serializability isolation level. Currently, the algorithm that achieves Serializability with good performance is *[Serializable Snapshot Isolation](https://dl.acm.org/doi/10.1145/1620585.1620587)*, used by PgSQL to implement Serializability.

## Summary
Different isolation levels are a trade-off between data consistency and performance.

This article first mentions the common perception of transactions in business—serialized transaction execution (serializability). It then analyzes potential data anomalies starting from weak isolation levels:
1. Read Uncommitted (RU) => Dirty Read, Dirty Write
2. Read Committed (RC) => Read Skew, Non-repeatable Read, Phantom Read
3. Repeatable Read (RR)/Snapshot Isolation (SI) => Write Skew, Phantom
4. Serializable Snapshot Isolation (SSI)/Serializable

The article also touches on concurrency control techniques related to MySQL's implementation of isolation levels, namely MVCC + 2PL, and other concurrency control methods such as:
1. Timestamp Ordering  
   This technique assigns timestamps to transactions to ensure a consistent order of execution. [Read more](https://en.wikipedia.org/wiki/Serializability#Timestamp-based_protocols)

2. Commitment Ordering  
   This method ensures that transactions are committed in a specific order to maintain consistency. [Read more](https://en.wikipedia.org/wiki/Commitment_ordering)

3. Serialization Graph Checking  
   This technique involves constructing a graph of transactions to detect cycles that indicate conflicts. [Read more](https://en.wikipedia.org/wiki/Serializability#Serialization_graph)

Readers interested can look up additional database resources. 