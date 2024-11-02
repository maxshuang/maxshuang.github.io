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

## Background

This blog post mainly discusses the Simple-Paxos algorithm, which belongs to the field of distributed system multi-replica consistency. Here, consistency refers to the recency requirement, i.e., when the newest write in a multi-replica scenario will be seen.

The need for multi-replica in distributed systems arises from the demand for fault-tolerant systems. In the era of single data copies, the requirement for durability was that data could be written to non-volatile media (HDD/SSD). Later, it was realized that in the event of disk failure, fire, earthquake, or other unavoidable natural disasters, **data could still be unrecoverable**.

Thus, the requirement for data durability naturally evolved to whether the same data could be kept in multiple replicas, replicated to different data centers, different physical regions, or even different countries. This way, even if some replicas are lost, the system can still be recovered using other replicas.

Multi-replica naturally introduces the issue of replica consistency, which is a tradeoff between data consistency and efficiency. **The weaker the replica consistency, the higher the write efficiency**, as there is no need to confirm that other replicas have also applied the write operation.

Generally, we categorize the consistency of a system's replicas as:
1. Strong consistency: Reading and writing all replicas is the same as reading and writing a single data copy, and the newest write will **be seen immediately**.
2. Weak consistency: The newest write will be seen **within a specified time limit**, e.g., the system guarantees that "the newest write will be seen by all users within 5 seconds."
3. Eventual consistency: The newest write will **eventually be seen**, but there is no specified time limit.

Strong consistency does not mean that every write must wait for all replicas to confirm, as this would make the system unable to write if even one replica is inaccessible. Modern strong consistency typically uses **majority read/write**, where the set of nodes involved in majority read overlaps with the set of nodes involved in majority write, ensuring fault tolerance while still reading the newest write.

## Paxos consensus algorithm

The goal of the Paxos consensus algorithm is to allow multiple nodes to reach consensus (majority of nodes agree on the same value) on the same value without a central node or manually designated central node. For example, at least 2 out of 3 people agree on A as the leader. Paxos ensures that the system can still reach consensus on the same value even if fewer than half of the nodes fail.

> NOTE:  
> *Pay special attention to the phrase "same value" in the goal. Paxos solves problems like "who is the leader," but it cannot be directly used for scenarios like log synchronization.*

The scenarios where the Paxos consensus algorithm is applicable are:
1. Non-Byzantine systems, i.e., systems where there is no malicious deception, corrupted messages can be detected, discarded, or recovered.
2. Distributed environments where communication between nodes relies on unreliable networks, which may result in message loss, duplication, or out-of-order delivery.

Paxos is my favorite algorithm. Given the freedom to set initial values on multiple nodes, it is almost unbelievable that consensus on the same value can be reached through communication and interaction. However, the reasoning process of Paxos is so logical and reasonable.

It can be said that **Paxos once again achieves reliability on top of unreliable mechanisms**.

Let's follow the steps of the paper [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) to understand Paxos reasoning and the algorithm.

### Paxos Discussion
The reason for introducing Paxos reasoning before the algorithm, apart from aligning with the original paper, is to showcase the logical thinking ability demonstrated by this algorithm, which I find most attractive.

If you have thought about the multi-node consensus problem before learning Paxos, you can appreciate the difficulty of this problem. Since multiple nodes can freely set initial values during initialization, deciding which node's value to adopt is a challenge.

In practical production scenarios, this translates to how to ensure that all nodes in a multi-master system have a consistent view of data when multiple masters simultaneously operate on the same data. For example, how to ensure that all nodes return consistent results when querying the same data.

Initially, systems would avoid this problem by adopting a single-master architecture and synchronizing or asynchronously replicating to slave nodes. Even now, cross-city multi-master architectures are used for asynchronous disaster recovery, providing eventual consistency and requiring logical isolation of data, e.g., data in Datacenter A can only be read and written by residents of City A.

Returning to Paxos reasoning itself, in my view, the core point of the entire reasoning process is **the construction of a logical clock among multiple nodes**.

