# Chapter 3 - Storage and Retrieval

*07 Dec 2019*

These are my notes from the third chapter of Martin Kleppmann's: Designing Data Intensive Applications.

**Table of Contents**

- [Storage Engines](#storage-engines)
    - [Log-Structured Storage Engines](#log-structured-storage-engines)
        - [Indexing](#indexing)
            - [Hash Index](#hash-index)
            - [SSTables and LSM-Trees](#sstables-and-lsm-trees)
                - [Constructing and maintaining SSTables](#constructing-and-maintaining-sstables)
                - [Making an LSM-tree out of SSTables](#making-an-lsm-tree-out-of-sstables)
                - [Performance Optimizations](#performance-optimizations)
            - [B-Trees](#b-trees)
                - [Making B- Trees reliable](#making-b--trees-reliable)
                - [B-tree optimizations](#b-tree-optimizations)
                - [Comparing B-Trees and LSM-Trees](#comparing-b-trees-and-lsm-trees)
                - [Advantages of LSM Trees](#advantages-of-lsm-trees)
                - [Downsides of LSM Trees](#downsides-of-lsm-trees)
            - [Other Indexing Structures](#other-indexing-structures)
                - [Storing values within the index](#storing-values-within-the-index)
                - [Approach 2 - Heap file](#approach-2---heap-file)
                - [Approach 1 - Actual row](#approach-1---actual-row)
                - [Covering Index](#covering-index)
                - [Multi-column indexes](#multi-column-indexes)
                - [Full-text search and fuzzy indexes](#full-text-search-and-fuzzy-indexes)
                - [Keep everything in memory](#keep-everything-in-memory)
            - [Transaction Processing or Analytics?](#transaction-processing-or-analytics)
                - [Data Warehousing](#data-warehousing)
                - [Stars and Snowflakes: Schemas for Analytics](#stars-and-snowflakes-schemas-for-analytics)
            - [Column Oriented Storage](#column-oriented-storage)
                - [Column Compression](#column-compression)
                - [Column-oriented storage and column families](#column-oriented-storage-and-column-families)
                - [Sort Order in Column Storage](#sort-order-in-column-storage)

* * *

## Introduction

This chapter helps to arm with the knowledge of how databases work under the hood.

There's a difference between storage engines that are optimized for transactional workloads and those that are optimized for analytics.

There are two families of storage engines: log-structured storage engines (log structured merge trees), and page-oriented storage engines (b trees). A storage engine’s job is to write things to disk on a single node.

## Log-Structured Storage Engines [#](#log-structured-storage-engines)

Many databases internally use a *log,* an append-only data file for adding something to it. Each line in the log contains a key-value pair, separated by a comma (similar to a CSV file, ignoring escaping issues). The log does not have to be internally-readable, it might be binary and intended only for other programs to read.

### Indexing [#](#indexing)

An index is an additional structure derived from the primary data. Any kind of index usually slows down writes, since the index has to be updated every time data is written.

*Well-chosen indexes speed up read queries, but every index slows down writes.*

#### Hash Index [#](#hash-index)

These are indexes for key-value data. For a data storage that consists of only appending to a file, a simple indexing strategy is to keep an in-memory hash map where the value for every key is a byte offset which is where the key is located. The value for the key is then read.

Bitcask (the default storage engine in Riak - Riak is a distributed datastore similar to Cassandra) uses the approach above. The only requirement is that all the keys fit in the available RAM as the hash map is kept completely in memory. The values don't have to fit in memory since they can be loaded from disk with a simple disk seek. Something like Bitcask is suitable for situations where the value for a key is updated frequently.

The obvious challenge in appending to a file is that the file can grow too large and then we run out of disk space. A solution to this is to break the log into segments of a certain size. A segment file is closed when it reaches that size, and subsequent writes are made to a new segment.

We can then perform *compaction* on these segments. Compaction means keeping the most recent update for each key and throwing away duplicate keys. Compaction often makes segments smaller (relies on the assumption that a key is overwritten several times on average within one segment), and so we can *merge* several segments together at the same time as performing the compaction.

Basically, we compact and merge segment files together. The merged segment is written to a new file. This can happen as a background process, so the old segment files can still serve read and write requests, until the merging process is complete.

**Each segment will have its own in-memory hash table.** To find a value for a key, we'll check the most recent segment. If it's not there, we'll check the second-most-recent segment, and so on.

There are certain practical issues that must be considered in a real life implementation of this hash index in a log structure. Some of them are:

- *File Format*: As opposed to using a CSV format, it's faster and simpler to use a binary format that first encodes the length of a string in bytes, followed by the raw string.
- *Deleting Records*: To delete a key and its value, it's not practical to search for all the occurrences of that key in the segments. What happens is that a special deletion record is appended to the data file (sometimes called a *tombstone).* When log segments are merged, the tombstone tells the merging process to discard any previous values for the deleted key.
- *Crash Recovery*: If the database is restarted, the in-memory hash maps will be lost. In principle, the segment's hash maps can be restored by reading the entire segment files and constructing the hash maps from scratch. This might take a while though, so could make server restarts painful. **Bitcask's** approach to recovery is by storing a snapshot of each segment's hash map on disk, which can be loaded into memory more quickly
- *Concurrency Control:* Since writes are appended in a sequential order, a common implementation is to have only one writer thread. Data files are append-only and otherwise immutable, so they can be read concurrently by multiple threads.

#### There are good reasons why an append-only log are a good choice as opposed to a storage where files are updated in place, overwriting the old value with the new value.

Some of those reasons are:

- Appending and segment merging are sequential write operations, which are generally faster than random writes, especially on magnetic spinning-disk hard drives.
- Concurrency and crash recovery are simpler if segment files are append-only or immutable. For crash recovery, you don't need to worry if a crash happened while a value was being overwritten, leaving you with partial data.
- Merging old segments avoids the problem of data files getting fragmented over time. Fragmentation occurs on a hard drive, a memory module, or other media when data is not written closely enough physically on the drive. Those fragmented, individual pieces of data are referred to generally as fragments.

#### Form of fragmentation
*Basically, when data files are far from each other, it's a form of fragmentation.*

There are limitations to the hash table index though, some of them are:

- The hash table must fit in memory, so it's not efficient if there are a large number of keys. An option is to maintain a map on disk, but it doesn't perform so well. It requires a lot of random access I/O, is expensive to grow when it becomes full, and hash collisions require fiddly logic.
- Range queries are not efficient. You have to look up each key individually in the map.

So, in this approach, writes are made to the segments on a disk while the hash table index being stored is in - memory.

#### SSTables and LSM-Trees [#](#sstables-and-lsm-trees)

In log segments with hash indexes, each key-value pair appears in the order that it was written, and values later in the log take precedence over values for the same key earlier in the log. Apart from that, the order of key-value pairs in the file is irrelevant.

There is a change to this approach in a *Sorted String Table* format, or SSTable for short. Here, it is required that the sequence of key-value pairs is sorted by key. Hence, new key-value pairs cannot be appended to the segment immediately. There are several advantages to this over log segments with hash indexes:

- Merging segments is simple and efficient. The approach here is similar to the mergesort algorithm. The same principle as the log with hash indexes applies if a key is duplicated across several segments. We keep the most recent one and discard the others.
- You don't need to keep an index of all the keys in memory. Because the file is sorted, if you're looking for the offset of a particular key, it won't be difficult to find that offset once you can determine the offset of keys that are smaller and larger than it in the ordering.

You still need an in-memory index to tell you the offsets of some keys, but it can be sparse.

- Read requests often need to scan over several key-value pairs, therefore it is possible to group the records into a block and compress it before writing it to disk. Each entry of the sparse index then points at the start of a compressed block. This has the advantage of saving disk space and reducing the IO bandwidth.

SSTables store their keys in blocks, and have an internal index, so even though a single SSTable may be very large (gigabytes in size), only the index and the relevant block needs to be loaded into memory.

##### Constructing and maintaining SSTables [#](#constructing-and-maintaining-sstables)

It's possible to maintain a sorted structure on disk( see B-Trees) but maintaining it in memory is easier and is the approach described here. The approach is to use well-known tree data structures such as red-black trees or AVL trees into which keys can be inserted in any order and read back in sorted order.

So the storage engine works as follows:

- When a write comes in, it is written to an in-memory balanced tree data structure. The in-memory tree is sometimes called a *memtable.*
- When the memtable exceeds a threshold, write it out to disk as an SSTable file. This operation is efficient because the tree already maintains the key-value pairs sorted by key. The new SSTable file then becomes the most recent segment of the database. While the SSTable is being written out to disk, writes can continue to a new memtable instance.
- To serve a read request, first check for the key in the memtable. If it's not there, check the most recent segment, then the next-older segment etc
- From time to time, run a merging and compaction process in the background to combine segment files and to discard overwritten or deleted values.

An obvious problem with this approach is that if the database crashes, the most recent writes (which are in the memtable but not yet written to disk) will disappear. To avoid that problem, one approach is to keep a separate log on disk to which every write is immediately appended. This separate log is not in sorted order, but that's irrelevant because the content can easily be sorted in a memtable. The corresponding log can be discarded every time the memtable is written out to an SSTable.

##### Making an LSM-tree out of SSTables [#](#making-an-lsm-tree-out-of-sstables)

The algorithm described above is used in LevelDB and RocksDB. Key-value storage engine libraries are designed to be embedded into other applications. Among other things, LevelDB can be used in Riak as an alternative to Bitcask as its storage engine.

This indexing structure was originally described under the name *Log-Structured Merge-Tree.*

Lucene which is an indexing engine for full-text search uses a similar method for storing it's *term dictionary.* A full-text index is more complex than a key-value index but is based on a similar idea: given a word in a search query, find all the documents that mention the word. It's usually implemented with a key-value structure where the key is a word ( a *term*) and the value is a list of IDs of all the documents that contain the word (the *postings list).* In Lucene, the mapping from term to postings list is kept in SSTable-like sorted files that are merged in the background as needed.

*Right so, LSM trees are the indices, but the actual storage on disk is as an SSTable? Or do we still need to keep an in-memory index of the memtable?*

##### Performance Optimizations [#](#performance-optimizations)

The LSM-tree algorithm can be slow when looking up keys that do not exist in the database: you first have to check the memtable, then all the segments all the way up to the oldest (possibly having to read from disk for each one) to be certain that the key does not exist. In order to optimize this access, storage engines often make use of *Bloom filters.*

A *Bloom filter* is a memory- efficient data structure for approximating the contents of a set. It can tell you if a key does not appear in a database, thus saving you from unnecessary disk reads for nonexistent keys.

There are also strategies to determine the order and timing of how SSTables are compacted and merged. Two most common options are *size-tiered* and *leveled* compaction. LevelDB and RocksDB use leveled compaction, Hbase uses size-tiered and Cassandra supports both.

*Size-Tiered Compaction*: Here, newer and smaller SSTables are successively merged into older and larger SSTables.

*Leveled Compaction***:** The key range is split into smaller SSTables and older data is moved into separate "levels". This allows compaction to proceed more incrementally and use less disk space. The levels are structured roughly so that each level is in total 10x as large as the level above it. New keys arrive at the highest layer, and as that level gets larger and larger and hits a threshold, some SSTables at that level get compacted into fewer (but larger) SSTables one level lower.

Within a single level, SSTables are non-overlapping: one SSTable might contain keys covering the range \[a,b), the next \[b,d), and so on. The key-space does overlap between levels: if you have two levels, the first might have two SSTables (covering the ranges above), but the second level might have a single SSTable over the key space \[a,e). Looking for the key aardvark requires looking in two SSTables: the \[a,b) SSTable in Level 1, and the \[a,e) SSTable in Level 2.

*Basically, a level has many SSTables.*

#### B-Trees [#](#b-trees)

B-trees are a popular indexing structure. Like SSTables, they keep key-value pairs sorted by key, but the similarity ends there.

Log-structured indexes break the database down into segments, however B-trees break the database down into fixed size *blocks* or *pages.* Each page can be identified with its address or location on disk, which allows one page to refer to another. Pages are usually small in size, typically 4kb compared to segments which can be several megabytes. Pages are stored on disk.

*Right - so are B-trees simply an indexing structure? Like they don’t refer to how the underlying data is stored?*

One page is designated as the root of the B-tree; whenever you want to look up a key in the index, you start here. The page contains several keys and references to child pages. Each child is responsible for a continuous range of keys, and the keys between the references indicate where the boundaries between those ranges lie.

*Branching factor:* The number of references to child pages in one page the B-tree.

To update the value of an existing key in a B-tree, you search for the leaf page containing that key, change the value in that page, and write the page back to disk (any references to that page remain valid). To add a new key, find the page whose range encompasses the new key and add it to that page. If there's no free space on that page, split the page into two half-full pages, and update the parent page to account for the new subdivision of key ranges.

##### Making B- Trees reliable [#](#making-b--trees-reliable)

The main write operation of a B-tree is to overwrite a page on disk with new data. The assumption is that an overwrite does not change where a page is located i.e. all the references to a page typically remain intact when the page is overwritten. This differs from LSM trees where things are never updated in place, and are append only.

Some operations require different pages to be overwritten e.g. when a page is split because an insertion caused it to be overfull. We'll need to write the two pages that were split and update the parent page with references to the two child pages. This operation is dangerous especially if the database crashes after only some pages have been written, this can lead to a corrupted index.

A solution used to make databases resilient to crashes is to keep a ***write-ahead log*** on disk. It is an append-only file to which every B-tree modification must be written before it can be applied to the pages of the tree itself. It's used to restore the DB when it comes back from a crash.

There are also concurrency issues associated with updating pages in place. If multiple threads access a B-tree at the same time, a thread may see the tree in an inconsistent state. The solution is usually implemented by protecting the tree's data structures with *latches* (lightweight locks). This is not an issue with log structured approaches since all the merging happens in the background without interfering with incoming queries.

##### B-tree optimizations [#](#b-tree-optimizations)

Different optimizations have been made with B-trees:

- Additional pointers been added to the tree. E.g. a leaf page may have references to its sibling pages to the left and right, this allows scanning keys in order without jumping back to parent pages.
- Some databases use a copy-on-write scheme instead of overwriting pages and maintaining a WAL for crash recovery. What this means is that a modified page is written to a different location, and a new version of the parent pages in the tree is created, pointing at the new location.

*So, page structured storage engines are organized into fixed-size pages. These pages are all part of a tree called b-tree.*

SQLite, for example, has a btree for every table in the database, as well as a btree for every index in the database. For the indexes , the key stored on a page is the column value of the index, while the value is the row id where it can be found. For the table btree, the key is the rowid while I believe the value is all the data in that row: [https://jvns.ca/blog/2014/10/02/how-does-sqlite-work-part-2-btrees/](https://jvns.ca/blog/2014/10/02/how-does-sqlite-work-part-2-btrees/)

[**https://hackernoon.com/fundamentals-of-system-design-part-3-8da61773a631**](https://hackernoon.com/fundamentals-of-system-design-part-3-8da61773a631)

##### Comparing B-Trees and LSM-Trees [#](#comparing-b-trees-and-lsm-trees)

As a rule of thumb, LSM trees are typically faster for writes, while B-trees are thought to be faster for reads. Reads are slower on LSM-trees because they have to check different data structures and SSTables at different stages of compaction.

*Don't forget that in the end, these databases are just abstractions over writing to disk.

##### Advantages of LSM Trees [#](#advantages-of-lsm-trees)

- A B-tree index must write every piece of data at least twice: once to the write-ahead log, and once to the page itself (and perhaps again as pages are split). There's also overhead from having to write an entire page at a type, even if only few bytes in the page change. Log-structured indexes also rewrite data multiple times due to the repeated compaction and merging of SSTables.
    
    ***Write amplification:*** When one write to the database results in multiple writes to the disk over the course of the database's lifetime. This is of particular concern on SSDs, which can only overwrite blocks a limited number of times before wearing out.
    
- LSM-trees are typically able to sustain higher write throughput than B-trees partly because they sometimes have lower write amplification, and also because they sequentially write compact SSTable files rather than having to overwrite several pages in the tree. This is important on magnetic hard drives, where sequential writes are faster than random writes.
    
- LSM-trees can be compressed better, and this often produces smaller files on disk than B-trees. B-trees leave some disk space used due to fragmentation: when a row cannot fit into an existing page or a page is split, some space in a page remains unused(Basically, if the existing space on the page cannot fit a new row, the row will be moved to a page). Sending and receiving smaller files over IO is useful if you bandwidth is limited.
    

On many SSDs, the firmware internally uses a log-structured algorithm to turn random writes into sequential writes on the underlying storage chips, so the impact of the storage engine's write pattern is less pronounced (point 2). Note that lower write amplification and reduced fragmentation is still advantageous on SSDs: representing data more compactly allows more read and write requests within the available I/O bandwidth.

##### Downsides of LSM Trees [#](#downsides-of-lsm-trees)

- A downside of log-structured storage is that the compaction process happening can interfere with the performance of ongoing reads and writes. Storage engines typically try to perform compaction incrementally and without affecting concurrent access, but it can easily happen that a request needs to wait while the disk finishes an expensive compaction operation.
    
- One other issue with compaction arises at high write throughput: The disk needs to share its finite write bandwidth between the initial write (logging and flushing a memtable to disk) and the compaction threads running in the background.
    
    Compaction has to keep up with the rate of incoming writes, even at high write throughput.
    
- A key can exist in multiple places in different segments with LSM trees. This differs from B-trees where a key can only exist in one place. This prospect makes B-trees more appealing for strong transactional semantics - e.g. with b-trees transaction isolation can be implemented by attaching locks on a range of keys to a tree.
    

#### Other Indexing Structures [#](#other-indexing-structures)

We've mainly covered key-value indexes which are like a primary key index, but we can also have secondary indexes. You can typically create several secondary indexes on the same table in relational databases. A secondary index can be constructed from a key-value index.

With secondary indexes, note that the indexed values are not necessarily unique. Several rows (documents, vertices) may exist under the same index entry. This can be expressed in two ways:

- Making each value in the index a list of matching row identifiers.
- Making each key unique by appending a row identifier to it.

Both B-trees and log-structured indexes can be used as secondary indexes.

##### Storing values within the index [#](#storing-values-within-the-index)

The key in an index is the thing that queries search for, but the value can be either:

1.  The actual row (document, vertex) in question
2.  A reference to the row stored elsewhere. The rows in this case are stored somewhere known as a ***heap file*** which stores data in no particular order (could be append-only, or may keep track of deleted rows in order to overwrite them with new data later). This approach is common because it avoids duplicating data in the presence of several secondary indexes. Each index just references a location in the heap file.

##### Approach 2 - Heap file [#](#approach-2---heap-file)

The heap file approach can be efficient when updating a value without changing the key, provided the new value is not larger than the old value. If it is larger, the record might need to be moved to a new location in the heap where there is enough space. When this happens, either all indexes need to be updated to point to the new heap location of the record, or a **forwarding pointer** is left behind in the old heap location.

##### Approach 1 - Actual row [#](#approach-1---actual-row)

In some cases, the hop from the index to the heap file is too much of a performance penalty for reads, so the indexed row is stored directly within the index. This is known as a ***clustered index.*** In MySQL's InnoDB storage engine, the primary key of a table is always a clustered index, and secondary indexes refer to the primary key (rather than a heap file).

##### Covering Index [#](#covering-index)

There's a compromise between a clustered index (storing all row data within the index) and a nonclustered index(storing only references to the data within the index) which is known as a *covering index* or *index with included columns,* which stores *some* of a table's columns within the index. With this approach, some queries can be answered using the index alone).

##### Multi-column indexes [#](#multi-column-indexes)

We've only dealt with indexes which map a single key to a value so far. We need more than that to query multiple columns of a table, or multiple documents simultaneously.

The most common type of multi-column index is a **concatenated index.** This type of index combines several fields into one key by appending the columns. This kind is useless if you only want to search for the values for one of the columns. The columns should be in the order of the common search queries/pattern, because the index will sort by the first column.

Another approach is a **multi-dimensional index.** These kind are a more general way of querying several columns at once, which is useful for geospatial data, for example. Say you want to perform a search for records between both a longitude range **and** a latitude range, an LSM tree or B-tree cannot answer that efficiently. You can get all the records within a range of latitudes (but at any longitude), and within a range of longitudes, but not both simultaneously.

An option is to translate a two-dimensional location into a single number using a space-filling curve(?) and then use a regular B-tree index.

##### Full-text search and fuzzy indexes [#](#full-text-search-and-fuzzy-indexes)

The indexes we've discussed so far assume that we have exact data, and we know the exact values of a key, or a range of values of a key with a sort order. For dealing with things like searching *similar* keys, such as misspelled words, we look at *fuzzy* querying techniques.

**Levenshtein automaton:** Supports efficient search for words within a given edit distance.

##### Keep everything in memory [#](#keep-everything-in-memory)

The data structures discussed so far provide answers to the limitations of disks. Disks are awkward to deal with compared to main memory. With both magnetic disks and SSDs, data on disk must be laid out carefully to get good read and write performance. Disks have 2 significant advantages over main memory though:

- They are durable. Content is not lost if power is turned off.
- They have a lower cost per gigabyte than RAM.

There have been developments of *in-memory databases* lately, especially since RAM has become cheaper and many datasets are not that big so keeping them in memory is feasible.

In-memory databases aim for durability by:

- Using special hardware (battery-powered RAM)
- Writing a log of changes to disk
- Writing periodic snapshots to disk
- Replicating the in-memory state to other machines.

VoltDB, MemSQL and Oracle TimesTen are in-memory databases with a relational model.

Interestingly, the performance advantage of in-memory databases is not due to the fact that they don't need to read from disk. A disk-based storage engine may never need to read from disk if there's enough memory, because the OS caches recently used data blocks in memory anyway. Rather, they can be faster because there's no overhead of encoding in-memory data structures in a form that can be written to disk.

Besides performance, an interesting area for in-memory databases is that they allow for the use of data models that are difficult to implement with disk-based indexes. E.g. Redis offers a db-like interface to data structures such as priority queues and sets.

Recent research indicates that in-memory database architecture can be extended to support datasets larger than the available memory, without bringing back the overheads of a disk-centric architecture. This approach works by evicting the least recently used data from memory to disk when there's not enough memory, and loading it back into memory when it's accessed again in the future.

#### Transaction Processing or Analytics? [#](#transaction-processing-or-analytics%3F)

Transaction: A group of reads and writes that form a logical unit.

**OLAP**: Online Analytics Processing. Refers to queries generally performed by business analytics that scan over a huge number of record and calculating aggregate statistics.

**OLTP:** Online Transaction Processing. Interactive queries which typically return a small number of records.

In the past, OLTP-type queries and OLAP-type queries were performed on the same databases. However, there's been a push for OLAP-type queries to be run on **data warehouses.**

##### Data Warehousing [#](#data-warehousing)

A data warehouse is a separate DB that analysts can query without affecting OLTP operations. The data warehouse contains a read-only copy of the data in all the various OLTP systems in the company. Data is extracted from OLTP databases and loaded into the warehouse using an ETL (*Extract-Transform-Load)* process.

It turns out that the indexing algorithms discussed so far work well for OLTP, but not so much for answering analytics queries.

Transaction processing and data warehousing databases look similar, but the latter is optimized for analytics queries. They are both often accessible through a common SQL interface though.

A number of SQL-on-Hadoop data warehouses have emerged such as Apache Hive, Spark SQL, Cloudera Impala etc.

##### Stars and Snowflakes: Schemas for Analytics [#](#stars-and-snowflakes%3A-schemas-for-analytics)

Many data warehouses are used in a fairly formulaic style - *star schema* (or dimensional modelling). The name "star schema" comes from the fact that when the table relationships are visualized, the fact tables is in the middle, surrounded by its dimension tables (these represent the who, what, where, when, how, and why); The connections to the these tables are like the rays of a star.

We also have the *snowflake schema,* where dimensions are further broken down into subdimensions.

#### Column Oriented Storage [#](#column-oriented-storage)

In most OLTP databases, storage is laid out in a *row-oriented* fashion: all the values from one row of a table are stored next to each other. Document databases are similar: an entire document is typically stored as one contiguous sequence of bytes.

Analytics queries often access millions of rows, but few columns.

The idea behind *column-oriented storage* is straight forward: don't store all the values from one row together, but store all the values from each *column* together instead. If each column is stored in a separate file, a query only needs to read and parse the columns that it is interested in, which can save work.

The column-oriented storage layout relies on each column file containing the rows in the same order.

##### Column Compression [#](#column-compression)

In addition to loading only the columns from disk that are required for a query, we can reduce the demands on disk throughput by compressing data. Column-oriented storage lends itself well to compression. Different compression techniques can be used such as *Bitmap encoding.* The number of distinct values in a column is often small compared to the number of rows. Therefore, we can take a column with *n* distinct values and turn it into *n* separate bitmaps: one bitmap for each distinct value, with one bit for each row. The bit is 1 if the row has tat value, and 0 if not.

So I believe that in column oriented storage, each 'file' for a column has one row, where the number of columns for that row will be the number of rows in a standard row-wise table, and columns contain the rows in the same order.

##### Column-oriented storage and column families [#](#column-oriented-storage-and-column-families)

Cassandra and Hbase have a concept of *column families*, which differs from being column oriented. Within each column family, they store all the columns from a row together, along with a row key, and do not use column compression.

##### Sort Order in Column Storage [#](#sort-order-in-column-storage)

It doesn't really matter in which order the rows are stored in a column store. It's easiest to store them in the order of insertion, but we can choose to impose an order.

It won't make sense to sort each column individually though because we'll lose track of which columns belong to the same row. Rather, we sort the entire row at a time, even though it is stored by column. We can choose the *columns* by which the table should be sorted.

**Several different sort orders**

[**C-stores**](http://vldb.org/pvldb/vol5/p1790_andrewlamb_vldb2012.pdf) provide an extension to sorting on column stores. Different queries benefit from different sort orders, so why not store the same data sorted in *several different ways?*

**Writing to Column-Oriented Storage**

Writes are more difficult with column-oriented storage. An update-in-place approach, like B-trees use, is not possible with compressed columns. To insert a row in the middle of a sorted table, you would likely have to rewrite all the column files.

Fortunately, a good approach for writing has been discussed earlier: LSM-trees. All writes go to an in-memory store first, where they are added to a sorted structure and prepared for writing to disk. It doesn't matter whether the in-memory store is row-oriented or column-oriented.

Further reading:

Oracle internals: [https://stackoverflow.com/a/40740893](https://stackoverflow.com/a/40740893)

How indexes work: [https://stackoverflow.com/questions/1108/how-does-database-indexing-work](https://stackoverflow.com/questions/1108/how-does-database-indexing-work)

SQL Server indexes: [https://sqlity.net/en/2445/b-plus-tree/](https://sqlity.net/en/2445/b-plus-tree/)

How data is stored on disk: [https://www.red-gate.com/simple-talk/sql/database-administration/sql-server-storage-internals-101/](https://www.red-gate.com/simple-talk/sql/database-administration/sql-server-storage-internals-101/)
