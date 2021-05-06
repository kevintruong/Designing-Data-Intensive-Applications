Chapter 5 - Replication - A blog by Timi Adeniran

*13 Dec 2019*

My notes from the fifth chapter of Martin Kleppmann's book: Designing Data Intensive Applications.

**Table of Contents**

- [Leaders and Followers](#leaders-and-followers)
    - [Synchronous Versus Asynchronous Replication](#synchronous-versus-asynchronous-replication)
        - [Synchronous Replication](#synchronous-replication)
        - [Asynchronous Replication](#asynchronous-replication)
        - [Setting Up New Followers](#setting-up-new-followers)
        - [Handling Node Outages](#handling-node-outages)
        - [Scenario A - Follower Failure: Catch-up recovery](#scenario-a---follower-failure-catch-up-recovery)
        - [Scenario B - Leader failure: Failover](#scenario-b---leader-failure-failover)
        - [Implementation of Replication Logs](#implementation-of-replication-logs)
- [Problems with Replication Lag](#problems-with-replication-lag)
    - [Other Consistency Levels](#other-consistency-levels)
    - [Solutions for Replication Lag](#solutions-for-replication-lag)
- [Multi-Leader Replication](#multi-leader-replication)
    - [Use Cases for Multi-Leader Replication](#use-cases-for-multi-leader-replication)
    - [Handling Write Conflicts.](#handling-write-conflicts)
        - [Synchronous versus asynchronous conflict detection](#synchronous-versus-asynchronous-conflict-detection)
    - [Conflict Avoidance](#conflict-avoidance)
    - [Converging toward a consistent state](#converging-toward-a-consistent-state)
    - [Custom Conflict Resolution Logic](#custom-conflict-resolution-logic)
    - [Multi-Leader Replication Topologies](#multi-leader-replication-topologies)
- [Leaderless Replication](#leaderless-replication)
    - [Preventing Stale Reads](#preventing-stale-reads)
    - [Read repair and anti-entropy](#read-repair-and-anti-entropy)
    - [Quorums for reading and writing](#quorums-for-reading-and-writing)
    - [Limitations of Quorum Consistency](#limitations-of-quorum-consistency)
        - [Monitoring Staleness](#monitoring-staleness)
    - [Sloppy Quorums and Hinted Handoff](#sloppy-quorums-and-hinted-handoff)
        - [Multi-datacenter operation](#multi-datacenter-operation)
    - [Detecting Concurrent Writes](#detecting-concurrent-writes)
        - [Last write wins (discarding concurrent writes)](#last-write-wins-discarding-concurrent-writes)
        - [The "happens-before" relationship and concurrency](#the-happens-before-relationship-and-concurrency)
        - [Capturing the happens-before relationship](#capturing-the-happens-before-relationship)
        - [Merging Concurrently Written Values](#merging-concurrently-written-values)
        - [Version Vectors](#version-vectors)

* * *

Replication involves keeping a copy of the same data on multiple machines connected via a network. Reasons for this involve:

- Increasing the number of machines that can handle failed requests - Leads to increased read throughput.
- To allow a system continue working in the event of failed parts - Leads to increased availability.
- To keep data geographically close to users - Reduced latency.

The challenge with replication lies in handling changes to replicated data. Three algorithms for replicating changes between nodes:

- Single leader
- Multi-leader
- Leaderless

*Aside:* These replication models are per database table or partition (or index for Elasticsearch). So in Elasticsearch for example, each index in a cluster has its primary shard, and only the primary shard can accept writes. Therefore, a node can have both primary shards and replica shards.

### Leaders and Followers [#](#leaders-and-followers)

Every node that keeps a copy of data is a *replica*. Obvious question is: how do we make sure that the data on all the replicas is the same? The most common approach for this is ***leader-based replication*.** In this approach:

1.  Only the leader accepts writes.
2.  The followers read off a replication log and apply all the writes in the same order that they were processed by the leader.
3.  A client can query either the leader or any of its followers for read requests.

So here, the followers are read-only, while writes are only accepted by the reader.

This approach is used by MySQL, PostgreSQL etc., as well as non-relational databases like MongoDB, RethinkDB, and Espresso.

#### Synchronous Versus Asynchronous Replication [#](#synchronous-versus-asynchronous-replication)

With Synchronous replication, the leader must wait for a positive acknowledgement that the data has been replicated from at least one of the followers before terming the write as successful, while with Asynchronous replication, the leader does not have to wait.

##### Synchronous Replication [#](#synchronous-replication)

The advantage of synchronous replication is that if the leader suddenly fails, we are guaranteed that the data is available on the follower.

The disadvantage is that if the synchronous follower does not respond (say it has crashed or there's a network delay or something else), the write cannot be processed. A leader must block all writes and wait until the synchronous replica is available again. Therefore, it's impractical for all the followers to be synchronous, since just one node failure can cause the system to become unavailable.

In practice, enabling synchronous replication on a database usually means that *one* of the followers is synchronous, and the others are asynchronous. If the synchronous one is down, one of the asynchronous followers is made synchronous. This configuration is sometimes called *semi-synchronous*.

##### Asynchronous Replication [#](#asynchronous-replication)

In this approach, if the leaders fails and is not recoverable, any writes that have not been replicated to followers are lost.

An advantage of this approach though, is that the leader can continue processing writes, even if all its followers have fallen behind.

There's some research into how to prevent asynchronous-performance like systems from losing data if the leader fails. A new replication method called *Chain replication* is a variant of synchronous replication that aims to provide good performance and availability without losing data.

##### Setting Up New Followers [#](#setting-up-new-followers)

New followers can be added to an existing cluster for reasons such as to replace a failed node, or to add an additional replica. The next question is how to ensure the new follower has an accurate copy of the leader's data?

Two options that are not sufficient are:

- Just copying data files from one node to another. The data in the leader is always updating and a copy will see different versions at different points in time.
- Locking the database (hence making it unavailable for writes). This will go against the goal of high availability.

There's an option that works without downtime which involves:

1.  Taking a consistent snapshot of the leader's db at some point in time. It's possible to do this without taking a lock on the entire db. Most databases have this feature.
2.  Copy the snapshot to the follower node.
3.  The follower requests all the data changes that happened since the snapshot was taken.
4.  When the follower has processed the log of changes since the snapshot, we say it has caught up.

In some systems, this process is fully automated, while in others, it is manually performed by an administrator.

##### Handling Node Outages [#](#handling-node-outages)

Any node can fail, therefore, we need to keep the system running despite individual node failures, and minimize the impact of a node outage. How do we achieve high availability with leader-based replication?

##### Scenario A - Follower Failure: Catch-up recovery [#](#scenario-a---follower-failure%3A-catch-up-recovery)

Each follower typically keeps a local log of the data changes it has received from the leader. If a follower node fails, it can compare its local log to the replication log maintained by the leader, and then process all the data changes that occurred when the follower was disconnected.

##### Scenario B - Leader failure: Failover [#](#scenario-b---leader-failure%3A-failover)

This is trickier: One of the nodes needs to be promoted to be the new leader, clients need to be reconfigured to send their writes to the new leader, and the other followers need to start consuming data changes from the new leader. This whole process is called a *failover.* Failover can be handled manually or automatically. An automatic failover consists of:

1.  *Determining that the leader has failed*: Many things could go wrong: crashes, power outages, network issues etc. There's no foolproof way of determining what has gone wrong, so most systems use a timeout. If the leader does not respond within a given interval, it's assumed to be dead.
2.  *Choosing a new leader:* This can be done through an election process (where the new leader is chosen by a majority of the remaining replicas), or a new leader could be appointed by a previously elected *controller node.* The best candidate for leadership is typically the one with the most up-to-date data changes from the old leader (to minimize data loss)
3.  *Reconfiguring the system to use the new leader:* Clients need to send write requests to the new leader, and followers need to process the replication log from the new leader. The system also needs to ensure that when the old leader comes back, it does not believe that it is still the leader. It must become a follower.

There are a number of things that can go wrong during the failover process:

- For asynchronous systems, we may have to discard some writes if they have not been processed on a follower at the time of the leader failure. This violates clients' durability expectations.
- Discarding writes is especially dangerous if other storage systems are coordinated with the database contents. For example, say an autoincrementing counter is used as a MySQL primary key and a redis store, if the old leader fails and some writes have not been processed, the new leader could begin using some primary keys which have already been assigned in redis. This will lead to inconsistency in the data, and it's what happened to Github ([https://github.blog/2012-09-14-github-availability-this-week/](https://github.blog/2012-09-14-github-availability-this-week/)).
- In fault scenarios, we could have two nodes both believe that they are the leader: *split brain.* Data is likely to be lost/corrupted if both leaders accept writes and there's no process for resolving conflicts. Some systems have a mechanism to shut down one node if two leaders are detected. This mechanism needs to be designed properly though, or what happened at Github can happen again( [https://github.blog/2012-12-26-downtime-last-saturday/](https://github.blog/2012-12-26-downtime-last-saturday/))
- It's difficult to determine the right timeout before the leader is declared dead. If it's too long, it means a longer time to recovery in the case where the leader fails. If it's too short, we can have unnecessary failovers, since a temporary load spike could cause a node's response time to increase above the timeout, or a network glitch could cause delayed packets. If the system is already struggling with high load or network problems, unnecessary failover can make the situation worse.

##### Implementation of Replication Logs [#](#implementation-of-replication-logs)

Several replication methods are used in leader-based replication. These include:

**a) Statement-based replication:** In this approach, the leader logs every write request (statement) that it executes, and sends the statement log to every follower. Each follower parses and executes the SQL statement as if it had been received from a client.

- A problem with this approach is that a statement can have different effects on different followers. A statement that calls a nondeterministic function such as NOW() or RAND() will likely have a different value on each replica.
- If statements use an autoincrementing column, they must be executed in exactly the same order on each replica, or else they may have a different effect. This can be limiting when executing multiple concurrent transactions, as statements without any causal dependencies can be executed in any order.
- Statements with side effects (e.g. triggers, stored procedures) may result in different side effects occurring on each replica, unless the side effects are deterministic.

Some databases work around this issues by requiring transactions to be deterministic, or configuring the leader to replace nondeterministic function calls with a fixed return value.

**b) Write-ahead log (WAL) shipping:** The log is an append-only sequence of bytes containing all writes to the db. Besides writing the log to disk, the leader can also send the log to its followers across the network.

The main disadvantage of this approach is that the log describes the data on a low level. It details which bytes were changed in which disk blocks. This makes the replication closely coupled to the storage engine. Meaning that if the storage engine changes in another version, we cannot have different versions running on the leader and the followers, which prevents us from making zero-downtime upgrades.

**c) Logical (row-based) log replication:** This logs the changes that have occurred at the granularity of a row. Meaning that:

- For an inserted row, the log contains the new values of all columns.
- For a deleted row , the log contains enough information to identify the deleted column. Typically the primary key, but it could also log the old values of all columns.
- For an updated row, it contains enough information to identify the updated row, and the new values of all columns.

This decouples the logical log from the storage engine internals. Thus, it makes it easier for external applications (say a data warehouse for offline analysis, or for building custom indexes and caches) to parse. This technique is called *change data capture.*

**d) Trigger-based replication:** This involves handling replication within the application code. It provides flexibility in dealing with things like: replicating only a subset of data, conflict resolution logic, replicating from one kind of database to another etc. *Trigger* and *Stored procedures* provide this functionality. This method has more overhead than other replication methods, and is more prone to bugs and limitations than the database's built-in replication.

### Problems with Replication Lag [#](#problems-with-replication-lag)

**Eventual Consistency**: If an application reads from an asynchronous follower, it may see outdated information if the follower has fallen the leader. This inconsistency is a temporary state, and the followers will eventually catchup. That's *eventual consistency.*

The delay between when a write happens on a reader and gets reflected on a follower is *replication lag*.

#### Other Consistency Levels [#](#other-consistency-levels)

There are a number of issues that can occur as a result of replication lag. In this section, I'll summarize them under the minimum consistency level needed to prevent it from happening.

**a) Reading Your Own Writes:** If a client writes a value to a leader and tries to read that same value, the read request might go to an asynchronous follower that has not received the write yet as a result of replication lag. The user might think the data was lost, when it really wasn't. The consistency level needed to prevent this situation is known as *read-after-write consistency* or *read-your-writes consistency.* It makes the guarantee that a user will always see their writes. There are a number of various techniques for implementing this:

- When reading a field that a user might have modified, read it from the leader, else read it from a follower. E.g. A user's profile on a social network can only be modified by the owner. A simple rule could be that user's profiles are always read from the leader, and other users' profiles are read from a follower.
- Of course this won't be effective if most things are editable by the user, since it'll drive most reads to the leader. Another option is to keep track of the time of the update, and only read from followers whose last updated time is at least that. The timestamp could be a logical one, like a sequence of writes.

There's an extra complication with this if the same user is accessing my service across multiple devices say a desktop browser and a mobile app. They might be connected through different networks, yet we need to make sure they're in sync. This is known as *cross-device* read-after-write consistency. This is more complicated for reasons like the fact that:

- We can't use the last update time as suggested earlier, since the code on one device will not know about what updates have happened on the other device.
- If replicas are distributed across different datacenters, each device might hit different data datacenters which will have followers that may or may not have received the write. A solution to this is to force that all the reads from a user must be routed to the leader. This will of course introduce the complexity of routing all requests from all of a user's devices to the same datacenter.

**b) Monotonic Reads:** An anomaly that can occur when reading from asynchronous followers is that it's possible for a user to see things *moving backward in time.* Imagine a scenario where a user makes the same read multiple times, and each read request goes to a different follower. It's possible that a write has appeared on some followers, and not on others. Time might seem to go backwards sometimes when the user sees old data, after having read newer data.

Monotonic reads is a consistency level that guarantees that a user will not read older data after having previously read newer data. This guarantee is stronger than eventual consistency, but weaker than strong consistency.

A solution to this is that every read from a user should go to the same replica. The hash of a user's id could be used to determine what replica to go to.

**c) Consistent Prefix Reads:** Another anomaly that can occur as a result of replication lag is a violation of causality. Meaning that a sequence of writes that occur in one order might be read in another order. This can especially happen in distributed databases where different partitions operate independently and there's no global ordering of writes. *Consistent prefix reads* is a guarantee that prevents this kind of problem.

One solution is to ensure that causally related writes are always written to the same partition, but this cannot always be done efficiently.

#### Solutions for Replication Lag [#](#solutions-for-replication-lag)

Application developers should ideally not have to worry about subtle replication issues and should trust that their databases "do the right thing". This is why *transactions* exist. They allow databases to provide stronger guarantees about things like consistency. However, many distributed databases have abandoned transactions because of the complexity, and have asserted that eventual consistency is inevitable. Martin discusses these claims later in the chapter.

### Multi-Leader Replication [#](#multi-leader-replication)

The downside of single-leader replication is that all writes must go through that leader. If the leader is down, or a connection can't be made for whatever reason, you can't write to the database.

Multi-leader/Master-master/Active-Active replication allows more than one node to accept writes. Each leader accepts writes from a client, and acts as a follower by accepting the writes on other leaders.

#### Use Cases for Multi-Leader Replication [#](#use-cases-for-multi-leader-replication)

- **Multi-datacenter operation:** Here, each datacenter can have its own leader. This has a better performance for writes, since every write can be processed in its local datacenter (as opposed to being transmitted to a remote datacenter) and replicated asynchronously to other datacenters. It also means that if a datacenter is down, each data center can continue operating independently of the others.
    
    Multi-leader replication has the disadvantage that the same data may be concurrently modified in two different datacenters, and so there needs to be a way to handle conflicts.
    
- **Clients with offline operation:** Some applications need to work even when offline. Say mobile apps for example, apps like Google Calendar need to accept writes even when the user is not connected to the internet. These writes are then asynchronously replicated to other nodes when the user is connected again. In this setup, each device stores data in its local database. Meaning that each device essentially acts like a leader. CouchDB is designed for this mode of operation apparently.
    
- **Collaborative Editing:** Real-time collaborative editing applications like Confluence and Google Docs allow several people edit a document at the same time. This is also a database replication problem. Each user that edits a document has their changes saved to a local replica, from which it is then replicated asynchronously.
    
    For faster collaboration, the unit of change can be a single keystroke. That is, after a keystroke is saved, it should be replicated.
    

#### Handling Write Conflicts. [#](#handling-write-conflicts.)

Multi-leader replication has the big disadvantage that write conflicts can occur, which requires conflict resolution.

If two users change the same record, the writes may be successfully applied to their local leader. However, when the writes are asynchronously replicated, a conflict will be detected. This does not happen in a single-leader database.

##### Synchronous versus asynchronous conflict detection [#](#synchronous-versus-asynchronous-conflict-detection)

In theory, we could make conflict detection synchronous, meaning that we wait for the write to be replicated to all replicas before telling the user that the write was successful. Doing this will make one lose the main advantage of multi-leader replication though, which is allowing each replica to accept writes independently. Use single-leader replication if you want synchronous conflict detection.

#### Conflict Avoidance [#](#conflict-avoidance)

Conflict avoidance is the simplest strategy for dealing with conflicts. Conflicts can be avoided by ensuring that all the writes for a particular record go through the same leader. For example, you can make all the writes for a user go to the same datacenter, and use the leader there for reading and writing. This of course has a downside that if a datacenter fails, traffic needs to be rerouted to another datacenter, and there's a possibility of concurrent writes on different leaders, which could break down conflict avoidance.

#### Converging toward a consistent state [#](#converging-toward-a-consistent-state)

A database must resolve conflicts in a convergent way, meaning that all the replicas must arrive at the same final value when all changes have been replicated.

Various ways of achieving this are by:

- Giving each write a unique ID ( e.g. a timestamp, UUID etc.), pick the write with the highest ID as the winner, and throw away the other writes. If timestamp is used, that's *Last write wins*, it's popular, but also prone to data loss.
- Giving each replica a unique ID, and letting writes from the higher-number replica always take precedence over writes from a lower-number replica. This is also prone to data loss.
- Recording the conflict in an explicit data structure that preserves the information, and writing application code that resolves the conflict at some later time (e.g. by prompting the user).
- Merge the values together e.g. ordering them alphabetically.

#### Custom Conflict Resolution Logic [#](#custom-conflict-resolution-logic)

The most appropriate conflict resolution method may depend on the application, and thus, multi-leader replication tools often let users write conflict resolution logic using application code. The code may be executed on read or on write:

- *On write:* When the database detects a conflict in the log of replicated changes, it calls the conflict handler. The handler typically runs in a background process and must execute quickly. It has no user interaction.
- *On Read:* Conflicting writes are stored. However, when the data is read, the multiple versions of the data are returned to the user, either for the user to resolve them or for automatic resolution.

Automatic conflict resolution is a difficult problem, but there are some research ideas being used today:

- Conflict-free replicated datatypes (CRDTs) - Used in Riak 2.0
- Mergeable persistent data structure - Similar to Git. Tracks history explicitly
- Operational transformation: Algorithm behind Google Docs.

It's still an open area of research though.

#### Multi-Leader Replication Topologies [#](#multi-leader-replication-topologies)

A *replication topology* is the path through which writes are propagated from one node to another. The most general topology is *all-to-all,* where each leader sends its writes to every other leader. Other types are *circular topology* and *star topology.*

All-to-all topology is more fault tolerant than the circular and star topologies because in those topologies, one node failing can interrupt the flow of replication messages across other nodes, making them unable to communicate until the node is fixed.

### Leaderless Replication [#](#leaderless-replication)

In this replication style, the concept of a leader is abandoned, and any replica can typically accept writes from clients directly.

This style is used by Amazon for its in-house *Dynamo* system. Riak, Cassandra and Voldermort also use this model. These are called *Dynamo* style systems.

In some leaderless implementations, the client writes directly to several replicas, while in others there's a coordinator node that does this on behalf of the client. Unlike a leader database though, this coordinator does not enforce any ordering of the writes.

#### Preventing Stale Reads [#](#preventing-stale-reads)

Say there are 3 replicas and one of the replicas goes down. A client could write to the system and have 2 of the replicas successfully acknowledge the write. However, when the offline node gets back up, anyone who reads from it may get stale responses.

To prevent stale reads, as well as writing to multiple replicas, the client also reads from multiple replicas in parallel. Version numbers are attached to the result to determine which value is newer.

#### Read repair and anti-entropy [#](#read-repair-and-anti-entropy)

When offline nodes come back up, the replication system must ensure that all data is eventually copied to every replica. Two mechanisms used in Dynamo-style datastores are:

- *Read repair*: When data is read from multiple replicas and the system detects that one of the replicas has a lower version number, the data could be copied to it immediately. This works for frequently read values, but has the downside that any data that is not frequently read may be missing from some replicas and thus have reduced durability.
- *Anti-entropy process*: In addition to the above, some databases have a background process that looks for differences in data between replicas and copies any missing data from one replica to another. This process does not copy writes in any particular order, and there may be a notable delay before data is copied.

#### Quorums for reading and writing [#](#quorums-for-reading-and-writing)

Quorum reads and writes refer to the minimum number of votes for a read or a write to be valid. If there are *n* replicas, every write must be confirmed by at least *w* nodes to be considered successful, and every read must be confirmed by at least *r* nodes to be successful. The general rule that the number chosen for *r* and *w* should obey is that:

*w + r > n.*

This way, we can typically expect an up-to-date value when reading because at least one of the *r* nodes we're reading from must overlap with the *w* nodes (barring sloppy quorums which are discussed below)

The parameters *n, w,* and *r* are typically configurable. A common choice is to make n an odd number such that w = r = (n + 1)/2. These numbers can be varied though. For a workload with few writes and many reads, it may make sense to set w = n and r = 1. Of course this has the disadvantage of reduced availability for writes if just one node fails.

Note that *n* does not always refer to the number of nodes in the cluster, it may just be the number of nodes that any given value must be stored on. This allows datasets to be partitioned. Partitioning is discussed in Chapter 5.

Notes:

- With *w* and *r* being less than *n,* we can still process writes if a node is unavailable.
- Reads and writes are always sent to all n replicas in parallel, *w* and *r* determine how many nodes we wait for i.e. how many nodes need to report success before we consider the read or write to be successful.

#### Limitations of Quorum Consistency [#](#limitations-of-quorum-consistency)

Quorums don't necessarily have to be majorities i.e. *w + r > n.* What matters is that the sets of nodes used by the read and write operations overlap in at least one node.

We could also set *w* and *r* to smaller numbers, so that *w + r ≤ n.* With this, reads and writes are still sent to n nodes, but a smaller number of successful responses is required for the operation to succeed. However, you are also more likely to read stale values, as it's more likely that a read did not include the node with the latest value.

The upside of the approach though is that it allows lower latency and higher availability: if there's a network interruption and many replicas become unreachable, there's a higher chance that reads and writes can still be processed.

Even if we configure our database such that *w + r > n* , there are still edge cases where stale values may be returned. Possible scenarios are:

- If a sloppy quorum is used, the nodes for reading and writing may not overlap. Sloppy quorums are discussed further down.
- If two writes occur concurrently, it's still not clear which happened first. Therefore, the database may wrongly return the more stale one. If we pick a winner based on a timestamp (last write wins), writes can be lost due to clock skew.
- If a write happens concurrently with a read, the write may be reflected on only some of the replicas. It's unclear whether the read will return the old or new value.
- In a non-transaction model, if a write succeeds on some replicas but fails on others, it is not rolled back on the replicas where it succeeded.

From these points and others not listed, there is no absolute guarantee that quorum reads return the latest written value. These style of databases are optimized for use cases that can tolerate eventual consistency. Stronger guarantees require transactions or consensus.

##### Monitoring Staleness [#](#monitoring-staleness)

It's important to monitor whether databases are returning up-to-date results, even if the application can tolerate stale reads. If a replica falls behind significantly, the database should alert you so that you can investigate the cause.

For leader-based replication, databases expose metrics for the replication lag. It's possible to do this because writes are applied to the leader and followers in the same order. We can determine how far behind a follower has fallen from a leader by subtracting it's position from the leader's current position.

This is more difficult in leaderless replication systems as there is no fixed order in which writes are applied. There's some research into this, but it's not common practice yet.

#### Sloppy Quorums and Hinted Handoff [#](#sloppy-quorums-and-hinted-handoff)

Databases with leaderless replication are appealing for use cases where high availability and low latency is required, as well as the ability to tolerate occasional stale reads. This is because they can tolerate failure of individual nodes without needing to failover since they're not relying on one node. They can also tolerate individual nodes going slow, as long as *w* or *r* nodes have responded.

Note that the quorums described so far are not as fault tolerant as they can be. If any of the designated *n* nodes is unavailable for whatever reason, it's less likely that you'll be able to have *w* or *r* nodes reachable, making the system unavailable. Nodes being unavailable can be caused by anything, even something as simple as a network interruption.

To make the system more fault tolerant, instead of returning errors to all requests for which can't reach a quorum of *w* or *r* nodes, the system could accept reads and writes on nodes that are reachable, even if they are not among the designated *n* nodes on which the value usually lives. This concept is known as a *sloppy quorum.*

With a sloppy quorum, during network interruptions, reads and writes still require *r* and *w* successful responses, but they do not have to be among the designated *n* "home" nodes for a value. These are like temporary homes for the value.

When the network interruption is fixed, the writes that were temporarily accepted on behalf of another node are sent to the appropriate "home" node. This is *hinted handoff.*

Sloppy quorums are particularly useful for increasing write availability. However, it also means that even when *w + r > n,* there is a possibility of reading stale data, as the latest value may have been temporarily written to some values outside of *n.*

Sloppy quorum is more of an assurance of durability, than an actual quorum.

##### Multi-datacenter operation [#](#multi-datacenter-operation)

For datastores like Cassandra and Voldermort which implement leaderless replication across multiple datacenters, the number of replicas *n* includes replicas in all datacenter.

Each write is also sent to all datacenters, but it only waits for acknowledgement from a quorum of nodes within its local datacenter so that it's not affected by delays and interruptions on the link between multiple datacenters.

#### Detecting Concurrent Writes [#](#detecting-concurrent-writes)

In dynamo-style databases, several clients can concurrently write to the same key. When this happens, we have a conflict. We've briefly touched on conflict resolution techniques already, but we'll discuss them in more detail.

##### Last write wins (discarding concurrent writes) [#](#last-write-wins-%28discarding-concurrent-writes%29)

One approach for conflict resolution is the last write wins approach. It involves forcing an arbitrary ordering on concurrent writes (could be by using timestamps), picking the most "recent" value, and discarding writes with an earlier timestamp.

This helps to achieve the goal of eventual convergence across the data in replicas, at the cost of durability. If there were several concurrent writes the same key, only one of the writes will survive and the others will be discarded, even if all the writes were reported as successful.

Last write wins (LWW) is the only conflict resolution method supported by Apache Cassandra.

If losing data is not acceptable, LWW is not a good choice for conflict resolution.

##### The "happens-before" relationship and concurrency [#](#the-%22happens-before%22-relationship-and-concurrency)

Whenever we have two operations A and B, there are three possibilities:

- Either A happened before B
- Or B happened before A
- Or A and B are concurrent.

We say that an operation A happened before operation B if either of the following applies:

- B knows about A
- B depends on A
- B builds upon A

Thus, if we cannot capture this relationship between A and B, we say that they are concurrent. If they are concurrent, we have a conflict that needs to be resolved.

**Note:** Exact time does not matter for defining concurrency, two operations are concurrent if they are both unaware of each other, regardless of the physical time which they occurred. Two operations can happen sometime apart and still be concurrent, as long as they are unaware of each other.

##### Capturing the happens-before relationship [#](#capturing-the-happens-before-relationship)

In a single database replica, version numbers are used to determine concurrency.

It works like this:

- Each key is assigned a version number, and that version number is *incremented* every time that key is written, and the database stores the version number along with the value written. That version number is returned to a client.
- A client must read a key before writing. *When it reads a key, the server returns the latest version number together with the values that have not been overwritten.*
- When a client wants to write a new value, it returns the last version number it received in the prior step alongside the write.
- If the version number being passed with a write is higher than the version number of other values in the db, it means the new write is aware of those values at the time of the write (since it was returned from the prior read), and can overwrite all values with that version number or below.
- If there are higher version numbers, the database must keep all values with the higher version number (because those values are concurrent with the incoming write- it did not know about them).

Example scenario:

If two clients are trying to write a value for the same key at the same time, both would first read the data for that key and get the latest version number of say: 3. If one of them writes first, the version number will be updated to 4 from the database end. However, since the slower one will pass a version number of 3, it means it is concurrent with the other one since it's not aware of the higher version number of 4.

When a write includes the version number from a prior read, that tells us which previous state the write is based on.

##### Merging Concurrently Written Values [#](#merging-concurrently-written-values)

With the algorithm described above, clients have to do the work of merging concurrently written values. Riak calls these values *siblings.*

A simple merging approach is to take a union of the values. However, this can be faulty if one operation deleted a value but that value is still present in a sibling. To prevent this problem, the system must leave a marker *(tombstone)* to indicate that an item has been removed when merging siblings.

CRDTs are data structures that can automatically merge siblings in sensible ways, including preserving deletions.

##### Version Vectors [#](#version-vectors)

The algorithm described above used only a single replica. When we have multiple replicas, we use a version number *per replica* **and** *per key* and follow the same algorithm. Note that each replica also keeps track of the version numbers seen from each of the other replicas. With this information, we know which values to overwrite and which values to keep as siblings.

The collection of version numbers from all the replicas is called a *version vector.* Dotted version vectors are a nice variant of this used in Riak: [https://riak.com/posts/technical/vector-clocks-revisited-part-2-dotted-version-vectors/](https://riak.com/posts/technical/vector-clocks-revisited-part-2-dotted-version-vectors/)

Version vectors are also sent to clients when values are read, and need to be sent back to the database when a value is written.

*Version vectors enable us to distinguish between overwrites and concurrent writes.*

We also have Vector clocks, which are different from Version Vectors apparently: [https://haslab.wordpress.com/2011/07/08/version-vectors-are-not-vector-clocks/](https://haslab.wordpress.com/2011/07/08/version-vectors-are-not-vector-clocks/)

[distributed-systems](https://timilearning.com/tags/distributed-systems/) [learning-diary](https://timilearning.com/tags/learning-diary/) [ddia](https://timilearning.com/tags/ddia/)

To get notified when I write something new, you can [subscribe](https://feeds.feedburner.com/timilearning) to the RSS feed.

[← Home](https://timilearning.com/)