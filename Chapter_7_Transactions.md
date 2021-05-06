# Chapter 7 - Transactions - A blog by Timi Adeniran

*15 Dec 2019*

My notes from Chapter 7 of 'Designing Data-Intensive Applications' by Martin Kleppmann.

**Table of Contents**

- [The Meaning of ACID](#the-meaning-of-acid)
- [Single-Object and Multi-Object Operations](#single-object-and-multi-object-operations)
- [Weak Isolation Levels](#weak-isolation-levels)
    - [Read Committed](#read-committed)
        - [Dirty Reads](#dirty-reads)
        - [Dirty Writes](#dirty-writes)
        - [Implementing read committed](#implementing-read-committed)
    - [Snapshot Isolation and Repeatable Read](#snapshot-isolation-and-repeatable-read)
        - [Implementing snapshot isolation](#implementing-snapshot-isolation)
        - [Indexes and snapshot isolation](#indexes-and-snapshot-isolation)
        - [Repeatable read and naming confusion](#repeatable-read-and-naming-confusion)
    - [Preventing Lost Updates](#preventing-lost-updates)
        - [Automatically detecting lost updates](#automatically-detecting-lost-updates)
        - [Compare-and-set](#compare-and-set)
        - [Conflict resolution and replication](#conflict-resolution-and-replication)
    - [Write Skew and Phantoms](#write-skew-and-phantoms)
        - [Materializing Conflicts](#materializing-conflicts)
- [Serializability](#serializability)
    - [Actual Serial Execution](#actual-serial-execution)
        - [Encapsulating transactions in stored procedures](#encapsulating-transactions-in-stored-procedures)
        - [Partitioning](#partitioning)
        - [Summary of serial execution](#summary-of-serial-execution)
    - [Two-Phase Locking (2PL)](#two-phase-locking-2pl)
        - [Implementation of two-phase locking](#implementation-of-two-phase-locking)
        - [Performance of two-phase locking](#performance-of-two-phase-locking)
        - [Predicate Locks](#predicate-locks)
        - [Index Range Locks](#index-range-locks)
    - [Serializable Snapshot Isolation](#serializable-snapshot-isolation)
        - [Decisions based on an outdated premise](#decisions-based-on-an-outdated-premise)
        - [Performance of serializable snapshot isolation](#performance-of-serializable-snapshot-isolation)

* * *

### Introduction 

Transactions were created to simplify the programming model for applications accessing a database.

All the reads and writes in a transaction are executed as one operation: either the entire operation succeeds (*commit*) or it fails (*abort*, *rollback*).

NoSQL databases started gaining popularity in the late 2000s and aimed to improve the status quo of relational databases by offering new data models, and including replication and partitioning by default. However, many of these new models didn't implement transactions, or watered down the meaning of the word to describe a weaker set of guarantees than had previously been understood.

As these distributed databases started to emerge, the belief that transactions oppose scalability became popular; that a system would have to abandon transactions in order to maintain good performance and high availability. This is not true though.

Like every technical design choice, there are advantages and disadvantages of using transactions. It's a tradeoff.

### The Meaning of ACID [#](#the-meaning-of-acid)

**ACID** stands for *Atomicity, Consistency, Isolation,* and *Durability.* It is often used to describe the safety guarantees provided by transactions.

However, different databases have different implementations of ACID, and there are ambiguous definitions of a term like Isolation. ACID has essentially become a marketing term now.

Systems that don't meet the ACID criteria are sometimes called *BASE: Basically Available, Soft state,* and *Eventual Consistency,* which can mean almost anything you want.

----

**Atomicity**

The word *atomic* in general means something that cannot be broken down into smaller parts.

In the context of a database transaction, atomicity refers to the ability to abort a transaction on error and have all the writes from the transaction discarded.

----


**Consistency**

Consistency in the context of ACID is an application-level constraint. It means that there are certain statements about your data (*invariants*) that must always be true.

It is up to the application to define what those invariants are so the transaction can preserve them correctly.

Unlike the other terms, consistency is a property of the application not the database, but transactions enforce those rules. These invariants are typically enforced through database constraints like uniqueness.

----

**Isolation**

Isolation is the property of ensuring that concurrently executing transactions are isolated from each other i.e. they don't step on each other's toes. They pretend like they don't know about each other.

Isolation ensures that when concurrently executing transactions are committed, the result is the same as if they had run *serially,* even though they may have run concurrently.

In practice though, serializable isolation is rarely used because of the performance penalty it carries.

----

**Durability**

Durability is the promise that when a transaction is committed successfully, any data that has been written will not be forgotten, even in the event of a hardware fault or database crashes.

----

In single node databases, durability means that the data has been written to nonvolatile storage like a hard drive or SSD. It also usually involves a write-ahead log or similar which helps with recovery in case the data structures on disk are corrupted.

In a replicated database, durability often means that data has been successfully copied to a number of nodes.

However, perfect durability does not exist. If all the hard disks and backups are destroyed at the same time, there's nothing the database can do to save you. In replicated systems for example, faults can be correlated (say a power outage or a bug that crashes every node that has a particular input) and can knock out all replicas are once.

### Single-Object and Multi-Object Operations [#](#single-object-and-multi-object-operations)

The definitions of *atomicity* and *isolation* so far assume that several objects (rows, documents, records) will be modified at once. These are known as *multi-object transactions* and are often needed if several pieces of data are to be kept in sync.

We need a way to determine which read and write operations belong to the same transaction.

In relational databases, this is done based on the client's TCP connection to the database server: on any particular connection, everything between BEGIN TRANSACTION and a COMMIT is considered to be part of the same transaction.

----

**Single-object writes**

Atomicity and isolation also apply when a single object is being changed. E.g.

- If a 20KB JSON document is being written to a database and the network connection is interrupted after the first 10KB have been sent, does the database store the 10KB fragment of JSON?
- If power fails while the database is in the middle of overwriting the previous value on disk, will we have the previous and new values spliced together?
- If another client reads a document while it's being updated, will it see a partially updated value.

These issues are why storage engines almost universally aim to provide atomicity and isolation on the level of a single object (such as a key-value pair) on one node.

Atomicity can be implemented by using a log for crash recovery, while isolation can be implemented using a lock on each object.

These single object operations are useful, but are not transactions in the typical sense of the word. A transaction is generally considered as a mechanism for grouping multiple operations on multiple objects into a single unit of execution.

----

**The need for multi-object transactions**

Some distributed datastores have abandoned multi-object transactions because they are difficult to implement across partitions, and can hinder performance when high availability/performance is required.

There are some use cases where multi-object operations need to be coordinated e.g.

- When we are adding new rows to a table which have references to a row in another table using foreign keys. The foreign keys have to be coordinated across the tables and must be correct and up to date.
- In a document data model, when denormalized information needs to be updated, several documents often need to be updated in one go.
- In databases with secondary indexes (i.e. almost everything except pure key-value stores), the indexes also need to be updated every time a value is changed. That is, the indexes needs to be updated with the new records.

These can be implemented without transactions, but error handling is more complex without atomicity and isolation (to prevent concurrency problems).

----

**Handling Errors And Aborts**

The idea of atomicity is to make it so that retrying failed transactions is safe. However, it's not so straightforward. There are a number of things to take into consideration:

- What if the transaction actually succeeded but the network failed while the server tried to acknowledge the successful commit to the client (so the client thinks it failed), retrying the transaction will cause it to be performed twice - unless there's a de-duplication mechanism in place.
- If the error is due to overload, retrying the transaction will only compound the problem.
- If a transaction has side effects outside of the database, those side effects may happen even if the transaction is aborted. E.g. Sending an email

### Weak Isolation Levels [#](#weak-isolation-levels)

Concurrency issues (e.g. race conditions) happen when one transaction reads data that is concurrently modified by another transaction, or when two transactions simultaneously modify the same data.

Concurrency issues are often hard to reproduce or test.

----

*Transaction Isolation* is the means by which databases typically try to hide concurrency issues from application developers.

----

*Serializable Isolation* is the ideal isolation required, as the database guarantees that transactions have the same effect as if they ran serially (one after another). However, this form has a performance cost which most databases don't want to pay. As a result, databases use weaker levels of isolation which prevent against some concurrency issues, but not all.

----

Concurrency bugs caused by weak transaction isolation are not just theoretical. They are real issues which have led to loss of money, data corruption, and even an investigation by financial auditors. (e.g. [https://bitcointalk.org/index.php?topic=499580](https://bitcointalk.org/index.php?topic=499580))

#### Read Committed [#](#read-committed)

The core characteristics of this isolation level are that it prevents *dirty reads* and *dirty writes.*

##### Dirty Reads [#](#dirty-reads)

If an object has been updated in a transaction but has not yet been committed, the act of any transaction being able to see that uncommitted data is known as a *dirty read* i.e. when reading data from the database, you will only see data that has been committed.

##### Dirty Writes [#](#dirty-writes)

If an object has been updated in a transaction but has not yet been committed, the act of any transaction being able to overwrite the uncommitted value is a *dirty write.* i.e. when writing to the database, you will only overwrite data that has been committed.

While this isolation level is commonly used, it does not prevent certain race conditions. For example:

- Imagine that I read a value as '30' and increment it in one transaction, and another transaction reads that value *before* the increment operation. If that new transaction also tries to increment it even after my transaction has committed, they would be incrementing it based on the earlier value seen as '30' before any locks were added when modifying the object. *This is because the isolation level is implemented for objects that have been modified.*

##### Implementing read committed [#](#implementing-read-committed)

This is the default setting in many databases like Oracle 11g, PostgreSQL, SQL Server and other databases.

Most databases prevent dirty-writes by using row-level locks i.e. when a transaction wants to modify an object, it must first acquire a lock on the object. It must then hold the lock until the transaction is committed or aborted. Only one transaction can hold a lock at a time.

Preventing dirty reads can also be implemented in a similar fashion. One can require that any transaction that wants to read an object should briefly acquire the lock and release it again after reading. This way, any write on an object that hasn't been committed cannot be read since the transaction that performed the write would still hold the lock.

This approach of requiring locks before reading is inefficient is practice because one long-running write transaction can force other read-only transactions to wait for a long time. Because of this, dirty reads are prevented by most databases using this approach: *for every object that is written, the database remembers both the old committed value and the new value set by the transaction which holds the write lock. Any transactions that want to read the object are simply given the old value until the new value is committed.*

#### Snapshot Isolation and Repeatable Read [#](#snapshot-isolation-and-repeatable-read)

With the read committed isolation level, there is still room for concurrency bugs. One of the anomalies that can happen is *a non-repeatable read* or a *read skew.*

A *read skew* means that you might read the value of an object in one transaction before a separate transaction begins, and when that separate transaction ends, that value has changed into a new value. This happens because the read committed isolation only applies a lock on values that are about to be modified.

Thus, a long running read-only transaction can have situations where the value of an object or multiple objects changes between when the transaction starts and when it ends, which can lead to inconsistencies.

----

Basically, an example flow for this kind of anomaly is this:

- Transaction A begins and reads objects from DB
- Transaction B begins and updates some of those objects (this can happen since Transaction A won't have a lock on those objects, as it's read-only).
- If Transaction A re-reads those objects within the same transaction for whatever reason, those values will have changed and the earlier read is *non-repeatable.*

"A Fuzzy or Non-Repeatable Read occurs when a value that has been read by a still in-flight transaction is overwritten by another transaction. Even without a second read of the value actually occurring this can still cause database invariants to be violated" - Source: [https://blog.acolyer.org/2016/02/24/a-critique-of-ansi-sql-isolation-levels/](https://blog.acolyer.org/2016/02/24/a-critique-of-ansi-sql-isolation-levels/)

----

Read skew is considered acceptable under read committed isolation, but some situations cannot tolerate that temporary inconsistency:

- **Backups:** A backup requires making a copy of a database which can take long hours. During this time, writes will be made to the database. It's possible that some parts of the backup will contain an older version of the data, and other parts will have a newer version. These inconsistencies will become permanent on the database level.
- **Analytic queries and integrity checks:** Long running analytics queries could end up returning incorrect data if the data in the db has changed over the course of the run.

----

To solve this problem, *Snapshot isolation* is commonly used. The main idea is that each transaction reads a *consistent snapshot* of the database - that is, *a transaction will only see all the data that was committed in the database at the start of the transaction.* Even if another transaction changes the data, it won't be seen by the current transaction.

This kind of isolation is especially beneficial for long-running, read only queries like backups and analytics, as the data on which they operate remains the same throughout the transaction.

##### Implementing snapshot isolation [#](#implementing-snapshot-isolation)

A core principle of snapshot isolation is this:

*Readers never block writers, and writers never block readers.*

----

Implementations of snapshot isolation typically use write locks to prevent dirty writes, but have an alternate mechanism for preventing dirty reads.

Write locks mean that a transaction that makes a write to an object can block the progress of another transaction that makes a write to the same object.

To implement snapshot isolation, databases potentially keep multiple different committed version of the same object. Due to the fact that it maintains several versions of an object side by side, the technique is known as *multi-version concurrency control (MVCC).*

----

For a database providing only read committed isolation, we would only need to keep two versions of an object: the committed version and the overwritten-but-uncommitted version. However, with snapshot isolation, we keep different versions of the same object. The scenario below explains why:

- If Transaction A has a snapshot of the database and Transaction B has the same snapshot of the database. If transaction A commits before Transaction B, the database still needs to keep track of the snapshot being used by Transaction B, and the new committed value of Transaction A.
- This can continue if there's a Transaction C, D, etc.

That's why we could have multiple versions of the same object.

Note that storage engines that support snapshot isolation typically use MVCC for their read committed isolation level as well.

----

*MVCC-based snapshot isolation is typically implemented* by given each transaction a unique, always-increasing transaction ID. Any writes to the database by a transaction are tagged with the transaction ID of the writer. Each row in the table is tagged with a created\_by and deleted\_by field which has the transaction ID that performed the creation or deletion (when applicable).

----

The transaction IDs are used as follows:

- At the start of each transaction, the database notes all the other transactions that are in progress (i.e. not committed or aborted yet). Any writes by the other transactions are ignored.
- Any writes made by transactions with a later transaction ID than the current one are ignored, regardless of whether they have committed or not.

This is of course more efficient than actually taking a snapshot of the database like with Elasticsearch.

##### Indexes and snapshot isolation [#](#indexes-and-snapshot-isolation)

One option with indexes and snapshot isolation is to have the index point to all the versions of an object and require any index query to filter out object versions which are not visible to the current transaction.

Some databases like CouchDB and Datomic use an *append-only B-tree* which does not overwrites pages of the tree when they are updated or modified, but creates a new copy of each modified page.

PostgreSQL has optimizations to avoid index updates if different versions of the same object can fit on the same page -- *not sure I understand this*

##### Repeatable read and naming confusion [#](#repeatable-read-and-naming-confusion)

SQL Isolation levels are not standardized, and so some databases refer to an implementation of snapshot isolation as *serializable* (e.g. Oracle) or *repeatable read* (e.g. PostgreSQL and MySQL).

DB2 refers to serializability as "repeatable read".

In summary, naming here is a shitshow in database town.

#### Preventing Lost Updates [#](#preventing-lost-updates)

So far, we have only discussed how to prevent dirty writes, but haven't spoken about another problem that could occur: the *lost update* problem. Basically, if two writes concurrently update a value, what's going to happen?

The lost update problem mainly occurs when an application *reads* a value, *modifies* it, and *writes* back the modified value (called a *read-modify-write cycle)*. If two transactions try to do this concurrently, one of the updates can be lost as the second write does not include the first modification.

The key difference between this and a dirty write is that: *if you overwrite a value that has been committed, it's no longer a dirty write. A dirty write happens when in a transaction, you overwrite a value which has been updated in another uncommitted transaction.*

----

This can happen in different scenarios:

- If a counter needs to be incremented. It requires reading the current value, calculating the new value, and writing back the updated value. If two transactions increment the counter by different values, one of those updates will be lost.
- Making a local change to a complex value. E.g. Adding an element to a list within a JSON document.
- Two users editing a wiki page at the same time, where each user's changed is saved by sending the entire page contents to the server, it will overwrite whatever is in the database.

----

A variety of solutions have been developed to deal with this scenario:

- **Atomic Write Operations:** Many databases support atomic updates, which remove the need for the read-modify-write cycles. Atomic updates look like:
    
    UPDATE counters SET counter= counter + 1 WHERE key = 'foo'
    
    Atomic operations are usually implemented by taking an exclusive lock on the object when it's read to prevent any other transaction from reading it until the update has been applied.
    
    Another option for implementing atomic writes is to ensure that all atomic operations run on the same database thread.
    
    However, not all database updates fit into this model. Some updates require more complex logic and won't benefit from atomic writes.
    
- **Explicit Locking:** Another option for preventing lost updates is to explicitly lock objects which are going to be updated. You may need to specify in your application's code through an ORM or directly in SQL that the rows returned from a query should be locked.
    

##### Automatically detecting lost updates [#](#automatically-detecting-lost-updates)

The methods discussed above (atomic operations and locks) are good ways of preventing lost updates as they force the read-modify-write cycles to occur sequentially. One alternative is to allow concurrent cycles to execute in parallel and let the transaction manager detect a lost update.

When a lost update is detected, the transaction can be aborted and it can be forced to retry its read-modify-write cycle.

This approach has an advantage that the check for lost updates can be performed efficiently with snapshot isolation.

PostgreSQL, Oracle and SQL Server automatically detect lost updates.

Another advantage is that application code does not have to use any special database features like atomic locks or explicit locking to prevent lost updates.

##### Compare-and-set [#](#compare-and-set)

In databases which don't provide transactions, an atomic compare-and-set operation is usually found.

What this means is that when an operation wants to update a value, it reads the previous value and only completes if the value at the time of the update is the same as the value it read earlier.

This is safe as long as the database is not comparing the current value against an old snapshot.

##### Conflict resolution and replication [#](#conflict-resolution-and-replication)

Locks and compare-and-set operations assume that there's a single up-to-date copy of the data. However, for databases with multi-leader or leaderless replication, there's no guarantee of a single up-to-date copy of data.

A common approach in replicated databases is to allow concurrent writes create several conflicting versions of a value, and allow application code or special data structures to be used to resolve the conflicts.

#### Write Skew and Phantoms [#](#write-skew-and-phantoms)

To recap the two race conditions we have treated so far:

- *Dirty Writes:* The ability of one running transaction to overwrite an update made by another running, uncommitted transaction. If a transaction updates multiple objects, dirty writes can lead to a bad outcome. There's the car example in the book of Alice and Bob buying a car in different and having to update the listings and invoices tables. If Alice writes to the listings table first and Bob overrides it, but Bob writes to the listings table first and Alice overrides it due to a delay, we'll have inconsistent records in our tables as the record for that car should have the same recipient.
    
- *Lost Update:* If two transactions happen concurrently and another commit firsts, the later one could overwrite an update made by the earlier transaction, which could lead to lost changes. E.g. Incrementing a counter.

----

Another anomaly that can happen is a *Write skew.* Basically, if two transactions read from the same objects, and then update some of those objects (as different transactions may update different objects), a write skew can occur.

Imagine a scenario where two transactions running concurrently first make a query, and then update a database object based on the result of the first query. The operations performed by the transaction to commit first may render the result of the query invalid for the later transaction.

These transactions may update different objects (so it's neither a dirty write nor a lost update), but they'll still make the application function incorrectly. E.g. A meeting room booking app where two transactions running concurrently first see that a timespan was not booked, and then add a row each for different meetings. *This wouldn't be a problem if we could somehow have unique constraints on all the time ranges, but it's a problem if we don't.*

Database constraints like uniqueness and foreign key constraints may help to enforce this, but they're not always applicable.

Serializable isolation helps to prevent this. However, if it's not available, one way of preventing this is to explicitly lock the rows that a transaction depends on. Unfortunately, if the original query returns no rows (say it's checking for the absence of rows matching a condition), we can't attach locks to anything

----

*Another example drummed up from my head:*

If you're distributing students into the same class room and you want to make sure that no students with the same last name belong to the same class. Two concurrently executing transactions could first check that the condition is met, then insert separate rows for a surname. Of course, a simple solution to this is a uniqueness constraint.

The effect, where a write in one transaction changes the result of a search query in another transaction, is called a ***phantom.***

##### Materializing Conflicts [#](#materializing-conflicts)

As described above, we can reduce the effect of phantoms by attaching locks to the rows used in a transaction. However, if there's no object to which we can attach the locks (say if our initial query is searching for the absence of rows), we can artificially introduce locks.

The approach of taking a phantom and turning it into a lock conflict on a concrete set of rows introduced in the database is known as *materializing conflicts.*

This should be a last resort, as a serializable isolation level is much preferable in most cases.

### Serializability [#](#serializability)

Serializable isolation is regarded as the strongest isolation level. It guarantees that even though transactions may execute in parallel, the end result will be as if they executed one at a time, without any concurrency.

Most databases use of the following three techniques for serializable isolation, which we will explore next:

- *Actual serial execution* i.e. literally executing transactions in a serial order.
- *Two-phase locking*
- *Serializable snapshot isolation*

#### Actual Serial Execution [#](#actual-serial-execution)

The simplest way to avoid concurrency issues is by removing concurrency entirely, and making sure transactions are executed in serial order, on a single thread.

This idea only started being used in practice fairly recently, around 2007, when database designers saw that it was feasible to use a single-threaded loop for executing transactions and still get good performance. Two developments led to this revelation:

- RAM become cheap enough that it is now possible to fit an entire dataset in memory for many use cases. When the entire dataset needed for a transaction is stored in memory, transactions execute much *faster.*
- They also realized that OLTP transactions are usually short and make only a small number of reads and writes. Unlike OLAP transactions which are typically run on a consistent snapshot, these transactions are short enough to be run one by one.

----

This approach is serial transaction execution is implemented in Datomic, Redis, and others.

This can even be faster than databases that implement concurrency, as there is no coordination overhead of locking.

The downside is that its throughput is limited to a single CPU core (processor), except you can structure your transactions differently into partitions (discussed below).

##### Encapsulating transactions in stored procedures [#](#encapsulating-transactions-in-stored-procedures)

Transactions are typically executed in a client/server style, one statement at a time: an application makes a query, reads the result, maybe makes another query depending on the first result etc.

In this interactive style, a lot of time is spent in network communication between the application and the database.

In a database with no concurrency that only processes one transaction at a time, the throughput won't be great, as the transaction will spend most of its time waiting for the next request.

For this reason, databases which have single-threaded serial transaction processing do not allow interactive multi-statement transactions. Instead, all the requests in a transaction must be submitted at the same time as a *stored procedure.* With this approach, there's only one network hop instead of having multiple.

Stored procedures and in-memory data make executing transactions on a single thread become feasible.

##### Partitioning [#](#partitioning)

As mentioned earlier, executing transactions serially makes concurrency control simpler, but limits the throughput to the speed of a single CPU core on a single machine.

What if we could take advantage of the presence of multiple cores and multiple node by partitioning the data?

If we can partition the data so that each transaction only needs to read and write data within a single partition, then each partition can have its own transaction processing thread running independently from the others.

This will likely involve some cross-partition coordination though, especially in the presence of secondary indexes and what not. Cross-partition coordination has much less throughput than single partitions.

##### Summary of serial execution [#](#summary-of-serial-execution)

- It requires that each transaction is small and fast, else one slow transaction can stall all transaction processing.
- It's limited to use cases where the active dataset can fit in memory. A transaction that needs to access data not in memory can slow down processing.
- Write throughput must be low enough to be handled on a CPU core, or else transactions need to be partitioned without requiring cross-partition coordination.

#### Two-Phase Locking (2PL) [#](#two-phase-locking-%282pl%29)

For a long time (around 30 years), two-phase locking was the most widely used algorithm for serializability in databases.

The key ideas behind two-phase locking are these:

- A transaction cannot write a value that has been read by another transaction.
- A transaction cannot read a value that has been written by another transaction. It must wait till the other transaction commits or aborts. Reading an old version of the object (like in snapshot isolation) is not acceptable.

Unlike snapshot isolation, readers can block writers here, and writers can block readers. (Recall that snapshot isolation has the mantra: *readers never block writers, and writers never block readers)*

##### Implementation of two-phase locking [#](#implementation-of-two-phase-locking)

The blocking of readers and writers is implemented by having a lock on each object used in a transaction. The lock can either be in *shared mode* or in *exclusive mode.* The lock is used as follows:

- When a transaction wants to read an object, it must first acquire a shared mode lock. Multiple read-only transactions can share the lock on an object.
- When a transaction wants to write an object, it must acquire an exclusive lock on that object.
- If a transaction first reads and then writes to an object, it may upgrade its shared lock to an exclusive lock.
- After a transaction has acquired the lock, it must continue to hold the lock until the end of the transaction.

----

Two-phase origin: First phase is acquiring the locks, second phase is when all the locks are released.

If an exclusive lock already exists on an object, a transaction which wants to acquire a shared mode lock must wait for the lock to be released (when the transaction is aborted or committed), and vice versa.

Since so many locks are in use, a deadlock can easily happen if transaction A is stuck waiting for transaction B to release its lock, and vice versa. E.g. If transaction A has a lock on table A, and needs table B, but transaction B already has a lock on table B but needs table A, we are in a deadlock situation.

*Aside: Note that some databases perform row-level locks and others perform table-level locks.*

##### Performance of two-phase locking [#](#performance-of-two-phase-locking)

Transaction throughput and response times of queries are significantly worse under two-phase locking than under weak isolation.

This is as a result of the overhead of acquiring and releasing locks, but more importantly due to reduced concurrency as some transactions have wait for the others to complete. One slow transaction can cause the system to be slow.

Deadlocks also occur more frequently here than in lock-based read committed isolation levels.

##### Predicate Locks [#](#predicate-locks)

The idea with predicate locks is to lock all the objects that meet a condition, even for objects that do not yet exist in the database. A condition is first run in a select query, and a predicate lock holds a lock on any objects that could meet that query.

##### Index Range Locks [#](#index-range-locks)

Predicate locks don't perform well, however, and checking for matching locks can become time consuming. Thus, most databases implement index-range locking.

With index range locks, we don't just lock the objects which match a condition, we lock a bigger range of objects. For example, if an index is hit in the original query, we could lock any writes to that index entry, even if it doesn't match the condition.

These are not as precise as predicate locks, but there's less overhead with checking for locks as more objects will be locked.

#### Serializable Snapshot Isolation [#](#serializable-snapshot-isolation)

Most of this chapter has been bleak, and has made it look like we either have serializable isolation with poor performance, or weak isolation levels that are prone to lost updates, dirty writes, write skews and phantoms.

However, in 2008, Michael Cahill's PhD thesis introduced a new concept known as *serializable snapshot isolation.* It provides full serializability at only a small performance penalty compared to snapshot isolation.

----

The main idea here is that instead of holding locks on transactions, it allows transactions to continue to execute as normal until the stage where the transaction is about to commit, when it then decides whether the transaction executed in a serializable manner. This approach is known as *an optimistic concurrency control technique*.

Optimistic in this sense means that instead of blocking if something potentially dangerous happens, transactions continue anyway, in the hope that everything will be fine. If everything isn't fine, it's only controlled at the time the transactions want to commit, after which it will be aborted. This approach differs from the *pessimistic technique* used in Two-phase locking.

The pessimistic approach believes that if anything can go wrong, it's better to wait until the situation is safe again (using locks) before doing anything.

----

Optimistic concurrency control is an old idea, but in the right conditions (e.g. contention between transactions is not too high and there's enough spare capacity), they tend to perform better than pessimistic ones.

*SSI* is based on snapshot isolation and obeys the rules that readers don’t block writers, and writers don't block readers. The main difference is that SSI adds an algorithm for detecting serialization conflicts among writes and determining which transactions to abort.

##### Decisions based on an outdated premise [#](#decisions-based-on-an-outdated-premise)

With write skew in snapshot isolation, the recurring pattern was this: a transaction reads some data from the database, examines the result of the query and takes some action based on the result of that query. However, the result from the original query may no longer be valid as at the time the transaction commits, because the data may have been modified in the meantime.

----

The database has to be able to detect situations in which a transaction may have acted on outdated premise and abort the transaction in that case. To do this, there are two cases to consider:

- Detecting reads of a stale MVCC object version (uncommitted write occurred before the read).
- Detecting writes that affect prior reads (the write occurs after the read).

----

**Detecting stale MVCC reads**

Basically, with snapshot isolation, there can be multiple versions of an object. When a transaction reads from a consistent snapshot in an MVCC database, it ignores writes that were made by transactions that had not committed at the time the snapshot was taken.

This means that if Transaction A reads a value when there are uncommitted writes by Transaction B to that value, and transaction B commits before transaction A, then Transaction A may have performed some operations as a result of that earlier read which is no longer valid.

To prevent this anomaly, a database needs to keep track of transactions which ignore another transaction's writes due to MVCC visibility rules. When the transaction that performed the read wants to commit, the database checks whether any of the ignored writes have been committed. If so, the transaction must be aborted.

----

Some of the reasons why it waits for the transaction to commit, rather than aborting a transaction when a stale read is detected are that:

- The reading transaction might be a read-only transaction, in which case there's no risk of a write skew. The database has no way of knowing whether the transaction will later perform a write.
- There's no guarantee that the transaction that performed the uncommitted write will actually commit, so the read may not be a stale one at the end.

SSI avoids unnecessary aborts and thus preserves snapshot isolation's support for long-running reads from a consistent snapshot.

----

**Detecting writes that affect prior reads**

The key idea here is to keep track of which values (for example, track the indexes) have been read by what transactions. When a transaction writes to the database, it looks in the indexes for what other transactions have recently read the data. It then notifies the transactions that what they read may be out of date.

##### Performance of serializable snapshot isolation [#](#performance-of-serializable-snapshot-isolation)

The advantage that this has over two-phase locking that one transaction does not need to be blocked when waiting for locks held by another transaction. Recall that like under snapshot isolation, readers don't block writers here and vice versa.

This also has the advantage over serial execution that it is not limited to the throughput of a single CPU core.

Due to the fact that the rate of aborts will affect the performance of SSI, it requires that read-write transactions be fairly short, as long running ones are more likely to run into conflicts. Long running read-only transactions may be fine.

However, note that SSI is likely to be less sensitive to slow transactions that two-phase locking or serial execution.