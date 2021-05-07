# Chapter 9 - Consistency and Consensus (Part Two)

*23 Mar 2020*

In the [first part](https://timilearning.com/posts/ddia/part-two/chapter-9-1/) of Chapter 9, we looked at Linearizability and Causality as consistency guarantees and used those topics to discuss the difference between total order and partial order. We also briefly discussed Lamport timestamps and how they are used to enforce a total ordering of operations across multiple nodes.

We concluded by seeing that it's not enough to know the total order of operations *after* all the operations have been collected. A node might want to make a decision "in the moment", and so it needs to know what the order is at each point in time.

This part will focus on "Total Order Broadcast" and "Consensus", which help to solve the challenge described above.

**Table of Contents**

- [Total Order Broadcast](#total-order-broadcast)
    - [Using total order broadcast](#using-total-order-broadcast)
    - [Implementing linearizable storage using total order broadcast](#implementing-linearizable-storage-using-total-order-broadcast)
        - [Linearizable writes using total order broadcast](#linearizable-writes-using-total-order-broadcast)
        - [Linearizable reads using total order broadcast](#linearizable-reads-using-total-order-broadcast)
    - [Implementing total order broadcast using linearizable storage](#implementing-total-order-broadcast-using-linearizable-storage)
- [Distributed Transactions and Consensus](#distributed-transactions-and-consensus)
    - [Atomic Commit and Two-Phase Commit (2PC)](#atomic-commit-and-two-phase-commit-2pc)
        - [From single-node to distributed atomic commit](#from-single-node-to-distributed-atomic-commit)
        - [Introduction to Two-phase commit](#introduction-to-two-phase-commit)
            - [Coordinator Failure](#coordinator-failure)
            - [Three-phase commit](#three-phase-commit)
    - [Distributed Transactions in Practice](#distributed-transactions-in-practice)
        - [Exactly-once message processing](#exactly-once-message-processing)
        - [XA Transactions](#xa-transactions)
        - [Holding locks while in doubt](#holding-locks-while-in-doubt)
        - [Limitations of distributed transactions](#limitations-of-distributed-transactions)
    - [Fault-Tolerant Consensus](#fault-tolerant-consensus)
        - [Consensus algorithms and total order broadcast](#consensus-algorithms-and-total-order-broadcast)
        - [Single-leader replication and consensus](#single-leader-replication-and-consensus)
        - [Epoch numbering and quorums](#epoch-numbering-and-quorums)
        - [Limitations of consensus](#limitations-of-consensus)
    - [Membership and Coordination Services](#membership-and-coordination-services)
        - [Allocating work to nodes](#allocating-work-to-nodes)
        - [Service discovery](#service-discovery)
- [Conclusion](#conclusion)

* * *

## Total Order Broadcast [#](#total-order-broadcast)

We discussed earlier how single-leader replication determines a total order of operations by sequencing all operations on a single CPU core on the leader. However, when the throughput needed is greater than a single leader can handle, there's a challenge of how to scale the system with multiple nodes, as well as handling failover if the leader fails.

Note that single-leader replication systems often only maintain ordering per partition. They do not have total ordering across all partitions. An example of such a system is Kafka. Total ordering across all partitions will require additional coordination.

----

*Total Order Broadcast* (or Atomic Broadcast) is a broadcast a protocol for exchanging messages between nodes. It requires that the following safety properties are always satisfied:

- *Reliable delivery*: No messages are lost. A message delivered to one node must be delivered to all the nodes.
- *Totally ordered delivery:* Messages are delivered to every node in the same order.

----

From Wikipedia: *The broadcast is termed "atomic" because it either eventually completes correctly at all participants, or all participants abort without side effects.*

An algorithm for total order broadcast must ensure that these properties are always satisfied, even in the face of network or node faults. In the face of failures, the algorithm must keep retrying so that messages can get through when the network is repaired.

----

***Q****: How can messages be delivered in the same order in multi-leader or leaderless replication systems?*

***A:*** *In leaderless replication systems, a client typically directly sends its writes to several replicas and then uses a quorum to determine if it's successful or not. Leaderless replication does not enforce a particular order of writes. Multi-leader systems are typically not linearizable, so it doesn't apply here.*

### Using total order broadcast [#](#using-total-order-broadcast)

Total order broadcast is exactly what is needed for database replication based on a principle known as *state machine replication.* It's stated as follows:

*"If every message represents a write to the database, and every replica processes the same writes in the same order, then the replicas will remain consistent with each other (aside from any temporary replication lag)."*

*Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 8950-8951). O'Reilly Media. Kindle Edition.*

It can also be used to implement serializable transactions:

*”If every message represents a deterministic transaction to be executed as a stored procedure, and if every node processes those messages in the same order, then the partitions and replicas of the database are kept consistent with each other"*

*Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 8957-8958). O'Reilly Media. Kindle Edition.*

One thing to note with total order broadcast is that *the order is fixed at the time of message delivery*. This means that a node cannot retroactively insert a message into an earlier position if subsequent messages have been delivered. Messages must be delivered in the right order. This makes total order broadcast stronger than timestamp ordering ([since we know that time can move backward](https://timilearning.com/posts/ddia/part-two/chapter-8/#time-of-day-clocks)).

Total Order broadcast can also be seen as a way of creating a *log* (like a replication log, transaction log, or write-ahead log). Delivering a message is like appending to the log, and if all the nodes read from the log, they will see the same sequence of messages.

Another use of total order broadcast is for implementing [fencing tokens](https://timilearning.com/posts/ddia/part-two/chapter-8/#fencing-tokens)*.* Each request to acquire the lock can be appended as a message to the log, and the messages can be given a sequence number in the order of their appearance in the log. This sequence number can then be used as a fencing token due to the fact that it's monotonically increasing.

### Implementing linearizable storage using total order broadcast [#](#implementing-linearizable-storage-using-total-order-broadcast)

Although total order broadcast is a reasonably strong guarantee, it is not quite as strong as linearizability. However, they are closely related.

Total Order Broadcast guarantees that messages will be delivered reliably in the same order, but it provides no guarantee about *when* the message will be delivered (so a read to a node may return stale data). Linearizability, on the other hand, is a *recency guarantee*, it guarantees that a read will see the latest value written.

These two concepts are closely related, and so we can implement linearizable storage using total order broadcast and vice versa.

#### Linearizable writes using total order broadcast [#](#linearizable-writes-using-total-order-broadcast)

Linearizable writes are instantaneous writes, meaning that once a client has written a value, all other reads to that value must see the newly written value or the value of a later write.

Imagine that we are building a system where each user has a unique username, and we want to deal with a situation where multiple users concurrently try to grab the same username. (*Aside:* Now, the scenario I have in my head is one in which these users can write to different replicas (think multiple leaders) concurrently. I imagine that in a single leader system, we would simply use the first successful operation).

----

To ensure linearizable writes in this system using total order broadcast:

- Each node can append a message to the log indicating the username they want to claim.
- The nodes then read the log, waiting for the message they appended to be delivered back to them.
- If the first message that a node receives with the username it wants is its own message, then the node is successful and can commit the username claim. All other nodes that want to claim this username can then abort their operations.

This works because if there are several concurrent writes, all the nodes will agree on which came first, and these messages are delivered to all the nodes in the same order.

This algorithm does not guarantee linearizable reads though. A client can still get stale reads if they read from an asynchronously updated store.

#### Linearizable reads using total order broadcast [#](#linearizable-reads-using-total-order-broadcast)

There are a number of options for linearizable reads with total order broadcast:

- When you want to read a value, you could append a message to the log, read the log and then perform the actual read of the value when the message you appended is delivered back to you. I think this works because since all nodes have to agree on the order of messages, you will always see the latest write that happened before your 'read message'. If client B sends a 'write message' to the log before client A's 'read message' to the same log, client A will see the effects of that 'write message'.
- If you can fetch the position of the latest log message in a linearizable way, you can query that position, wait for all the entries up to that position to be delivered to you, and then actually perform the read. This is the idea used in Zookeeper's sync() operation.
- You can always make reads from a replica that is synchronously updated on writes.

### Implementing total order broadcast using linearizable storage [#](#implementing-total-order-broadcast-using-linearizable-storage)

What we're essentially implementing is a mechanism for generating sequence numbers for each message we want to send. The easiest way to do this is to assume that we have a linearizable register which stores an integer and has an atomic increment-and-get operation.

The algorithm is this:

*"For every message you want to send through total order broadcast, you increment-and-get the linearizable integer, and then attach the value you got from the register as a sequence number to the message."*

*Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 9023-9024). O'Reilly Media. Kindle Edition.*

----

This way, we'll avoid race conditions on the integer and each message will have a unique sequence number.

Something to note is that unlike Lamport timestamps, the numbers gotten from incrementing the linearizable register form a sequence that has no gaps. The sequence won't jump from 4 to 6. Therefore, if a node has delivered a message with a sequence number of 4 and receives an incoming message with a sequence number of 6, it must wait for message 5 before it can deliver message 6 (because messages must be delivered in the same order to all nodes, and if it delivers 6 before 5 to other nodes, it is likely breaking some order).

**Note**: It can be proved that a linearizable compare-and-set (or increment-and-get) register and total order broadcast are both equivalents to consensus, meaning that if you can solve one of those problems, we can transform it into a solution for the others.

We'll discuss the consensus problem next.

## Distributed Transactions and Consensus [#](#distributed-transactions-and-consensus)

Simply put, consensus means getting *several nodes to agree on something.* However, it turns out that this is not an easy problem to solve, and it is one of the fundamental problems in distributed computing.

----

Some situations where it is important for the nodes to agree include:

- *Leader election:* In a single-leader setup, all the nodes need to agree on which node is the leader. If the nodes don't agree on who the leader is, it could lead to a split-brain situation in which multiple 'leaders' could accept writes, leading to inconsistency and data loss.
- *Atomic commit:* If a transaction spans several nodes or partitions, there's a chance that it may fail on some nodes and succeed on others. However, to preserve the atomicity property of ACID transactions, it must either succeed or fail on *all* of them. We have to get all the nodes to *agree* on the outcome of the transaction. This is known as the *atomic commit* problem.

We'll address the atomic commit problem first before delving into other consensus scenarios.

### Atomic Commit and Two-Phase Commit (2PC) [#](#atomic-commit-and-two-phase-commit-%282pc%29)

Two-phase commit is the most commonly used algorithm for implementing atomic commit. It is a kind of consensus algorithm, but not a very good one and we'll learn why soon.

We learned in [Chapter 7](https://timilearning.com/posts/ddia/part-two/chapter-7/) that the purpose of transaction atomicity is to prevent the database from getting in an *inconsistent state* in the event of failure.

#### From single-node to distributed atomic commit [#](#from-single-node-to-distributed-atomic-commit)

Atomicity for single database node transactions is usually implemented by the storage engine. When a request is made to commit a transaction, the writes in the transaction are made durable (typically using a [write-ahead log](https://timilearning.com/posts/data-storage-on-disk/part-one/#write-ahead-logs)*)* and then a commit record is appended on disk. If the database crashes during this process, upon restarting, it decides whether to commit or rollback the transaction based on whether or not the commit record was written to the disk before the crash.

However, if multiple nodes are involved, it's not sufficient to simply send a commit request to all the nodes and then commit the transaction on each one. Some of the scenarios where multiple nodes could be involved are: a multi-object transaction in a partitioned database, or writing to a [term-partitioned index](https://timilearning.com/posts/ddia/part-two/chapter-6/#partitioning-secondary-indexes-by-term)*.*

It's possible that the commit succeeds on some nodes and fails on other nodes, which is a violation of the atomicity guarantee. Possible scenarios are:

- Some nodes may detect a violation of a uniqueness constraint or something similar and may have to abort, while other nodes are able to commit successfully.
- Some commit requests might get lost in the network, and may eventually abort due to a timeout, while other requests are successful.
- Some nodes may crash before the commit record is fully written and then have to roll back on recovery, while other nodes successfully commit.

----

If some nodes commit a transaction while others abort it, the nodes will be in an inconsistent state. Note that a transaction commit on a node must be irrevocable. It cannot be retracted once it has been committed. The reason for this is that data becomes visible to other transactions once it has been committed by a transaction, and other clients may now rely on that data. Therefore, it's important that a node commits a transaction only when it is certain that all other nodes in the transaction will commit.

#### Introduction to Two-phase commit [#](#introduction-to-two-phase-commit)

Two-phase commit (or 2PC) is an algorithm used for achieving atomic transaction commit when multiple nodes are involved. 'Atomic' in the sense that either all nodes commit or all abort.

The key thing here is that the commit process is split into two phases: the *prepare* phase and the *actual commit* phase.

----

It achieves atomicity across multiple nodes by introducing a new component known as *the coordinator*. The coordinator can run in the same process as the service requesting the transaction or in an entirely different process. When the application is ready to commit a transaction, the two phases are as follows:

1.  The coordinator sends a *prepare* request to all the nodes participating in the transaction, for which the nodes have to respond with essentially a 'YES' or 'NO' message.
2.  If all the participants reply 'YES', then the coordinator will send a *commit* request in the second phase for them to actually perform the commit. However, if *any* of the nodes reply 'NO', the coordinator sends an *abort* request to all the participants.

----

In case it's still not clear how this protocol ensures atomicity while one-phase commit across multiple nodes does not, note that there are two essential "points of no return":

- When a participant responds with "YES", it means that it must be able to commit under all circumstances. A power failure, crash, or memory issue cannot be an excuse for refusing to commit later. It *must* definitely be able to commit the transaction without error if needed.
- When the coordinator decides and that decision is written to disk, the decision is irrevocable. It doesn't matter if the commit or abort request fails at first, it must be retried forever until it succeeds. If a participant crashes before it can complete the commit/abort request, the transaction will be committed in the meantime.

##### Coordinator Failure [#](#coordinator-failure)

If any of the *prepare* requests fails or times out during a 2PC, the coordinator will abort the transaction. If any commit or abort request fails, the coordinator will retry them indefinitely.

If the coordinator fails before it can send a prepare request, a participant can safely abort the transaction. However, once a participant has received a prepare request and voted "YES", it can no longer abort by itself. It has to wait to hear from the coordinator about whether or not it should commit the transaction. The *downside* of this is that if the coordinator crashes or the network fails after a participant has responded "YES", the participant can do nothing but wait. In this state, it is said to be *in doubt* or *uncertain.*

----

The reason why a participant has to wait for the coordinator in the event of a failure is that it does not know whether the failure extends to all participants or just itself. It's possible that the network failed after the commit request was sent to one of the participants. If the *in doubt participants* then decide to abort after a timeout due to not receiving from the coordinator, it will leave the database in an inconsistent state.

In principle, the in doubt participants could communicate among themselves to find out how each participant voted and then come to an agreement, but that is not part of the 2PC protocol.

----

This possibility of failure is why the coordinator must write its decision to a transaction log on disk before sending the request to the participants. When it recovers from a failure, it can read its transaction log to determine the status of all in-doubt transactions. Transactions without a commit record in the coordinator's log are aborted. In essence, the commit point of 2PC is a regular single-node atomic commit on the coordinator.

##### Three-phase commit [#](#three-phase-commit)

Two-phase commit is referred to as a *blocking* atomic commit protocol because of the fact that it can get stuck waiting for the coordinator to recover.

An alternative to 2PC that has been proposed is an algorithm called *three-phase commit (3PC).* The idea here is that it assumes a network with bounded delays and nodes with bounded response times. This means that when a delay exceeds that bound, a participant can safely assume that the coordinator has crashed.

----

However, most practical systems have unbounded network delays and process pauses, and so it cannot guarantee atomicity. If we wrongly declare the coordinator to be dead, the coordinator could resume and end up sending commit or abort requests, even when the participants have already decided. *(**Q**: I wonder if this is something that can be avoided by ensuring that once a coordinator has been declared dead for a particular transaction, it cannot come back and send requests? Might be possible through some form of sequence numbers).*

This difficulty in coming up with a *perfect failure detector* is why 2PC continues to be used today.

### Distributed Transactions in Practice [#](#distributed-transactions-in-practice)

Distributed transactions, especially those implemented with two-phase commit, are contentious because of the performance implications and operational problems they cause. This has led to many cloud services choosing not to implement them.

However, despite these limitations, it's useful to examine them in more detail as there are lessons that can be learned from them.

----

There are two types of distributed transaction which often get conflated:

- *Database-internal distributed transactions:* This refers to transactions performed by a distributed database that spans multiple replicas or partitions. VoltDB and MySQL Cluster's NDB storage engine support such transactions. Here, all the nodes participating in the transaction are running the same database software.
- *Heterogenous distributed transactions:* Here, the participants are two or more different technologies. For example, we could have two databases from different vendors, or even non-database systems such as message brokers. Although the systems may be entirely different under the hood, a distributed transaction has to ensure atomic commit across these systems.

#### Exactly-once message processing [#](#exactly-once-message-processing)

With heterogeneous transactions, we can integrate diverse systems in powerful ways. For example, we can perform a transaction that spans across a message queue and a database. Say we want to acknowledge a message from a queue as processed if and only if the transaction for processing the message was successfully committed, we could perform this using distributed transactions. This can be implemented by atomically committing the message acknowledgment and the database writes in a single transaction.

If the transaction fails and the message is not acknowledged, the message broker can safely redeliver the message later.

----

An advantage of atomically committing a message together with the side effects of its processing is that it ensures that the message is *effectively* processed exactly once. If the transaction fails, the effects of processing the message can simply be rolled back.

However, this is only possible if all the systems involved in the transaction are able to use the same atomic commit protocol. For example, if a side effect of processing a message involves sending an email and the email server does not support two-phase commit, it will be difficult to roll-back the email. Processing the message multiple times may involve sending multiple emails.

Next, we'll discuss the atomic commit protocol that allows such heterogeneous distributed transactions.

#### XA Transactions [#](#xa-transactions)

XA (*eXtended Architecture)* is a standard for implementing two-phase commit across heterogeneous technologies.

XA is a C API for interacting with a transaction coordinator, but bindings for the API exist in other languages.

It assumes that communication between your application and the participant databases/messaging services is done through a network driver (like JDBC) or a client library which supports XA. If the driver does support XA, it will call the XA API to find out whether an operation should be part of a distributed transaction - and if so, it sends the necessary information to the participant database server. The driver also exposes callbacks needed by the coordinator to interact with the participant, through which it can ask a participant to prepare, commit, or abort.

----

The transaction coordinator is what implements the XA API. The coordinator is usually just a library that's loaded into the same process as the application issuing the transaction. It keeps track of the participants involved in a transaction, their responses after asking them to prepare, and then uses a log to keep track of its commit/abort decision for each transaction.

Note that a participant database cannot contact the coordinator directly. All of the communication must go through its client library through the XA callbacks.

#### Holding locks while in doubt [#](#holding-locks-while-in-doubt)

The reason we care so much about transactions not being stuck in doubt is *locking.* Transactions often need to take row-level locks on any rows they modify, to prevent dirty writes. These locks must be held until the transaction commits or aborts.

If we're using a two-phase commit protocol and the coordinator crashes, the locks will be held until the coordinator is restarted. No other transaction can modify these rows while the locks are held.

The impact of this is that it can lead to large parts of your application being unavailable: If other transactions want to access the rows held by an in-doubt transaction, they will be blocked until the transaction is resolved.

#### Limitations of distributed transactions [#](#limitations-of-distributed-transactions)

While XA transactions are useful for coordinating transactions across heterogeneous data systems, we have seen that they can introduce major operational problems. One key insight here is that the transaction coordinator is a kind of database itself (in the sense that it keeps track of a transaction log which is durably persisted), and so it needs to be treated with the same level of importance as other databases. Some of the other limitations of distribute transactions are:

- 2PC needs *all* participants to respond before it can commit a transaction. As a result, if *any* part of the system is broken, the transaction will fail. This means that distributed transactions have a tendency of *amplifying failures*, which is not what we want when building fault-tolerant systems.
- If the coordinator is not replicated across multiple machines, it becomes a single point of failure for the system.
- XA needs to be compatible across a wide range of data systems and so it is a lowest common denominator meaning that it cannot have implementations that are specific to any system. For example, it cannot detect deadlocks across different systems, as that would require a standardized protocol with which different systems can inform each other on what locks are being held by another transaction. It also cannot work with [Serializable Snapshot Isolation](https://timilearning.com/posts/ddia/part-two/chapter-7/#serializable-snapshot-isolation), as we would need a protocol for identifying conflicts across multiple systems.

### Fault-Tolerant Consensus [#](#fault-tolerant-consensus)

In simple terms, consensus means getting several nodes to agree on something. For example, if we have several people concurrently trying to book the same meeting room or the same username, we can use a consensus algorithm to determine which one should be the winner.

In formal terms, we describe the consensus problem like this: One or more nodes may *propose* values, and the role of the consensus algorithm is to *decide* on one of those values. In the case of booking a meeting room, each node handling a user request may propose the username of the user making the request, and the consensus algorithm will decide on which user will get the room.

----

A consensus algorithm must satisfy the following properties:

- *Uniform agreement:* No two nodes decide differently.
- *Integrity:* No node decides twice.
- *Validity:* If a node decides a value *v,* then *v* was proposed by some node.
- *Termination:* Every node that does not crash eventually decides some value.

The core idea of consensus is captured in the uniform agreement and integrity properties: everyone must decide on the same outcome, and once the outcome has been decided, you cannot change your mind.

The validity property is mostly to rule out trivial solutions such as an algorithm that will always decide *null* regardless of what was proposed. An algorithm like that would satisfy the first two properties, but not the validity property.

The termination property is what ensures fault tolerance in consensus-based systems. Without this property, we could designate one node as the "dictator" and let it make all the decisions. However, if that node fails, the system will not be able to make a decision. We saw this situation in the case of two-phase commit which leaves participants in doubt.

What the termination property means is that a consensus algorithm cannot sit idle and do nothing forever i.e. it must make progress. If some nodes fail, the other nodes must reach a decision. [Note that termination is a liveness property, while the other three are safety properties.](https://timilearning.com/posts/ddia/part-two/chapter-8/#safety-and-liveness)

----

The consensus system model assumes that when a node "crashes", it disappears and never comes back. That means that any algorithm which must wait for a node to recover will not satisfy the termination property. However, note that this is subject to the assumption that fewer than half of the nodes crashed.

Note that the distinction between the safety and the liveness properties means that even if the termination property is not met, it cannot corrupt the consensus system by causing it to make invalid decisions. In addition, most consensus algorithms assume that there are no [Byzantine faults](https://timilearning.com/posts/ddia/part-two/chapter-8/#byzantine-faults). This means that if a node is Byzantine-faulty, it may break the safety properties of the protocol.

#### Consensus algorithms and total order broadcast [#](#consensus-algorithms-and-total-order-broadcast)

The most popular fault-tolerant consensus algorithms are Paxos, Zab, Raft, and Viewstamped Replication. However, most of these algorithms do not directly make use of the formal model described above i.e. proposing and deciding on a single value, while satisfying the liveness and safety properties. What these algorithms do is that they decide on a *sequence* of values, which makes them *total order broadcast* algorithms.

----

Recall from the discussion earlier that the following properties must be met for total order broadcast:

- Messages must be delivered to all nodes in the same order.
- No messages are lost.

----

If we look closely at these properties, total order broadcast can be seen as performing several rounds of consensus as all the nodes have to *agree* on what message goes next in the total order sequence. Each consensus decision can be seen as corresponding to one message delivery.

*Viewstamped Replication, Raft, and Zab implement total order broadcast directly, because that is more efficient than doing repeated rounds of one-value-at-a-time consensus. In the case of Paxos, this optimization is known as Multi-Paxos.*

*Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 9474-9476). O'Reilly Media. Kindle Edition.*

#### Single-leader replication and consensus [#](#single-leader-replication-and-consensus)

We've learned about single-leader replication takes all the writes to the leader and applies them to the followers in the same order, thereby keeping the replicas up to date. This is the same idea as total order broadcast, but interestingly, we haven't discussed consensus yet in the context of single-leader replication.

Consensus in single-leader replication depends on how the leader is chosen. If the leader is always chosen by a manual operator, there's a risk that it will not satisfy the termination property of consensus if the leader is unavailable for any reason.

Alternatively, some databases perform automatic leader election and failover by promoting a new leader if the old leader fails. However, in these systems, there is a risk of split-brain (where two nodes could think they're the leader) and so we still need all the nodes to *agree* on who the leader is.

So it looks like:

Consensus algorithms are actually total order broadcast algorithms -> total order broadcast algorithms are like single leader replication -> single-leader replication needs consensus to determine the leader - > (repeat cycle)

How do we break this cycle where it looks like to solve consensus, we must first solve consensus? We'll discuss that next.

#### Epoch numbering and quorums [#](#epoch-numbering-and-quorums)

The consensus protocols discussed above all use a leader internally. However, a key to note is that they don't guarantee that a leader is unique. They provide a weaker guarantee instead: The protocols define a monotonically increasing *epoch number* and the guarantee is that within each epoch, the leader is unique.

Whenever the current leader is thought to be dead, the nodes start a vote to elect a new leader. In each election round, the epoch number is incremented. If we have two leaders belonging to different epochs, the one with the higher epoch number will prevail.


----


Before a leader can decide anything, it must be sure that there is no leader with a higher epoch number than it. It does this by collecting votes from a *quorum* (typically the majority, but not always) of nodes for every decision that it wants to make. A node will vote for a proposal *only* if it is not aware of another leader with a higher epoch.

Therefore, we have two voting rounds in consensus protocols: one to elect a leader, and another to vote on a leader's proposal. The important thing is that there must be an overlap in the quorum of nodes that participate in both voting rounds. If a vote on a proposal succeeds, then at least one of the nodes that voted for it must have also been voted in the most recent leader election.

----

The biggest differences between 2PC and fault-tolerant consensus algorithms are that the coordinator in 2PC is not elected, and the latter only requires votes from a majority of nodes unlike in 2PC where all the participants must say "YES". In addition, consensus algorithms define a recovery process to get the nodes into a consistent state after a new leader is elected. These differences are what make consensus algorithms more fault-tolerant.

#### Limitations of consensus [#](#limitations-of-consensus)

Consensus algorithms have numerous advantages like bringing concrete safety properties, providing total order broadcast, and therefore linearizable operations in a fault-tolerant way, but they are not used everywhere because they come with a cost.

A potential downside of consensus systems is that they require a strict majority to operate. This means that to tolerate one failure, you need three nodes, and to tolerate two failures, a minimum of five nodes are needed.

Another challenge is that they rely on timeouts to detect failed nodes, and so in a system with variable network delays, it's possible for a node to falsely think that the leader has failed. This won't affect the safety properties, but it can lead to frequent leader elections which could harm system performance.

### Membership and Coordination Services [#](#membership-and-coordination-services)

Zookeeper and etcd are typically described as "distributed key-value stores" or "coordination and configuration services". These services look like databases which for which you can read and write the value of a given key, or iterate over keys.

However, it's important to note that these systems are not designed to be used as a general-purpose database. Zookeeper and etcd are designed to hold small amounts of data *in memory,* all of your application's data cannot be stored there. This small amount of data is then replicated across all the nodes using a *fault-tolerant total order broadcast algorithm.*

----

Some of the features provided by these services are:

- *Linearizable atomic operations*: Zookeeper can be used to implement distributed locks using an atomic compare-and-set operation. If several nodes concurrently try to obtain a lock on a row, it can help to guarantee that only one of them will succeed and the operation will be atomic and linearizable.
- *Total ordering of operations*: We've discussed [fencing tokens](https://timilearning.com/posts/ddia/part-two/chapter-8/#fencing-tokens) before, which can be used to prevent the clients from conflicting with each other when they want to access a resource protected by a lock or lease. Zookeeper helps to provide this by giving each operation a monotonically increasing transaction ID and version number.
- *Failure detection:* Clients maintain a long-lived session and Zookeeper servers, and both client and server periodically exchange 'heartbeats' to check that the other node is alive. If the heartbeats cease for a duration longer than the session timeout, Zookeeper will declare the session to be dead.
- *Change notifications:* With Zookeeper, clients can be made aware of when other nodes (clients) join the cluster since the new node will write to Zookeeper. A client can also be made aware of when another client leaves the cluster.

Note that only the linearizable atomic operations here require consensus, but the other features make Zookeeper useful for coordination among distributed systems. Also note that an application developer will rarely interact with Zookeeper. It is often relied on indirectly via another project like Kafka, Hbase, Hadoop YARN etc.

#### Allocating work to nodes [#](#allocating-work-to-nodes)

When new nodes join a partitioned cluster, some of the partitions need to be moved from existing nodes to the new ones in order to rebalance the load. Similarly, when nodes fail or are removed from the cluster, the partitions that they held have to be moved to the remaining nodes. Zookeeper can help to achieve tasks like this through the use of atomic operations, change notifications and ephemeral nodes.

----

An important thing to note is that Zookeeper typically manages data that is quite *slow-changing*:

*It represents information like “the node running on IP address 10.1.1.23 is the leader for partition 7,” and such assignments usually change on a timescale of minutes or hours.*

*Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 9615-9616). O'Reilly Media. Kindle Edition.*

It is not intended for storing the runtime state of an application, which is likely to change thousands or millions of times per second. Apache BookKeeper is a tool used to replicate runtime state to other nodes.

#### Service discovery [#](#service-discovery)

Service discovery is the process of finding out the IP address that you need to connect to in order to reach a service. Zookeeper, Consul, and etc are often used for service discovery.

The main idea is that services will register their network endpoints in a service registry from which they can be discovered by other services. The read requests for a service's endpoint do not need to be linearizable (DNS is the traditional method of retrieving the IP address of a service name and for availability purposes, its reads are not linearizable).

## Conclusion [#](#conclusion)

This will be the last set of notes I'll post from the book in a while. The last section of the book is on "Derived Data" and is a lot more practical than the theory we have discussed so far. In the meantime, I intend to post another set of notes from [this](https://pdos.csail.mit.edu/6.824/index.html) course which I will be starting soon.