This core idea is closely related to Lamport's other paper, [Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf), which discusses how to construct a global order without a global clock in a multi-node system. This global order, also known as a global logical clock, is crucial in distributed transactions because, without a global clock, it is impossible to determine the order of transactions, potentially leading to data anomalies.

*A core point of the global logical clock is constructing a global order through local causal relationships (not an entirely accurate term, still under consideration)*. For example:
```
node1 and node2 are two different nodes with different local clocks in the system

initial:
node1(clock): 1 2 3 4 5 6 7 8 9 10
node2(clock): 1 5 10 15 20 25 30 35

operation:
node1: 1 2 3 4 5(r1) 6 7 8(w2) 9 10
node2: 1 5 10(w1) 15(r3) 20 25 30 35
In this case: 
1. node1 has two operations: r1 and w2. r1 happens at clock 5, w2 happens at clock 8.
2. node2 has two operations: w1 and r3. w1 happens at clock 10, r3 happens at clock 15.
we have no idea about the global sequence of r1, w2, w1, r3 because their clocks are not synchronized.

operation:
node1: 1 2 3 4(r1') => 11(r1) 12 13(w2') => 31(w2) 31 32
            /                   /
            /                   /
node2: 1 5 10(w1) 15(r3) 20 25 30 35

In this case: For every operation initiated by node1, it first queries node2 to get the latest clock from node2, setting the clock for node1's read/write operations. This way, we obtain the following reliable cross-node logical order:
w1(10) => r1(11)
w1(10) => r3(15) => w2(31)
The reason is simple. For example, for w2, before the operation, it sees that node2's clock is already 30, and node2's clock is always monotonically increasing. Therefore, before w2 occurs, w1(10) and r3(15) must have already occurred. By comparing logical clocks 10<15<31, we can directly determine the order of cross-node events.
```

Paxos reasoning also implicitly includes this idea of constructing a global clock using local causal relationships.

### Paxos Reasoning

#### Algorithm Properties
First, Paxos is a consensus algorithm, which means it allows multiple nodes in a system to reach consensus on a single value. When the majority of nodes in the system accept a value \(v0\), we say that \(v0\) is chosen by the system.

> NOTE:  
> accepted: Only nodes can accept values, and they do not have a global view to know if a value has been chosen.  
> chosen: The act of being chosen does not have a corresponding entity executing it; it is a global perspective. When the majority of nodes accept the same value \(v0\), this value \(v0\) automatically becomes the chosen value.

For the correctness of the algorithm, we need to define the safety requirements:
![alt text](/assets/images/post/replication-simple-paxos/image.png)

That is:
1. Only proposed values (proposals) can be chosen by the system.  
In simple terms, the chosen value can only be selected from the proposed values and cannot be fabricated from other sources or out of thin air.  
2. There is only one value that will be chosen.  
The system cannot have a majority of nodes accepting proposal value \(v1\) and then another majority of nodes accepting proposal value \(v2\) (where \(v2 \neq v1\)).
3. Only after a value \(v0\) becomes the chosen value can observers consider \(v0\) to be chosen.  
Observers cannot mistakenly believe that a value has been chosen before the system reaches consensus.

#### Role Definitions
The system has three roles: proposer, acceptor, and learner.
* proposer: Responsible for proposing proposals, which include a proposal value that the proposer *hopes* will be chosen by the system.
* acceptor: Responsible for accepting proposals, similar to committee members.
* learner: Observers who do not affect the algorithm process but continuously learn the system's chosen value.

Nodes and roles are two different concepts and do not have a one-to-one relationship. For example, in a system with three nodes, each node can have the roles of proposer, acceptor, and learner, meaning they can propose, accept proposals, and learn the chosen value.

#### Reasoning
To obtain a consensus algorithm that meets the above safety requirements, the paper starts reasoning from the simplest requirements.

**Reasoning 1**: To reach consensus, a leader acceptor can be manually designated, and other acceptors only need to passively accept the leader acceptor's value. This way, the system can quickly reach consensus. For example, the leader acceptor can specify that the first received proposal value is the chosen value.

