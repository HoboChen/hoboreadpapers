## [Bigtable: A Distributed Storage System for Structured Data](http://static.usenix.org/event/osdi06/tech/chang/chang.pdf)

OSDI 2006.

This big table note is written by Casesium Liu.

Bigtable is a **distributed** storage system as a *NoSql* database for managing **structured data** that is designed to **scale to a very lage size of data and thousands of machines**. 

Bigtable has achieved several goals: **wide applicability**, **scalability**, **high performance**, and **high availability**. It doesn't support a **full relational data model**. Instead, it:

- Provides clients with a simple data model that supports dynamic control over data layout and format.
- Data is indexed using row and column names what can be arbitrary strings.
- Treats data as uninterpreted strings.
- Allows clients to reason about the locality properties of the data represented in the underlying storage.
- Let clients dynamically control whether to serve data out of memory or from disk.

## Data Model

Bigtable is a **sparse, distributed, persitent, multi-dimentional sorted map**. It is indexed by a row key, column key and a timestamp. 

### Rows

The row keys in a table are arbitrary strings (currently up to 64KB). Each read/write on a row key is **atomic**.

Bigtable maintains data in lexicographic order by row key. The row range for a table is dynamically partitioned. And each row range is called a *tablet*. *Tablet* is the unit of distribution and load balancing. So read short row ranges are efficient and typically require communication with only a small number of machines.

### Column Families

Column keys are grouped into sets called *column family*, and column family is the basic unit of access control. All data stored in the same column family must be the same type. A column key is named using *family:qualifier* format.

### Timestamps

Each cell in Bigtable can contain multiple versions of the same data; these versions are indexed by *timestamp*, it is 64-bit integers. Timestamp can be assigned by Bigtable or explicitly assigned by client applications. If assgined by client applications, applications need to take care of collisions of timestamp.

Bigtable provides 2 ways to garbage-collet cell versions automatically to make the management of versioned data less onerous:

- Client specify the last *n* verions of a cell be kept.
- Or only the new-enough versions be kept.

## Building Blocks

Bigtable use **GFS** (Google File System) to store log and data files. 

The Google **SSTable** file format is used internally to store Bigtable data. An SSTable provides a **persistent**, **ordered immutable** map from keys to values, and both keys and values are arbitrary byte strings. Internally, each SSTable contains a sequence of blocks (typically each block is 64KB in size but is configurable). A block index (stored at the end of the SSTable) is used to locate blocks; the index is loaded into memory when the SSTable is loaded into memory.

Bigtable relied on a highly-available and persistent distributed lock service called **Chubby**. Bigtable uses Chubby for:

- To ensure that there is almost 1 active master at anytime.
- To store the bootstrap location of Bigtable data.
- To discover tablet servers and finalize tablet server deaths.
- To store Bigtable schema information.
- To store access control lists.

So if Chubby becomes unavailable for an extended period of time, Bigtable becomes unavailable.

## Implementation

The Bigtable implementation contains 3 major components:

- A library that is linked into every client.
- One master server.
- Many tablet severs: each tablet server manage a set of tablets.

And clients data does not move through the master: clients communicate directly with tablet servers for reads and writes. And Bigtable doesn't rely on the master for tablet location information, so most clients never communicate with master. As a result, the master is **lightly loaded** in practice.

### Tablet Location

Is is a three-level hierarchy to store tablet location information:

![Tablet Location Hierarchy](.\images\tablet-location-hierarchy.pdf)

The first level is a file stored in Chubby that contains the location of the *root tablet*. The root tablet contains the location of all tablets in special *METADATA* table. Each METADATA tablet contains the location of a set of user tablets, and root tablet never split.

The client library caches tablet locations.

### Tablet Assignment

Each tablet is assigned to one tablet server at a time, and the master is in charge of this work: 

- keeps trace of the live tablet servers.
- the current assignment of tablets to tablet servers.
- Assign the unassigned tablet to an available tablet server.
- Detecting when a tablet server is no longer serving its tablet, reassign those tablets as soon as possible.

#### The life time of tablet server

Bigtable use Chubby to keep track of tablet servers. When a tablet server starts, it creates and acquires an exclusive lock (a uniquely-named file in specific Chubby directory). A tablet server stops serving its tablets if it loses it execlusive lock.

A tablet server will attempt to reacquire an exclusive lock on its file as long as the file still exists. If the file no longer exist, the tablet server will never be able to serve again, so it kills itself.

