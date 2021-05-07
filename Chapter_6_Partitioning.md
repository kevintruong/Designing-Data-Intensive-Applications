# Chapter 6 - Partitioning - A blog by Timi Adeniran

*15 Dec 2019*

My notes from Chapter 6 of 'Designing Data-Intensive Applications by Martin Kleppmann'

**Table of Contents**

- [Partitioning and Replication](#partitioning-and-replication)
- [Partitioning of Key-Value Data](#partitioning-of-key-value-data)
    - [Partitioning by Key Range](#partitioning-by-key-range)
    - [Partitioning by Hash of Key](#partitioning-by-hash-of-key)
    - [Skewed Workloads and Relieving Hot Spots](#skewed-workloads-and-relieving-hot-spots)
- [Partitioning and Secondary Indexes](#partitioning-and-secondary-indexes)
    - [Partitioning Secondary Indexes by Document](#partitioning-secondary-indexes-by-document)
    - [Partitioning Secondary Indexes by Term](#partitioning-secondary-indexes-by-term)
- [Rebalancing Partitions](#rebalancing-partitions)
    - [Operations: Automatic or Manual Rebalancing](#operations-automatic-or-manual-rebalancing)
- [Request Routing](#request-routing)

* * *

### Introduction 

- This refers to breaking up a large data set into *partitions.*
- Partitioning is also known as *sharding.*
- Partitions are known as *shards* in MongoDB, Elasticsearch and SolrCloud, *regions* in Hbase, *tablets* in Bigtable, *vnodes* in Cassandra and Riak, and *vBucket* in Couchbase.
- Normally, each piece of data belongs to only one partition.
- *Scalability* is the main reason for partitioning data. It enables a large dataset to be distributed across many disks, and a query load can be distributed across many processors.

### Partitioning and Replication [#](#partitioning-and-replication)

Copies of each partition are usually stored on multiple nodes. Therefore, although a record belongs to only one partition, it may be stored on several nodes for fault tolerance.

A node may also store more than one partition. If the leader-follower replication model is used for partitions, each node may be the leader for some partitions and a follower for other partitions.

### Partitioning of Key-Value Data [#](#partitioning-of-key-value-data)

The goal of partitioning is to spread data and query load evenly across nodes.

If partitioning is unfair i.e. some partitions have more data than others, we call it *skewed.*

If the partitioning is skewed, it makes partitioning less effective as all the load could end up one partition, effectively making the other nodes idle. A partition with a disproportionately high load is called a *hot spot.*

Assigning records to nodes randomly is the simplest approach for avoiding hot spots, but this has the downside that it will be more difficult to read a particular item since there's no way of knowing which node the item is on.

There are other approaches for this though, and we'll talk about them.

#### Partitioning by Key Range [#](#partitioning-by-key-range)

For key-value data where the key could be a primary key, each partition could be assigned a continuous range of keys, say a-d, e-g etc. This way, once we know the boundaries between the ranges, it's easy to determine which partition contains a given key.

This range does not have to be evenly spaced. Some partitions could cover a wider range of keys than others, because the data may not be evenly distributed. Say we're indexing based on a name, one partition could hold keys for: u,v, w, x,y and z, while another could hold only a-c.

To distribute data evenly, the partition boundaries need to adapt to the data.

Hbase, Bigtable, RethinkDB etc use this partitioning strategy.

Keys could also be kept in sorted order within each partition. This makes range-scans effective e.g. find all records that start with 'a' .

The downside of this partitioning strategy is that some access patterns can lead to hotspots. E.g . If the key is a timestamp, then partitions will correspond to ranges of time e.g. one partition by day. This means all the writes end up going to the same partition for each day, so that partition could be overloaded with writes while others sit idle.

The solution for the example above is to use something apart from the timestamp as the primary key, but this is most effective if you know something else about the data. Say for sensor data, we could partition first by the sensor name and then by time. Though with this approach, you would need to perform a separate range query for each sensor name.

#### Partitioning by Hash of Key [#](#partitioning-by-hash-of-key)

Many distributed datastores use a hash function to determine the partition for a given key.

When we have a suitable hash function for keys, each partition can be assigned a range of hashes, and every key whose hash falls within a partition's range will be stored for that partition.

A good hash function takes skewed data and makes it uniformly distributed. The partition boundaries can be evenly spaced or chosen pseudo-randomly (with the latter being sometimes known as *consistent hashing)*.

----

*Aside copied from book:*

Consistent hashing, as defined by Karger et al, is a way of evenly distributing load across an internet-wide system of caches such as a content delivery network (CDN). It uses randomly chosen partition boundaries to avoid the need for central control or distributed consensus. Note that consistent here has nothing to do with replica consistency or ACID consistency, but rather describes a particular approach to rebalancing.

As we shall see in “Rebalancing Partitions”, this particular approach actually doesn’t work very well for databases, so it is rarely used in practice (the documentation of some databases still refers to consistent hashing, but it is often inaccurate). It's best to avoid the term *consistent hashing* and just call it hash partitioning instead.

Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 5169-5174). O'Reilly Media. Kindle Edition.

----

This approach has the downside that we lose the ability to do efficient range queries. Keys that were once adjacent are now scattered across all the partitions, so sort order is lost.

Note:

- Range queries on the primary key are not supported by Riak, Couchbase, or Voldermort.
    
- MongoDB sends range queries to all the partitions.
    
- *Cassandra* has a nice compromise for this. A table in Cassandra can be declared with a compound primary key consisting of several columns. Only the first part of the key is hashed, but the other columns act as a concatenated index.
    
    Thus a query cannot search for a range of values within the first column of a compound key, but if it specifies a fixed value for the first column, it can perform an efficient range scan over the other columns of the key.
    

#### Skewed Workloads and Relieving Hot Spots [#](#skewed-workloads-and-relieving-hot-spots)

Even though hashing the key helps to reduce skew and the number of hot spots, it can't avoid them entirely. In an extreme case where reads and writes are for the same key, we still end up with all requests being routed to the same partition.

This workload seems unusual, but it's not unheard of. Imagine a social media site where a celebrity has lots of followers. If the celebrity posts something and that post has tons of replies, all the writes for that post could potentially end up at the same partition.

Most data systems today are not able to automatically compensate for a skewed workload. It's up to the application to reduce the skew. E.g if a key is known to be very hot, a technique is to add a random number to the beginning or end of the key to split write. This has the downside though that reads now have to do additional work to keep track of these keys.

### Partitioning and Secondary Indexes [#](#partitioning-and-secondary-indexes)

The partition techniques discussed so far rely on a key-value data model, where records are only ever accessed by their primary key. In these situations, we can determine the partition from that key and use it to route read and write requests to the partition responsible for the key.

If secondary indexes are involved though, this becomes more complex since they usually don't identify a record uniquely, but are a way of searching for all occurrences of a particular value.

Many key value stores like Hbase and Voldermort have avoided secondary indexes because of their complexity, but some like Riak have started adding them because of their usefulness for data modelling.

----

Secondary indexes are the bread & butter of search servers like Elasticsearch and Solr though. However, a challenge with secondary indexes is that they do not map neatly to partitions. Two main approaches to partitioning a database with secondary indexes are:

- Document-based partitioning and
- Term-based partitioning.

#### Partitioning Secondary Indexes by Document [#](#partitioning-secondary-indexes-by-document)

In this partitioning scheme, each document has a unique document ID. The documents are *initially* partitioned by that ID, which could be based on a hash range or term range of the ID. Each partition then maintains its own secondary index, covering only the documents that fit within that partition. It does not care what data is stored in other partitions.

A document-partitioned index is also known as a *local index.*

The downside of this approach is that reads are more complicated. If we want to search by a term in the secondary index (say we're searching for all the red cars where there's an index on the color field), we would typically have to search through all the partitions, and then combine all the results we get back.

This approach to querying a partitioned db is sometimes known as *scatter/gather,* and it can make read queries on secondary indexes quite expensive. This approach is also prone to tail latency amplification (amplification in the higher percentiles), but it's widely used in Elasticsearch, Cassandra, Riak, MongoDB etc.

#### Partitioning Secondary Indexes by Term [#](#partitioning-secondary-indexes-by-term)

In this approach, we keep a *global index* of the secondary terms that covers data in all the partitions. However, we don't store the global index on one node, since it would likely become a bottleneck, but rather, we partition it. The global index must be partitioned, but it can be partitioned differently from the primary key index.

The advantage of a global index over document-partitioned indexes is that reads are more efficient, since we don't need to scatter/gather over all partitions. A client only needs to make a request to the partition containing the desired document.

However, the downside is that writes are slower and more complicated, because a write to a single document may now affect multiple partitions of the index.

In practice, updates to global secondary indexes are often asynchronous (i.e. if you read the index shortly after a write, it may not have reflected a new change).

### Rebalancing Partitions [#](#rebalancing-partitions)

**How not to do it: hash mod n**

If we partition by *hash mod n* where n is the number of nodes, we run the risk of excessive rebalancing. If the number of nodes N changes, most of the keys will need to be moved from one node to another. Frequent moves make rebalancing excessively expensive.

Next we'll discuss approaches that don't move data around more than necessary.

----

**Fixed Number of Partitions**

This approach is fairly simple. It involves creating more partitions than nodes, and assigning several partitions to each node.

If a node is added to a cluster, the new node *steals* a few partitions from other nodes until partitions are fairly distributed again. Only entire partitions are moved between nodes, the number of partitions does not change, nor does the assignment of keys to partitions.

What changes is the assignment of partitions to nodes. It may take some time to transfer a large amount of data over the network between nodes, so the old assignment of partitions is typically used for any reads/writes that happen while the transfer is in progress.

This approach is used in Elasticsearch (where the number of primary shards is fixed on index creation), Riak, Couchbase and Voldermort.

With this approach, it's imperative to choose the right number of partitions.

----

**Dynamic partitioning**

In this rebalancing strategy, the number of partitions dynamically adjusts to fit the size of the data. This is especially useful for databases with key range partitioning, as a fixed number of partitions with fixed boundaries can be inconvenient if not configured correctly at first. All the data could end up on one node, leaving the other empty.

When a partition grows to exceed a configured size, it is split into two partitions so that approximately half of the data ends up on each side of the split.

Conversely, if lots of data is deleted and a partition shrinks below some threshold, it can be merged with an adjacent partition.

An advantage of this approach is that the number of partitions adapts to the total data volume. If there's only a small amount of data, a small number of partitions is sufficient, so overheads are small.

A downside of this approach though is that an empty database starts off with a single partition, since there's no information about where to draw the partition boundaries. While the data set is small, all the writes will be processed by a single node while the others sit idle. Some databases like Hbase and MongoDB mitigate this by allowing an initial set of partitions to be configured on an empty database.

----

**Partitioning proportionally to nodes**

Some databases (e.g. Cassandra) have a third option of making the number of partitions proportional to the number of nodes. There is a fixed number of partitions *per node*.

The size of each partition grows proportionally to the dataset size while the number of nodes remains unchanged, but when you increase the number of nodes, the partitions become smaller again.

When a new node joins the cluster, it randomly chooses a fixed number of existing partitions to split, and then takes one half of the split, leaving the other half in place.

This randomization can produce an unfair split, but databases like Cassandra have come up with algorithms to mitigate the effect of that.

#### Operations: Automatic or Manual Rebalancing [#](#operations%3A-automatic-or-manual-rebalancing)

Databases like Couchbase, Riak, and Voldermort are a good balance between automatic and manual rebalancing. They generate a suggested partitioned assignment automatically, but require an administrator to commit it before it takes effect.

Fully automated rebalancing can be unpredictable. Rebalancing is an expensive operation because it requires rerouting requests and moving a large amount of data from one node to another. This process can overload the network or the nodes if not done carefully.

For example, say one node is overloaded and is temporarily slow to respond to requests. The other nodes can conclude that the overloaded node is dead, and automatically rebalance the cluster to move load away from it. This puts additional load on the overloaded node, other nodes, and the network — making the situation worse and potentially causing a cascading failure.

### Request Routing [#](#request-routing)

There's an open question that remains: when a client makes a request, how does it know which node to connect to? It's especially important as the assignment of partitions to nodes changes.

This is an instance of the general problem of *service discovery i.e. locating things over a network.* This isn't just limited to databases, any piece of software that's accessible over a network has this problem.

---- 

There are different approaches to solving this problem on a high level:

- Allow clients to contact any node (e.g. via a round-robin load balancer). If the node happens to own the partition to which the request applies, it can handle the request directly. Otherwise, it'll forward it to the relevant node.
- Send all requests from clients to a routing tier first, which determines what node should handle what request and forwards it accordingly. This routing tier acts as a partition-aware load balancer.
- Require that clients be aware of the partitioning and assignment of partitions to nodes.

----

In all approaches, it's important for there to be a *consensus* among all the nodes about which partitions belong to which nodes, especially as we tend to rebalance often.

Many distributed systems rely on a coordination service like Zookeeper to keep track of this cluster metadata. This way, the other components of the system such as the routing tier or the partitioning-aware client can subscribe to this information in Zookeeper.

When using a routing tier or when sending requests to a random node, clients still need to find the IP addresses to connect to. These are not as fast-changing as the assignment of partitions to nodes, so it is often sufficient to use DNS for this purpose. *Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Locations 5478-5480). O'Reilly Media. Kindle Edition.*