**Problem**: *If the leader acceptor crashes, the system cannot operate, which does not meet the distributed fault-tolerance requirements*.

**Reasoning 2**: To solve the above problem and improve the system's fault tolerance, there is no need for a leader acceptor. Any acceptor can accept any proposer's proposal.

Due to network unreliability, acceptors do not know how many proposals are being or will be sent to them, so they are required not to wait and to directly accept the first received proposal.

We call this requirement **P1**.
![alt text](/assets/images/post/replication-simple-paxos/image-1.png)

Under the P1 requirement, we assume that each acceptor can only accept one proposal.

**Problem**: *Since the initial value of the proposal cannot be controlled, different acceptors may accept different values, resulting in no value being accepted by the majority of acceptors in the system, for example:*
```
 acceptor0 accepts proposal value v0;
 acceptor1 accepts proposal value v1;
 acceptor2 accepts proposal value v2;

 v0!=v1!=v2
```

**Reasoning 3**: To solve the above problem, we remove the constraint that *each acceptor can only accept one proposal* and *allow each acceptor to accept multiple proposals*, ensuring that the system may eventually have a chosen value.

At this point, to distinguish between different proposals, each proposal needs to be assigned a unique proposal number, using increasing numbers to express the sequence.

Under the premise of allowing *each acceptor to accept multiple proposals*, to satisfy safety requirement 2 => only one value is chosen, the paper proposes a requirement to meet safety requirement 2.

The requirement is that once proposal-m value(v0) is chosen (i.e., the majority of acceptors have accepted v0), all subsequent chosen proposal-n values (n>m) must be equal to v0 (the concept of "equal to" here is the same as "is"). We call this requirement **P2**.
![alt text](/assets/images/post/replication-simple-paxos/image-2.png)

This ensures that regardless of how acceptors are allowed to accept multiple proposals, the final result is that only one value is chosen. Even if the system reaches the chosen state multiple times, the values of these chosen states are the same.

> NOTE:  
> P2 is just a requirement for the algorithm. It does not specify how acceptors should accept multiple proposals or how proposers should set new proposal values.

**Reasoning 4**: Remember, our goal is to obtain a consensus algorithm that reaches consensus on the same value.
- Starting from the system's unreliability, we require the final algorithm to satisfy P1.
- To meet the algorithm's safety requirement 2, we need to continue to meet two conditions on top of P1:

```
1. Acceptors can accept multiple proposals.
2. Satisfy P2.
```

To satisfy P2, the paper further proposes a reasonable requirement **P2a** to achieve P2.
![alt text](/assets/images/post/replication-simple-paxos/image-3.png)

P2a means: Once proposal-m value v0 is chosen, all subsequent proposal-n values (n>m) accepted by any acceptor must be v0.

Satisfying P2a can satisfy P2 because the chosen v0 essentially means the majority of acceptors have accepted v0. Now, if any acceptor accepts any proposal-n value (n>m) as v0, it naturally satisfies that the chosen value is v0.

It can be said that P2a is a stronger constraint on the acceptor's acceptance behavior because it requires any acceptor, not just the majority of acceptors.

**Problem**: *Although P2a can satisfy P2, P1 will make P2a unachievable because if a new acceptor and new proposer come online after a failover in a system with an existing chosen value (v0), and the new proposer submits a proposal value (v1) to the new acceptor, according to P1, the new acceptor must directly accept v1. This violates P2a because P2a requires any acceptor's behavior.*

**Reasoning 5**: To resolve the conflict between P1 and P2a, we need to impose constraints on the proposer's submission behavior, not allowing the proposer to submit arbitrary values, at least not submitting random values when the system already has a chosen value. Therefore, the paper further proposes the **P2b** requirement:
![alt text](/assets/images/post/replication-simple-paxos/image-4.png)

P2b requires that once proposal-m value(v0) is chosen, any proposal-n value (n>m) submitted by any proposer must be equal to v0.

This requirement resolves the conflict between P1 and P2a and is a stronger constraint because it specifies that any proposer submits v0, naturally any acceptor can only accept v0, with no other options.

