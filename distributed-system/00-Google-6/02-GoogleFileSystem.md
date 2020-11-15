## [The Google file system](https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf)

SOSP 2003.

GFS provides fault tolerance while running on inexpensive commodity hardware, and it delivers high aggregate performance to a large number of clients.

Traditional choices are reexamined and different points of design space are explored radically.

1. Component failures are the norm rather than the exception.
1. Files are huge by traditional standards as multi GB files are common.
1. Most files are mutated by appending new data rather than overwriting existing data.
1. Co-designing the applications and the file system API benefits the overall system.

### Design Overview

Assume:

1. The system is built from many inexpensive commodity components.
1. The system stores a modest number of large files.
1. The workloads primarily consist of 2 kinds of read: large streaming reads and small random reads.
1. The workloads have many large sequential writes. And once written, files are seldom modified again.
1. The system must be efficient when appending concurrently with multiple clients.
1. High sustained bandwidth is more important than low lantency.

GFS provides a familiar file system interface but it does not implement fully a POSIX API.
GFS support the usual operations like `create`, `delete`, `open`, `close`, `read` and `write`. Moreover, GFS has `snapshot` and `record append` operations.

#### Architecture

![GFS-Architecture](./resource/gfs-fig-0.png)

A GFS cluster consists of a single *master* and multiple *chunkservers* and is accessed by multiple *clients*.

Files are divided to fixed-size *chunks*. Each chunk is identified by an immutable, globally unique 64bit *chunk handle* assigned by *master* at the creation.
For reliability, each chunk is replicated on multiple chunkservers. By default, three replicas are stored.

The master maintains all file system metadata including:

1. file and chunk namespaces
1. mapping from files to chunks
1. locations of each chunk's location

Clients interact with the master for metadata operations, but all data-bearing communication goes directly to the chunkservers.

Neither the client nor the chunkserver caches file data(not intended to, but may be cached by filesystem), but clients do cache metadata.

#### Chunk Size

64MB is much larger than typical file system block size.

Pros:

1. Reducing times of interacting with the master.
1. Reducing network overhead by keeping a persistent TCP connection.
1. Enabling keep all the metadata in memory.

Cons:

1. Hot spots.

#### Metadata

The master does not keep a persistent record of which chunkservers have a replica of a given chunk.
It simply pools chunkservers for that information at startup, which eliminated the problem of keeping the master and chunkservers in sync as chunkservers joins, leaves, fail, restart, etc.

The operation log contains a historical record of critical metadata changes.
Operation log is critical, so it is stored reliably and changes is not visible to clients until metadata changes are made persistent.
It does not respond to a client operation until flushing to both local and remote disk completed.

The master recovers its file system state by replaying the operation log.
The master checkpoints its state whenever the log grows beyond a certain size.
The checkpoint is in a compact B-tree like from.

#### Consistency Model

##### Guarantees by GFS

File namespace mutations are atomic, they are handled exclusively by the master.
The master's log defines a global total order of these operations.

The state of a file region after a data mutation depends on:

1. type of mutation
2. succeeds or fails
3. whether there are concurrent mutations

States are:

1. A file region is **consistent** is all clients will always see the same data, regradless of which replicas they read from.
1. A file region is **defined** after a file data mutation if it is consistent and clients will see what mutation writes in its entirety. When a mutation succeeds without interference from concurrent writers.
1. A file region is **undefined but consistent** after concurrent successful mutations : all clients see the same data, but it may not reflect what any one mutation has written.
1. A file region is **inconsistent** hence also **undifined** after a failed mutation.

Data mutations may be *writes* or *record appends*:

1. A write causes data to be written at an application-specified file offset.
1. A record append causes data to be appended **atomically at least once**.

After a sequence of successful mutations, the mutated file region is guaranteed to be defined and contain the data written by the last mutation. GFS achieves this by:

1. applying mutations to a chunk in the same order on all its replicas
1. using chunk version numbers to detect stale replicas