whenever a tablet server terminates (e.g., because the cluster management system is removing the tablet server's machine from the cluster), it attempts to release its lock so that the master will reassign its tablets more quickly.

#### Master reassign tablets

Master will reassign tablets when detecting a tablet server is nolonger serving its tables. Detection method: 

1. Decide whether tablet server is alive: the master will first periodically asks each server for the status of its lock. If a tablet server reports that it has lost its lock, or the master was unable to reach a server during its last several attempts.
2. Decide whether Chubby is alive: the master attempts to acquire an exclusive lock on the server's file. If the master is able to acquire the lock, then Chubby is live and the tablet server is either dead or having touble reaching Chubby.
3. Reassign the tablets: the master needs to ensure that the tablet server can never serve again by deleting its server file. Once a server's file has been deleted, the master can move all the tablets that were previously assigned to that server into the set of unassigned tablets.

#### Master failure

The master kills it self if its Chubby session expires.

#### The entire procedure

When a master is started by the cluster management system:

1. Grab master lock: the master grabs a unique master lock in Chubby.
2. Get live server lists: the master scans the servers directory in Chubby to find the live servers.
3. Get current assignment: the master communicates with every live tablet server to discover what tablets are already assigned to each server.
4. The master scans the METADATA table to learn the set of tablets. Whenever this scan encounters a tablet that is not already assigned, the master adds the tablet to the set of unassigned tablets. And master cannot scan METADATA table until the METADATA table is assigned to a tablet server, so master will add root table to unassigned tablets if root table isn't assigned before starting this step.

### Tablet Serving

The representation of tablet is shown in graph 4.

![Tablet representation](.\images\tablet-representations.pdf)

Tablet using LSM architecture, there are 3 parts of it:

- SSTable files: it's the data file and stores in GFS.
- Tablet log: it's the log file and stores in GFS.
- Memtable: it's the cache of tablet log with sorted structure and stores in memory. It's used to speed up the read operation.

For each read/write operation, the server will check: whether it's well-formed and authorized to perform the mutation. Authorization is performed by reading the list of permitted writes from a Chubby file and t almost always hit the cache.

For **write** operation: it first writes to the tablet log, and then insert it to the memtable.
For **read** operation: it needs to read both a sequence of SSTables and memtable.

### Compactions

There are 2 compactions can happen:

- *Minor compaction*: **it compacts the memtable into a new SSTable and writen to GFS.** With the size of memtable grows and reaches a threshold, the memtable is frozen and a new memtable is created, the frozen memtable is converted to an SSTable and writen to GFS. It has 2 goals:
    - Reduce the usage of memory.
    - Reduce the amount of data that has to be read from the commit log during recovery if this server dies.

- *Major compaction*: **it merge several SSTables and the memtable into exactly one SSTable**. Because each minor compaction creats a new SSTable, so read operations might need to merge updates from arbitrary number of SSTables if the operation keeps unchecked. So Bigtable executes a *merging compaction* periodically in the background.

## Refinements

BigTable requires a lot of refinements to achieve the high performance, availability, and reliability.

### Locality groups

Clients can group multiple column families together into a locality group. A seperate SSTable is generated for each locality group in each tablet. And some useful tuning parameters can be specified on per-locality groups basis. For example, locality group can be declared to be in-memory. It often used in the combination of **Compression**.

### Compressoin

Clients can control whether or not SSTables for a locality group are compressed, and if so, which compression format is used. Bigtable emphasized more in speed instead of space reduction.

### Caching for read performance

Tablet servers use two levels of caching:

- *Scan Cache*: a higher-level cache that caches the key-value pairs returned by the SSTable interface to the tablet server code.
- *Block Cache*: a lower-level cache that caches SSTables blocks that were read from GFS.

### Bloom filters

It's a cache for keys. Tells: whether the key exists in a specified SSTable. It's used to reduce the disk seek. A read operation has to read from all SSTables that make up the state of a tablet, and it needs a lot of disk accesses. Clients can specify a Bloom filters to be created for SSTables in a particular locality group. It allows to ask: whether an SSTable might contain any data for a specified row/column pair, and it also implies that most lookups for non-existent rows or columns do not need to touch disk.

### Commit-log implementation

For a tablet server, all tablets share 1 commit log because if 1 tablet share 1 commit log, there can be a very large number of files would be written concurently in GFS and causing large number of disk seeks. But share 1 commit log can **complicate the recovery process of a tablet server**. Because the recovered tablets can be moved to a large number of other tablets servers and commit log are co-mingled in a physical log file. If 1 tablet server read the full commit log file and apply just the entries need for the tablets it needs, then n times will be read if the tablets is assigned to n other tablet servers. Bigtable avoid duplicating log reads by **sorting the commit log entries** in order of the keys `<table, row name, log sequence number>`. And sort is splitted into segments in the aim of parallelizing the sorting on different tablet servers coordinated by master.

### Speeding up tablet recovery
Before tablet really unload from a tablet server, it does 2 minor compactions of the memtable on itself instead of the target tablet server consume the commit log to recovery. If master moves a tablet from 1 tablet server to another, the source tablet server first does a minor compaction on the tablet (because memtable compacts quickly than commit log in disk). And after finishing this compact, the tablet server stops serving the tablet. Before it actually unloads the tablet, the tablet does another minor compaction to eliminate any remainning uncompacted state in the tablet server's log that arrived while the first minor compaction was been performed (usually very fast).

### Exploiting immutability

Bigtable system has been simplified by the fact that all of the SSTable are immutable, ahd permanently removing deleted data is transformed to garbage collection. The only mutable data structure that is accessed by both reads and writes is the memtable, and Bigtable make each memtable copy-on-write and allow reads and writes to proceed in parallel.

## Questions

1. Why Bigtable lock tablet server? It is used to judge the status of server?
2. What does a tablet server own? The tablet server has the tablets physically located on it or its an abstraction. Real data is in GFS? And how does the tablet assignment work? Only modify the pointer or do data movement? I think it's the first.
3. If master fails, who choose master? Chubby? cluster management system?
4. How many memtable exists in a server? Only 1?
5. The minor compaction can also speed up the read operation?
6. How Bigtable use SSTable to store key/values, because it includes columns. What happens when we need to find a key?
7. How Bigtable promise the automatic modification on a single row.
8. The whole process when clients want to read a key.
9. If a tablet server is down, several tablets can becomes unavailable for some minutes?

## Summary

Bigtable是一个高性能，高扩展，高可用，基于行列模型的NoSql数据库**系统**。受限于它采用LSM数据结构，因为它适用于写多读少的场景。Bigtable是single master的，它实现了分布式系统中的CP。其采用row key, column key和timestamp作为index，Bigtable保证**单行事务**。

Bigtable的数据模型：

- Rows：Bigtable会对row key按照字典序排序，并会对row按照范围进行partition，每一个partition叫做*tablet*。**tablet是Bigtable做distribution和load balance的最小单位**。
- Columns：column keys会被分到不同的*column families*。Column key的以`family:qualifier`来访问。
- Timestamps：Bigtable的每个cell都会保存同一份数据的不同版本，这些版本用timestamp作为index。Bigtable提供了对版本的垃圾回收机制。

Bigtable是一个shared-storage的系统。其底层采用；

- **GFS**作为存储系统，存储log files和data files。
- **Google SSTable**作为存储data file的数据结构，SSTable是一个持久化的，有序的，不可变的从key到value的map。
- **Chubby**提供高可用的，持久的分布式锁服务。Bigtable依赖于Chubby：
  - 保证Bigtable的single master。
  - 存储Bigtable的root table的地址（bootstrap location）。
  - 发现tablet servers和确定tablet servers的存活状态。
  - 存储Bigtable的schema信息。
  - 存储access control lists。

Bigtable的实现包括3大部分：library，master和tablet servers：

- 一个tablet server管理多个tablets，client直接和tablet server交互来进行读写tablets。
- Master负责：
  - 分配tablets给tablets servers，一个tablet只能分配给一个tablet server来让其进行管理；
  - 当tablet server死去时，需要将死去的tablet server上的tablets进行重新分配。
  - Master还需诊断自身的存活状态。

Tablets的位置信息采用了3层结构：

- Chubby file：存储root tablet的位置信息。
- Root tablet：存储metadata tablets的位置信息。
- Metadata tablets：存储其他所有的tablets的位置信息。

Tablets采用LSM作为存储机制，它限制了Bigtable的使用场景（写多读少）。该结构的优点在于将磁盘的随机写转化成了顺序写来提高了写操作的性能。而读操作需要同时访问memtable和SSTable，因此降低了读操作的性能。
一个tablet由多个SStable作为其data files，tablet log作为其commit log，memtable是由commit log构建出的在内存中的有序的kv map来加速读操作。Bigtable的一个读操作需要先写commit log，再写memtable；一个读操作需要同时读memtable和SSTable files。

Tablets会由固定时机的两个compactions：

- minor compaction将memtable compact成SSTable来减少内存的使用量和减少tablet recovery时的操作。
- major compaction将多个SSTable compact成一个来减少每次读操作需要访问的SSTable的数量。

Bigtable用了很多的优化才达到了高性能，高可用性，其优化的主要目的是：尽可能用内存访问来替代磁盘的访问，减少磁盘的偏移，减小数据量。

这些优化操作如下：

- locality group：Bigtable会为每个locality group生成一个单独的SSTable，在同一个locality group内读只需要访问一个SSTable。客户端还可以指定很多额外的针对locality group的参数。
- Compression：Bigtable更关心压缩/解压速度而不是压缩率。
- Cache：Bigtable的*scan cache*缓存了SSTable返回的KV paris；其*Block cache*将从GFS中读出的SSTable blocks缓存（缓存到tablet servers的disk还是memory？）。
- Bloom filter：告诉我们一个SSTable是否包含某个key来减少读操作访问的SSTable。
- Commit log：一个tablet server上的多个tablets共享一份commit log而不是一个tablet使用一个commit log，来减少GFS上因为并发写多个文件而发生的大量磁盘偏移。其代价时使tablet recovery变得更复杂和开销更大，Bigtable也使用排序等手段来降低tablet recovery的开销。
- 加速tablet recovery：因为内存读比磁盘读更快，因此tablet server在unload一个tablet并让一个新的tablet server去load这个tablet并且从commit log恢复之前，会先使用memtable做一次minor compaction；第二次minor compaction是为了compact第一次minor compaction的过程中发生的写操作。
- Immutablity：Bigtable中唯一可变的数据结构是memtable，Bigtable采用copy-on-write来使对memtable的读写并行。