**Reasoning 6**: To achieve the P2b requirement, an intuition is that the proposer needs to query the system's majority acceptors before deciding on the proposal value; otherwise, it is impossible to know if the system already has a chosen value.

**But in what way should it query? What information should it query?**

Here comes the **highlight** of the paper.

Let's reiterate P2b: When proposal-m value v0 is chosen, for n>m, any proposal-n value proposed by any proposer must be v0.

We now want to find an algorithm Q that ensures the final conclusion that any proposal-n value is v0, thus satisfying the entire reasoning chain, **P2b => P2a => P2**.

To find this algorithm Q, we need to take a detour and assume that there is already an algorithm Q that satisfies P2b, making it a theorem. Then, we can use mathematical induction to prove P2b under some algorithm Q. The corresponding mathematical induction is as follows:
```
initial:  proposal-m value v0 is chosen
last state: For any proposal-(m, n-1], all proposal-(m, n-1] values are v0

Prove: Under the Q algorithm, any proposal-n value is v0.
```

To find the Q algorithm, we first analyze what useful information can be obtained under known conditions.
1. *proposal-m value v0 is chosen*
This means there is a majority acceptors set C in the system, and every acceptor in C has accepted v0. This is determined by the definition of chosen.

2. *For any proposal-(m, n-1], all proposal-(m, n-1] values are v0*
This means every acceptor in set C has accepted at least one proposal in the range (m, n-1], which includes the above property that at least every acceptor in C has accepted proposal-m value v0.
Another property is that since any proposal-(m, n-1] value is v0, any acceptor that accepted proposal-(m, n-1] value is v0 because there are no other options.

Under the above properties, to ensure any proposal-n value is v0, the paper proposes a Q algorithm, which is the **P2c invariant** in the paper:
![alt text](/assets/images/post/replication-simple-paxos/image-5.png)

Translated, it means: Before the proposer decides on proposal-n value, it queries a majority acceptors set S for their current accepted values and selects the proposal value corresponding to the highest proposal number less than n as its proposal-n value. This ensures that proposal-n value is v0.

**Why can P2c guarantee this?**

1. If the highest proposal number is in the range (m, n-1], it can be determined that the corresponding proposal value is v0 because all proposal values in this range are v0, which is a given condition.
2. If the highest proposal number is in the range (0, m], since any majority acceptors set S will overlap with set C by at least one acceptor, the highest proposal number is m, and the accepted value is v0, which is also a given condition.

Here, it can be seen that as long as P2c is satisfied, the mathematical induction can be proved, naturally satisfying the P2b requirement.

The above is the reasoning for P2c (b) condition. P2c (a) is a more trivial case where if the majority of nodes have not accepted any value, the value can be set to v0, which does not violate the P2b requirement.

**Problem**: *To achieve P2c, there is a practical problem to solve, which is that selecting the highest proposal number in a distributed environment is very difficult because P2c is under an ideal premise (the chosen state has been reached, and proposal-m has been submitted). Then, according to P2c's requirements, determine the proposal-n value.*

**The actual scenario is not like this because each proposer/acceptor independently decides what proposal to propose/accept. When proposer n starts querying, the system may not have started submitting proposal-m, and it is impossible to know that proposal-m value will definitely become the chosen value.**

*So, the highest proposal number in P2c refers to the highest proposal number that any acceptor **may obtain** in the range (0, n] when proposer n starts querying, not the highest proposal number that any acceptor **has obtained** in the range (0, n].*

**This is a subtle difference**.

A simple example:
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

In this example, when the system submits proposal number==5, a chosen v0 will appear, but because proposer0 and proposer1 run independently, when proposer1 starts the proposal with proposal number=10, proposer0 is also running, and at this time, the system does not have any chosen value.

At this time, even if proposer0 makes the system have a chosen value=v0 state, proposer1's behavior directly violates the P2c invariant, causing the reasoning chain P2c => P2b => P2a => P2 to all fail, thus failing to meet the consensus algorithm's safety requirement 2.