### System Interactions

The master grants a chunk lease to one of the replicas, which is called *primary*.
The primary picks a serial order for all mutations to the chunk, and all replicas follow this order.
The lease machanism is designed to minimize management overhead at the master.

![GFS-WriteFlow](./resource/gfs-fig-1.png)

Writes:

1. Client asks the master which chunkserver holds the current lease and the location of other replicas.
1. Master replies.
1. Client pushes the data to all the replicas; in any order.
1. All replicas acknowledged, then the client sends a write request to primary, which idetifies the data pushed earlier to all of the replicas. The priamry assigns consecutive serial numbers to all the mutation it receives, which provides the necessary serialization.
1. The primary forwards the write request to all other replicas.
1. Other replicas say completion.
1. The primary replies to client. Any errors encountered at any of the replicas are reported to the client.

#### Data Flow

Flow of data and flow of control are decoupled to use the network efficiently.
Data is pushed linearly, like a pipeline.
Pipelining is helpful as dull-duplex links are used.

#### Atomic Record Appends

GFS provides an atomic append operation called *record append*, which the client only provides data and GFS decides the offset. Record append is heavily used.

Like the write flow, but there is extra logic. After the client pushing data to all replicas, it will send requests to the primary. The primary will check to see if appending the record to the current chunk would cause the chunk to the maximum size(64MB).
If so, it pads the chunk to 64MB, and tell other replicas to do so and then replies to the client that the operation should be retried on the next new chunk. Record append is restricted to be at most 1/4 of 64MB to deal with fragmentation.

If a record append fails at any replica, the client retires the operation.
GFS does not guarantee that all replicas are bytewise identical. And with padding, if the atomic append operation is success, the data must have been written at the same offset of all chunks.

#### Snapshot

Standard copy-on-write techniques are used to implement snapshots.

## Master Operation

### Namespace Management and Locking

Many master operations like snapshot can take a long time, so locks over regions of the namespace are used to ensure proper serialization. 

GFS does not have a per-directory data structure that lists all files in that directory. Nor does it support aliases for the same file or directory. GFS logically reproesents its namespace as a lookup table mapping full pathnames to metadata.

Each node in the namespace tree, either an absolute file name or an absolute directory name, has an associated read-write lock.
Locks are acquired in a consistent order to prevent deadlock; they are first ordered by level in the namespace tree and lexicographically within the same level.

### Replica Placement

The chunk replica placement policy serves two purposes:

1. maximize data reliability and availability
1. maximize network bandwidth utilization

For both, it is not enough to spread replicas across machines.

### Create, Re-replication, Rebalancing

Chunk replicas are created for three reasons:

1. chunk creation
1. re-replication
1. rebalancing

### Garbage Collection

After a file is deleted, GFS does not immediately reclaim the available physical storage which is more dependable and simpler.

### Stale Replica Detection

The master maintains a *chunk version number* to distinguish the up-to-date and stale replicas.
Chunk replicas may become stale if a chunkserver fails and misses mutations to the chunk while it is down.

Whenever the master grants a new lease on a chunk, it increases the chunk version number and informs the up-todate replicas.

The master removes stale replicas in its regular garbage collection.

## Fault Tolerance and Diagnosis

A chunk is broken up into 64KB blocks. Each has a corresponding 32 bit checksum. Like other metadata, checksums are kept in memory and stored persistently with logging, separate from user data.

简单总结：

1. 设计上为以下优化：  
	a. 单次操作读写较多的数据(1MB)；文件也都比较大（GB）  
	b. 一旦写入很少修改  
	c. 并发append  
	d. 吞吐比延迟重要；时刻有硬件故障  
2. 设计概览：  
	a. 有master，master存元数据；是一个CP系统  
	b. 文件被切成chunk，每个chunk有一个全局唯一的id(64bit)；每个chunk 64MB，三副本
3. 为多client流式读/写优化
3. 写路径比较有趣；单个写的序列化在主replica上完成