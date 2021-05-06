# Chapter 8 - The Trouble with Distributed Systems

*26 Dec 2019*

Notes from Chapter 8 of Martin Kleppmann's 'Designing Data-Intensive Applications' book

* * *

In this chapter, we'll look at the things that may go wrong in distributed systems. We'll cover problems with network, clocks and timing issues, and other faults.

**Table of Contents**

- [Faults and Partial Failures](#faults-and-partial-failures)
    - [Cloud Computing and Supercomputing](#cloud-computing-and-supercomputing)
- [Unreliable Networks](#unreliable-networks)
    - [Network Faults in Practice](#network-faults-in-practice)
    - [Detecting Faults](#detecting-faults)
    - [Timeouts and Unbounded Delays](#timeouts-and-unbounded-delays)
        - [Network Congestion and Queueing](#network-congestion-and-queueing)
        - [*Aside:* TCP vs UDP(Transmission Control Protocol vs User-Datagram Protocol)](#aside-tcp-vs-udptransmission-control-protocol-vs-user-datagram-protocol)
    - [Synchronous Versus Asynchronous Networks](#synchronous-versus-asynchronous-networks)
- [Unreliable Clocks](#unreliable-clocks)
    - [Monotonic vs Time-of-Day Clocks](#monotonic-vs-time-of-day-clocks)
        - [Time-of-day clocks](#time-of-day-clocks)
        - [Monotonic Clocks](#monotonic-clocks)
    - [Clock Synchronization and Accuracy](#clock-synchronization-and-accuracy)
    - [Relying on Synchronized Clocks](#relying-on-synchronized-clocks)
        - [Timestamps for ordering events.](#timestamps-for-ordering-events)
        - [Clock readings have a confidence interval](#clock-readings-have-a-confidence-interval)
        - [Synchronized clocks for global snapshots](#synchronized-clocks-for-global-snapshots)
    - [Process Pauses](#process-pauses)
- [Knowledge, Truth, and Lies](#knowledge-truth-and-lies)
    - [The Truth is Defined by the Majority](#the-truth-is-defined-by-the-majority)
        - [The Leader and the Lock](#the-leader-and-the-lock)
        - [Fencing tokens](#fencing-tokens)
    - [Byzantine Faults](#byzantine-faults)
        - [Weak forms of lying](#weak-forms-of-lying)
    - [System Model and Reality](#system-model-and-reality)
        - [System Model for Timing Assumptions](#system-model-for-timing-assumptions)
        - [System Model for Node Failures](#system-model-for-node-failures)
        - [Correctness of an algorithm](#correctness-of-an-algorithm)
        - [Safety and liveness](#safety-and-liveness)
        - [Mapping system models to the real world](#mapping-system-models-to-the-real-world)

### Faults and Partial Failures [#](#faults-and-partial-failures)

Distributed systems differ from single node computers because: unlike single node computers where either the system is completely working or completely broken, we can have partial failures in distributed systems.

What makes partial failures more difficult to deal with is that they are *nondeterministic.* It may sometimes work, and sometimes fail.

#### Cloud Computing and Supercomputing [#](#cloud-computing-and-supercomputing)

We have high-performance computing (HPC) and cloud computing on both extremes of philosophies for building large-scale applications.

High performance computers or Supercomputers have thousands of CPUs used for computationally expensive tasks like weather forecasting. In general, a job will checkpoint the state of its computation and store it durably from time to time. If a node fails, the whole cluster is brought down. The state of computation is restarted from the last checkpoint. This makes supercomputers similar to single node computers.

Nowadays, many internet services need high availability. It's not acceptable to bring down the cluster due to failure in a node.

*To make distributed systems work, we must endeavor to build a reliable system from unreliable components.*

We sometimes like to assume that faults are rare and then don't account for them, but we must design with fault tolerance in mind.

Building a reliable system from unreliable components is not a unique idea to distributed systems, and is used in other areas as well. E.g.:

- In information theory, the idea is to have reliable communication over an unreliable channel. We achieve this using error correcting codes.
- In networking, IP is unreliable as it may drop, delay, duplicate, or reorder packets. TCP provides a more reliable layer on top of that as it re-transmits missing packets, eliminates duplicates, and reassembles packets in the right order.

Note that even though a system can be more reliable than the parts that it's made of, there's a limit to the level of reliability that can be attained. Error-correcting codes can only deal with a number of single-bit errors and TCP cannot remove delays in the network.

### Unreliable Networks [#](#unreliable-networks)

As stated earlier, this book focuses on shared-nothing systems which communicate with each other via a network. The advantage of this approach is that it is comparatively cheap, as it requires no special hardware. We can have a bunch of regular machines as part of the system.

Note that the internet and most internal networks in datacenters are *asynchronous packet networks.* This means that: one node can send a message to another node, but have no guarantee about when the message will arrive, or whether the message will actually arrive at all. Unfortunately, with this approach, many things could go wrong:

- The request may have been lost. E.g. The network cable might have been unplugged.
- The remote node may have successfully processed the request, but then the response got lost on the network. E.g. a misconfigured network switch.
- The remote node may have failed.
- The request may be waiting in a queue to be delivered later. E.g if the network or recipient is overloaded.
- The response from the remote node may have been delayed or will be delivered later.

In essence, the sender is unable to tell whether the packet was delivered unless it receives a response message from the recipient. It's impossible to distinguish these issues in an asynchronous network.

These issues are typically handled with a *timeout*, but that still gives no information about the state of the request.

#### Network Faults in Practice [#](#network-faults-in-practice)

Although network faults may be rare, the fact that they can happen means that software needs to be able to handle them. Handling network faults does not necessarily mean *tolerating* them, a simple approach can just be to show an error message to users. However, there has to be work done to know how the software reacts to network problems, and also ensure that the system can recover from them.

It might be a good idea to deliberately trigger network problems and test the system's response - Chaos Monkey.

#### Detecting Faults [#](#detecting-faults)

It's important to automatically detect network faults, as they might be linked to faulty nodes. Detecting faults quickly ensures that:

- A load balancer can stop sending requests to a dead node.
- A new follower can be promoted to a leader if the leader fails in a single-leader replication.

Due to the uncertainty about the network, it's difficult to tell whether a node is working or not. There are some specific ways to tell though, such as:

- If the machine on which the node is running is reachable, but no process is listening on the destination port (e.g. because the process crashed), the OS will close or refuse TCP connections. However, if the node crashed while processing the request, there's no way of knowing how much data was processed by the remote node.
- If you have access to the management interface of the network switches in your datacenter, they can be queried to detect link failures at hardware level. Of course this is not applicable if you're connecting over the internet or have no access to the datacenter.
- In a situation where a process on a node is crashed, but the node's OS is still running, there can be a script which notifies other nodes when a node has crashed. This will allow another node to take over. This approach is used in Hbase.

#### Timeouts and Unbounded Delays [#](#timeouts-and-unbounded-delays)

We have mentioned that timeouts are often used to detect a fault. However, there is no simple answer too how long a timeout should be. It simply depends.

With a long timeout, it means there can be a wait until a node is declared dead. This means users will have to wait a while or see error messages.

On the other hand, a short timeout means that nodes can be declared dead prematurely, even when it only suffers a temporary breakdown (e.g. due to a load spike on the node or the network). There are downsides of declaring a node dead prematurely:

- Its responsibilities need to be transferred to other nodes, which can place additional load on the other nodes and the network. This can lead to a cascading failure as other nodes can become slow to respond.
- If the node is in the middle of performing an action and another node takes over, the action may be performed twice.

In an ideal system, we could have a guarantee that the maximum delay for packets transmission will be *d*, and the node always handles a response within time *r*. In this kind of system, we could set a timeout for 2d + r and it'll be reasonable.

However, in most systems, we do not have either of those guarantees. Asynchronous networks have *unbounded delays.*

##### Network Congestion and Queueing [#](#network-congestion-and-queueing)

Queueing is the most common cause of the variability of network packet delays (i.e. unbounded delays). Queues can be formed at different points:

- If several nodes try to send packets to the same destination, the packets must be queued up by the network switch and fed into the destination network link one by one. On a busy network link, a packet may have to wait a while before it can get a slot. A packet can even get dropped if the switch queue fills up, and it needs to be resent. This can happen even if the network is functioning properly.
- When a packet reaches its destination node, if all the CPU cores are currently too busy to handle the request, the request needs to be queued by the operating system until it can handle it.
- TCP performs *flow control*, where a node limits its rate of sending to avoid overloading a network link or the receiving node. This means additional queueing at the sender even before the data enters the network.

##### *Aside:* TCP vs UDP(Transmission Control Protocol vs User-Datagram Protocol) [#](#aside%3A-tcp-vs-udp%28transmission-control-protocol-vs-user-datagram-protocol%29)

TCP is a reliable network transmission protocol while UDP is unreliable. It means that TCP implements:

- Flow Control
- Acknowledgement and Retransmission
- Sequencing: Ensuring that messages arrive in the right order even if packets are dropped.

Any messages not acknowledged will be retransmitted in TCP.

UDP is used in latency-sensitive applications like videoconferencing and Voice over IP, where there's less tolerance for delays. In UDP, delayed data is probably worthless so it does not try to retransmit it. E.g. in phone calls, instead of retransmitting, it simply fills the missing packet's time slot with silence. The retry happens at the human layer: "Could you please repeat that please?".

In essence, timeouts should typically be chosen experimentally: measure the distribution of network round trip times over an extended period, and over many machines to determine the expected variability of delays.

The *Phi Accrual failure detector* used in Akka and Cassandra measure response times and automatically adjust timeouts based on the observed response time distribution.

#### Synchronous Versus Asynchronous Networks [#](#synchronous-versus-asynchronous-networks)

A question that you might have is: *why don't we make the network reliable at a hardware level so the software does not need to worry about it?*

To address this, it's worth looking at the traditional fixed-line telephone network (non-cellular, non-VoIP) which is apparently very reliable and rarely drops messages.

The way it works is that:

- When a call is made over the network, it creates a *circuit.*
- This circuit has a fixed, guaranteed bandwidth for the call which remains in place until the call ends.
- This network is synchronous, and it does not suffer from queueing, since the required amount of space for the call has already been reserved. Because there is no queueing, it has a *bounded delay.*

Note that this approach differs from a TCP connection. While there is a fixed amount of reserved bandwidth here that no one else can use while the circuit is established, TCP packets will typically grab whatever network bandwidth is available.

Datacenter networks and the internet make use of the TCP approach of packet switching rather than establishing circuits, because they are optimizing for *bursty traffic.* With an audio or video call where the number of bits transferred per second is fairly constant, the traffic through the internet is unpredictable. We could be requesting a web page, or sending an email, or transferring a file etc. The goal is to just complete it as quickly as possible.

Using circuits for bursty data transfer will waste network capacity and could make transfers unnecessarily slow, as we would have to guess how much bandwidth to allocate beforehand. TCP dynamically adapts the data transfer rate to the available network capacity.

There's ongoing research to use *quality of service* and *admission control* to emulate circuit switching on packet networks, or provide statistically bounded delays.

### Unreliable Clocks [#](#unreliable-clocks)

Clocks and time are important in distributed systems. Applications use clocks to answer questions like:

- Has a request timed out yet?
- When was a request received?
- How long did a user spend on a site?
- When does a cache entry expire? Etc.

Some questions measure *duration*, while some describe *points in time.*

Time is tricky because each machine on a network has its own clock, and some may be faster or slower than others. Clocks can be synchronized to a degree though, by using the Network Time Protocol (NTP). It works by adjusting clocks using time reported from a group of servers. The group of servers get their time from a GPS receiver.

#### Monotonic vs Time-of-Day Clocks [#](#monotonic-vs-time-of-day-clocks)

Modern computers have at least two different kinds of clocks: a *time-of-day clock* and a *monotonic clock.*

##### Time-of-day clocks [#](#time-of-day-clocks)

These are like standard clocks, which just return the current date and time according to a calendar. These clocks are typically synchronized with NTP, which means that timestamps should match across machine ideally.

Note that if the local clock is too far ahead of NTP, it may appear to jump back to a previous point in time. It could also jump due to leap seconds. The tendency of these clocks to jump make them unsuitable for measuring elapsed time.

##### Monotonic Clocks [#](#monotonic-clocks)

These clocks are suitable for measuring a duration like a timeout or response time. They are guaranteed to move forward in time (unlike a time-of-day clock which may jump back in time).

System.nanoTime() in Java is a monotonic clock. With a monotonic clock, you can check the value at a point in time, perform an action, then check the value again and then use the difference between the two values to measure time elapsed between the two checks.

Monotonic clocks are fine for measuring elapsed time, because they do not assume any synchronization between nodes' clocks.

#### Clock Synchronization and Accuracy [#](#clock-synchronization-and-accuracy)

Unlike monotonic clocks which don't need synchronization, time-of-day clocks must be synchronized with an NTP server or another external time source. However, NTP and hard clocks are not as reliable or accurate as one might hope. For example:

- The quartz clock in a computer isn't very accurate. It may go faster or slower than it should. It can vary depending on the temperature of the machine. Basically, it *drifts.*
- If the computer's clock drifts too much from an NTP server, it may refuse to synchronize, or the computer's clock will be forcibly reset. Any application that's observing this app may see time go backward or jump forward.
- NTP synchronization is only as good as the network delay, so there's a limit to accuracy on a congested network with variable packet delays. An experiment ( [https://iopscience.iop.org/0143-0807/23/4/103/](https://iopscience.iop.org/0143-0807/23/4/103/)) showed that the only a minimum error of 35ms is achievable when synchronizing over the internet, though the error can be up to a second when we have occasional spikes in network delay.
- Leap seconds will lead to a minute that is 59 seconds or 61 seconds long. If a system isn't designed to handle leap seconds, this can lead to a crash. ([http://www.somebits.com/weblog/tech/bad/leap-second-2012.html](http://www.somebits.com/weblog/tech/bad/leap-second-2012.html))
- The hardware clock is virtualized in virtual machines. When multiple virtual machines share a CPU core, each VM is paused for tens of milliseconds while another VM is running. For an application running on a VM, it can look like the clock suddenly jumped forward.

Nevertheless, it's possible to achieve very good clock accuracy with significant investment into resources. For example, the MiFID II European regulation for financial companies mandates that HFT funds synchronize their clocks within 100 microseconds of UTC, to help debug market anomalies like "flash crashes".

#### Relying on Synchronized Clocks [#](#relying-on-synchronized-clocks)

While clocks may seem simple and straightforward, they have a good number of pitfalls. Some of the issues that may arise are:

- Time of day clocks can move backward in time
- A day may not exactly have 86400 seconds
- The time on a node may differ from the time on another.

Like with unreliable networks, robust software must be prepared to deal with incorrect clocks. Dealing with incorrect clocks can be even trickier because the problems caused by this can easily go unnoticed. A faulty CPU or misconfigured network is easier to detect, as the system would not work at all. However, for a defective clock, things will generally look fine. We're more likely to experience silent and subtle data loss than a dramatic crash.

Therefore, if a software requires synchronized clocks, it's essential to monitor the clock offsets between all machines. A node whose clock drifts too far from the others should be labelled as a dead node and removed from the cluster.

##### Timestamps for ordering events. [#](#timestamps-for-ordering-events.)

Time-of-day clocks are commonly used for ordering events in some systems and they often use the *last write wins* conflict resolution strategy. Some of these systems are Cassandra and Riak, typically multi-leader replication and leaderless databases. Some implementations of this generate the timestamp on the client's side rather than on the server, but this does not change the problems of LWW which include:

- Writes can mysteriously disappear.
- It's impossible to distinguish between concurrent writes and causal writes (where one write depends on another)
- Two nodes can independently generate writes with the same timestamp.

*Q*: Could NTP synchronization be made accurate enough that such incorrect orderings cannot occur?

*A*: Probably not. NTP's synchronization accuracy is also limited by the network round-trip time, in addition to other error sources like quartz drift.

*Logical clocks* are a safer alternative for ordering events than an oscillating quartz crystal. They measure the relative ordering of events, rather than actual elapsed time which *physical clocks* (like time-of-day and monotonic clocks).

##### Clock readings have a confidence interval [#](#clock-readings-have-a-confidence-interval)

Clock readings typically have an uncertainty range, like a margin of error. However, most systems don't expose this uncertainty. An exception to this is Google's TrueTime API which is used in Spanner, and gives a confidence interval on the local clock.

##### Synchronized clocks for global snapshots [#](#synchronized-clocks-for-global-snapshots)

Snapshot isolation is commonly implemented by giving each transaction a montonically increasing ID. If a write happened later than the snapshot (i.e. it has a transaction ID greater than the snapshot), the write is invisible to the snapshot transaction. This is easier to manage on a single-node database, as we can use a simple counter.

For a distributed database though, it is more difficult to coordinate a monotonically increasing transaction ID. The transaction ID must reflect causality. If transaction B reads a value written by transaction A, B must have a higher transaction ID than A for it to be consistent.

If we didn’t have uncertainty about clock accuracy, the timestamps from the synchronized time-of-day clocks would be suitable as transaction IDs as later transactions will have a higher timestamp.

However, Google's Spanner implements snapshot isolation across datacenters this way:

*Spanner implements snapshot isolation across datacenters in this way. It uses the clock’s confidence interval as reported by the TrueTime API, and is based on the following observation: if you have two confidence intervals, each consisting of an earliest and latest possible timestamp (A = \[Aearliest, Alatest\] and B = \[Bearliest, Blatest\]), and those two intervals do not overlap (i.e., Aearliest < Alatest < Bearliest < Blatest), then B definitely happened after A — there can be no doubt. Only if the intervals overlap are we unsure in which order A and B happened.*

Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 7547-7554). O'Reilly Media. Kindle Edition.

To ensure that transaction timestamps reflect causality, Spanner waits for the length of the confidence interval before committing a read-write transaction. This means that any transaction that reads the data is at a sufficiently later time, so the confidence intervals do not overlap. For example, if the confidence interval is 7ms, a read-write transaction will wait for 7ms before committing. Remember that with snapshot isolation, a transaction can't read anything that wasn't committed when it started. Therefore, we can be sure that any transaction that reads the now committed read-write transaction happened at a sufficiently later time.

To keep the wait time as small as possible, Google uses a GPS receiver in each datacenter, which allows clocks to be synchronized within about 7ms.

#### Process Pauses [#](#process-pauses)

A node in a distributed system must assume that its execution can be paused for a significant amount of time at any point, even in the middle of a function. When this pause happens, the rest of the system keeps moving and may declare the paused node dead because it's not responding. This paused node may eventually continue running, without noticing that it was asleep until it checks the clock later.

A distributed system must tailor for these pauses which can be caused by:

- Garbage collectors which stop all running threads. *I've experienced this in an Elasticsearch cluster.*
- In virtualized environments, a VM can be suspended and resumed e.g. for live migration of a VM from one host to another without a reboot. Suspending the VM means pausing the execution of all processes and saving memory contents to disk. Resuming it means restoring the memory contents and continuing execution.
- On laptops, the execution of a process could be paused and resumed arbitrarily e.g. when a user closes their laptop lid.
- IO operations could also lead to delays.

There's active research into limiting the impact of Garbage Collection pauses. Some of the options are:

- Treat GC pauses like brief planned node outages. When a node requires a GC pause, the runtime can warn the application and stop sending requests to that node. It could also wait for it to process outstanding requests and perform GC while no requests are in progress. This hides the GC pauses from clients and reduces high percentiles of response times. This approach is used in some latency-sensitive financial trading systems ([https://cdn2.hubspot.net/hubfs/1624455/Website_2016/Content/Whitepapers/Cinnober%20on%20GC%20pause%20free%20Java%20applications.pdf](https://cdn2.hubspot.net/hubfs/1624455/Website_2016/Content/Whitepapers/Cinnober%20on%20GC%20pause%20free%20Java%20applications.pdf))
- Use the GC only for short-lived objects that are fast to collect, and restart processes periodically, before they accumulate enough long-lived objects to requires a full GC of long-lived objects. *A node can be restarted at a time, and traffic can be shifted away from the node before the planned restart, like in a rolling upgrade.*

These options can't fully prevent GC pauses, but they can reduce their impact on the application.

### Knowledge, Truth, and Lies [#](#knowledge%2C-truth%2C-and-lies)

So far, we have discussed some of the distributed systems problems that can occur, which include: unreliable networks, unreliable clocks, faulty nodes, processing pauses etc. We've also discussed how distributed systems differ from programs running on a single node: there's no shared memory, there's only message passing via an unreliable network with variable delays.

As a result of these issues, a node in a distributed system cannot *know* anything for sure. It can only guess based on the messages it receives (or doesn't receive) via the network. There has to be a consensus.

In this section, we'll explore the concept of knowledge and truth, and guarantees we can provide under certain assumptions in a distributed system.

#### The Truth is Defined by the Majority [#](#the-truth-is-defined-by-the-majority)

A node cannot trust its assessment of a situation. A node may think it's the leader, while the other nodes have elected a new one; it may think it's alive, while other nodes have declared it dead. As a result, many distributed algorithms rely on a *quorum* for making decisions i.e. decisions require a minimum number of votes from several nodes in order to reduce dependence on a single node.

The quorum is typically an absolute majority of more than half the nodes. This is typically safe because there can only be one majority in a system at a time.

##### The Leader and the Lock [#](#the-leader-and-the-lock)

A system often requires there to only be one of a particular thing. For example:

- Only one leader for a database partition.
- Only one transaction is allowed to hold the lock for a particular resource or object.
- Only one user is allowed to register a particular username.

Due to the fact that a node can believe it’s the "chosen one" even when it isn't, the system must be designed to handle such situations and avoid problems like split-brain.

##### Fencing tokens [#](#fencing-tokens)

One of the ways by which systems handle a situation where a node is under a false belief of being "the chosen one", thereby disrupting the rest of the system, is by using fencing tokens.

Basically, each time a lock server grants a lock or a lease, it also generates a fencing token (a number that increases every time a lock is granted). We can then require that any client which wants to send a write request to the storage service must include the current fencing token.

The lock server will then perform validation on any request with the fencing token included and reject it if it has generated a fencing token with a higher number.

For applications using ZooKeeper as a lock service, the transaction ID zxid or node version cversion can be used as the fencing token, since they are guaranteed to be monotonically increasing - which is a required property for a fencing token.

#### Byzantine Faults [#](#byzantine-faults)

Fencing tokens can help detect and block a node that is not deliberately acting in error (e.g. because it hasn't yet realized that its lease has expired). However, for a node that is deliberately acting in error, it could simply send messages with a fake fencing token.

In this book, nodes are assumed to be unreliable but honest: any node that *does* respond is assumed to be telling the truth to the bests of its knowledge.

If there's a risk that nodes may "lie" (e.g. by sending corrupted messages or faulty responses), it becomes a much harder problem to deal with. That behavior is known as a *Byzantine fault* and systems that are designed to handle these faults are *Byzantine Fault Tolerant Systems.*

*A system is Byzantine fault-tolerant if it continues to operate correctly even if some of the nodes are malfunctioning and not obeying the protocol, or if malicious attackers are interfering with the network.*

Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 7812-7813). O'Reilly Media. Kindle Edition.

Dealing with Byzantine faults is relevant in specific circumstances like:

- In an aerospace environment, data in a computer's memory may become corrupted due to radiation, leading it to respond to other nodes in unpredictable ways. The system has to be equipped to handle this to prevent plane crashes. Therefore, flight control systems must tolerate Byzantine faults.
- In a system with multiple participating organizations (e.g peer-to-peer networks like Bitcoin), some participants may attempt to cheat or defraud others. It's not safe to simply trust another node's messages.

In most server-side data systems, however, the cost of deploying Byzantine fault-tolerant solutions makes them not practical. Web applications need some other controls though to prevent malicious behavior and that's why: input validation, sanitization and output escaping are important.

##### Weak forms of lying [#](#weak-forms-of-lying)

There are weaker forms of "lying" which are not full-blown Byzantine faults that we can protect against. For example:

- Network packets get corrupted due to hardware issues or bugs in the OS. These are usually caught by the checksums built into TCP and UDP, but they sometimes evade detection. Checksums in the application-level protocol is a simple measure that can provide protection against such corruption.
- A publicly accessible application must carefully sanitize all inputs from users.
- NTP clients can be synchronized with multiple server addresses, instead of just one server. This makes NTP more robust. One erroneous server among multiple good ones will have minimal impact compared to if it was the only server.

#### System Model and Reality [#](#system-model-and-reality)

When coming up with distributed systems algorithms, we need to write them in a way that doesn't depend too much on the hardware and software details. Basically, we need an abstraction for what the algorithm may assume, and the types of faults that we can expect in a system. This abstraction is known as a *system model.*

##### System Model for Timing Assumptions [#](#system-model-for-timing-assumptions)

- *Synchronous model:* In this model, we assume that there's a bounded network delay, bounded process pause, and bounded clock error. That is, although there might be errors or delays, it will never exceed a fixed upper bound.
- *Partially synchronous model:* This model assumes that the system behaves like a synchronous model *most of the time,* but sometimes exceeds the bounds for network delay, process pauses and pause drifts. This is the **most realistic** model for timing assumptions, since many systems work correctly most of the time but occasionally exceed the upper bound.
- *Asynchronous model:* Here, the system is not allowed to make any timing assumptions. It does not have a clock, and so it doesn't use timeouts. This model is

##### System Model for Node Failures [#](#system-model-for-node-failures)

- *Crash-stop faults:* This model assumes that a node can fail only by crashing, and the node never comes back. That is, once it stops responding, its gone forever.
- *Crash-recovery faults:* This model assumes that nodes may crash at any moment, but also start responding again after some unknown time. Nodes here are assumed to have stable storage that gets preserved across crashes, and the in-memory state is assumed to be lost.
- *Byzantine faults:* Nodes may do anything, including trying to trick other nodes.

For modeling real system, the most useful model is generally the partially synchronous model with crash-recovery faults.

##### Correctness of an algorithm [#](#correctness-of-an-algorithm)

For an algorithm to be *correct,* it must have certain *properties.* E.g. For a sorting algorithm to be correct, it must have certain properties expected of its output. Such as that the element further to the left is smaller than the element further to the right.

Likewise, we can define properties for what it means for a distributed algorithm to be correct. For example, for generating fencing tokens, the algorithm may be required to satisfy the following properties:

- *Uniqueness:* No two requests for a fencing token must return the same value.
- *Monotonic sequence:* It must always increase.
- *Availability:* A node that requests a fencing token and doesn't crash must eventually receive a response.

It's possible for some properties to hold, while others don't. How do we know which distinguish between which properties must hold and which could tolerate caveats? The next section helps to answer that.

##### Safety and liveness [#](#safety-and-liveness)

There are two different kinds of properties that we can distinguish between: *safety* and *liveness* properties. In the example above for a fencing token, *uniqueness* and *monotonic sequence* are safety properties, while *availability* is a liveness property.

Safety properties are informally defined as: *nothing bad happens*, while liveness properties are defined as *something good eventually happens.*

These informal definitions are subjective (what's good or bad, really) and it’s best not to read too much into them.

The actual definitions of safety and liveness are precise and mathematical:

- *If a safety property is violated, we can point at a particular point in time at which it was broken (for example, if the uniqueness property was violated, we can identify the particular operation in which a duplicate fencing token was returned). After a safety property has been violated, the violation cannot be undone — the damage is already done.*
- *A liveness property works the other way round: it may not hold at some point in time (for example, a node may have sent a request but not yet received a response), but there is always hope that it may be satisfied in the future (namely by receiving a response).*

Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 7924-7927). O'Reilly Media. Kindle Edition.

For distributed algorithms, it is commonly required that safety properties *always* hold, in all possible situations of a system model. Therefore, even in the occurrence of all nodes crashing, or entire network failures, the algorithm must always return a correct result.

On the other hand, we are allowed to make caveats with liveness properties. E.g. we could say that a request will only receive a response if majority of nodes have not crashed, and only if the network recovers from an outage eventually.

##### Mapping system models to the real world [#](#mapping-system-models-to-the-real-world)

While these theoretical abstractions are useful, reality must also be considered when designing algorithms. We may sometimes have to include code to handle situations where something that was assumed to be impossible actually happens.

Proving the correctness of an algorithm does not mean that the implementation on a real system will always behave correctly. However, theoretical analysis is still a good first step because it can uncover problems that may remain hidden for a long time.

Theoretical analysis and empirical testing are equally important.

[learning-diary](https://timilearning.com/tags/learning-diary/) [distributed-systems](https://timilearning.com/tags/distributed-systems/) [ddia](https://timilearning.com/tags/ddia/)

To get notified when I write something new, you can [subscribe](https://feeds.feedburner.com/timilearning) to the RSS feed.

[← Home](https://timilearning.com/)