**Reasoning 7**: The core point of the above problem is that in a distributed environment, after the proposer queries the acceptor's current state, it cannot predict whether there will be a proposal in the interval (current highest proposal number of acceptor, my proposal number) that will be accepted by the corresponding acceptor, that is, it cannot predict future events. For example:

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

The solution proposed in the paper is:

> Once an acceptor receives a proposal-n inquiry, it must make a promise: to reject all proposals with numbers less than n and to treat all **potentially** accepted proposals as **already** accepted proposals.

This ensures that the inquiry retrieves the highest proposal number that the acceptor has accepted during this period. Consequently, the highest proposal number obtained from all inquiries is naturally the highest proposal number that could have been accepted.

This concludes the derivation of the algorithm. Throughout this process, we have addressed all issues and established corresponding behavioral constraints for proposers and acceptors, ensuring the validity of the derivation chain.

### Paxos Algorithm Process

The behavioral constraints involved in the derivation are roughly as follows:

1. Before formally proposing, the proposer must perform a majority read to obtain the system's historical information and then submit the proposal.
2. Upon receiving an inquiry, the acceptor must make a promise to the proposer, agreeing to accept only proposals outside the promise range.

The formal algorithm process is as follows:

This algorithm process is divided into the Prepare phase and the Accept phase:

1. **Prepare Phase:**

   - **Proposer:** Obtains a new proposal number n and sends a prepare request (prepare-n) to a majority of acceptors.

   - **Acceptor:** Upon receiving prepare-n:
     - If it has already received a prepare-t (t > n) with a proposal number greater than n, it discards or responds with an invalid prepare, requesting the proposer to obtain a new proposal number.
     - Otherwise, it responds with the accepted `<highest proposal number, proposal value>`, ensuring it will not accept proposals with numbers less than n.

2. **Accept Phase:**

   - **Proposer:** If it does not receive valid responses from a majority of acceptors within a timeout period or receives an invalid prepare response, it restarts the Prepare phase. Otherwise, it selects the proposal value v corresponding to the highest proposal number from all prepare responses and submits an accept request `<n, v>`.

   - **Acceptor:** Upon receiving an accept request `<n, v>`:
     - If it has already received a prepare-t (t > n) with a proposal number greater than n, it discards or responds with an invalid proposal, requesting the proposer to obtain a new proposal number.
     - Otherwise, it accepts the proposal `<n, v>` and responds with success.

### Termination of the Paxos Algorithm

Since the Paxos algorithm ensures that only one value is chosen in the system, as long as a majority read reveals a value accepted by a majority of acceptors, the algorithm is considered complete.

Learners can obtain the final chosen value through various methods:

1. Each learner periodically performs a majority read to determine the chosen value.
2. Each time an acceptor accepts a proposal, it sends the value to all learners, allowing each learner to determine the chosen value independently.
3. Each time an acceptor accepts a proposal, it sends the value to some learners. Once these learners confirm the existence of the chosen value, they notify the other learners.

### Liveness of the Paxos Algorithm

The above derivation of the Paxos algorithm only considers the safety requirement and does not address liveness, i.e., the algorithm's ability to make progress.

From the Prepare and Accept phases of Paxos, it can be inferred that there may be scenarios where two proposers continuously invalidate each other's proposals during the Prepare phase, leading to a loop where each obtains a larger proposal number and restarts the Prepare phase, ultimately preventing the algorithm from making progress.

To address this issue, a recommended approach is to implement a random backoff delay for the next prepare attempt based on the severity of the conflict. This ensures that other proposals have sufficient time to be accepted, thereby quickly reaching a chosen state.

## Summary

The Paxos algorithm itself can only achieve consensus on a single value, such as electing a leader among three nodes. It cannot be directly applied to scenarios like log replication in practical applications. It can be considered as achieving consensus on a single write operation in a log. Log replication requires the use of Multi-Paxos to construct a state machine for logs. The original paper, "The Part-Time Parliament," provides a description of this aspect. Implementing this in practice is also quite complex.

The [simple paxos demo](https://github.com/maxshuang/simple-paxos) is a demo that simulates multiple nodes and network packet loss environments. Although not particularly comprehensive, it can be an interesting experiment.