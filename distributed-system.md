---
linkcolor: red
urlcolor: purple
citecolor: blue
...

# G-5

## [Websearch for a Planet: The Google Cluster Architecture](http://www.eecs.harvard.edu/~dbrooks/cs246-fall2004/google.pdf)

Google’s architecture features clusters of more than 15,000 **commodity** class PCs with fault-tolerant software. This architecture achieves superior performance at a fraction of the cost of a system built from fewer, but more expensive, high-end servers.

**Energy efficiency and price-performance** ratio are the most important factors which influence the design.
For energy efficiency, power consumption and cooling issues are taxing the limits of available data center power densities. Google is an example of a **throughput-oriented, parallel friendly** workload.

Reliability is provided in **software** rather than in server-class hardware, and **aggregate request throughput** rather than peak server response time are tailored for design.

To provide sufficient capacity to handle query traffic, google service consists of multiple clusters distributed worldwide.
Each cluster has around a few thousand machines, and the geographically distributed setup protects us against catastrophic data center failures.
A DNS-based load-balancing system selects a cluster by accounting for the user’s geographic proximity to each physical cluster.
Processing of each request is entirely local to that cluster, and there is load balancer to dispatch the request to one of the Google Web Servers(GWS).

First phase of the processing the query is to consult the inverted index that maps each query word to a matching list of documents and then the relevance score is computed. The inverted index is sharded by dividing the documents.

Second phase is to take the list of docids and computing the actual title and uniform resource locator of thse documents. Also, the documents is distributed randomly to smaller shards and there is a load balancer.

Most access to the inex and other data structures involved in answering a query are read-only, and updates are relatively infrequest.

Machines older than three years are so much slower than current-generation machines that it is difficult to achieve proper load distribution and configuration in clusters containing both types.

There is **not** that much exploitable instruction-level parallelism in the workload.
Measurements suggest that the level of aggressive out-of-order, speculative execution present in modern processors is already beyond the point of diminishing performance returns for such programs.

简单总结：

- Google使用很多廉价服务器组成集群来提供服务
    - 架构设计的考量中最重要的是**能效、性价比**
    - Google搜索是一种典型的**吞吐敏感、易于并行**的服务
    - 不使用更贵的企业级硬件，可靠性由软件层保证
    - 不极致优化单个请求延迟，优化目标是总吞吐量
- 一个搜索请求
    - 多个集群遍布全球，根据DNS给某个集群
    - 集群内部使单个GWS负载均衡
    - 倒排索引分片，靠随机划分文档集
    - 每个分片由一个集群提供服务，同样有负载均衡分发给某台机器
    - 靠文档ID去拿真正的文档标题等，同样划分-分片-负载均衡
- 多副本提高服务能力和可用性
    - 绝大多数请求链路只读
    - 灰度更新索引，不保证一致性
- 线上机器通常不会超过3年
- 指令级并行对提升性能没太大帮助

## [MapReduce: Simplified Data Processing on Large Clusters](https://www.usenix.org/legacy/events/osdi04/tech/full_papers/dean/dean.pdf)

OSDI 2004.

MapReduce is a programming model and an associated implementation for processing and generating large dataset.
Many real world tasks are expressible in this model.

- *map* function is to process a key/value pair to generate a set of intermediate key/value pairs.
- *reduce* merges all intermediate values associated with the same intermediate key.

The run-time system takes care of the details of partitioning the input data, scheduling the program's execution across a set of machines, handling machine failures and managing the required inter-machine communication.

The abstraction is inspired by the *map* and *reduce* primitives present in Lisp and many other functioinal languages.

### Programming Model

```go
// i simply copy my code of mit 6.824 lab 1 here
// i do recommend the course

func doMap(
	jobName string, // the name of the MapReduce job
	mapTask int, // which map task this is
	inFile string,
	nReduce int, // the number of reduce task that will be run ("R" in the paper)
	mapF func(filename string, contents string) []KeyValue,
) {
	reduceFiles := make(map[string]map[string][]string) // maps reduceFileName -> reduceFileContent

	b, err := ioutil.ReadFile(inFile)
	if err != nil {
		panic(err)
	}
	for _, val := range mapF(inFile, string(b)) {
		r := ihash(val.Key)
		reduceFileName := reduceName(jobName, mapTask, r%nReduce)
		if reduceFiles[reduceFileName] == nil {
			reduceFiles[reduceFileName] = make(map[string][]string)
		}
		reduceFiles[reduceFileName][val.Key] = append(reduceFiles[reduceFileName][val.Key], val.Value)
	}
	for reduceFileName, reduceFileContent := range reduceFiles {
		b, err := json.Marshal(reduceFileContent)
		if err != nil {
			panic(err)
		}
		reduceFile, err := os.Create(reduceFileName)
		if err != nil {
			panic(err)
		}
		reduceFile.Write(b)
		reduceFile.Close()
	}
}

func ihash(s string) int {
	h := fnv.New32a()
	h.Write([]byte(s))
	return int(h.Sum32() & 0x7fffffff)
}
func doReduce(
	jobName string, // the name of the whole MapReduce job
	reduceTask int, // which reduce task this is
	outFile string, // write the output here
	nMap int, // the number of map tasks that were run ("M" in the paper)
	reduceF func(key string, values []string) string,
) {
	reduceContent := make(map[string][]string)

	for mapTaskIdx := 0; mapTaskIdx < nMap; mapTaskIdx++ {
		reduceFileName := reduceName(jobName, mapTaskIdx, reduceTask)
		b, err := ioutil.ReadFile(reduceFileName)
		if err != nil {
			panic(err)
		}
		mapContent := make(map[string][]string)
		err = json.Unmarshal(b, &mapContent)
		if err != nil {
			panic(err)
		}
		for key, val := range mapContent {
			for _, v := range val {
				reduceContent[key] = append(reduceContent[key], v)
			}
		}
	}
	reduceFile, err := os.Create(outFile)
	if err != nil {
		panic(err)
	}
	enc := json.NewEncoder(reduceFile)
	for key, values := range reduceContent {
		enc.Encode(KeyValue{key, reduceF(key, values)})
	}
	reduceFile.Close()
}
```

### Implementation

![MapReduce-Fig](resource/mapreduce-fig.svg)

The right choice of MapReduce implementation depends on the environment.

1. Split the input files into M pieces of typically 16-64 megabytes. And then drop them to GFS.
1. There are M map tasks and R reduce tasks.
1. A worker who is assigned a map task reads the contents, and it parses and passes each pair to Map function. The intermediate key/value pairs are buffered in memory.
1. Periodically, buffered pairs are written to local disk, which partitioned into R regions by the partitioning function.
1. Reduce worker reads the intermediate data, and it sorts it by the intermediate keys.
1. For each unique key encountered, it passes the key and the corresponding set of intermediate values to the user's Reduce function. The output is to append to the final output file for this reduce partition.
1. When all map tasks and reduce tasks have been completed, the MapReduce job finished.

#### Fault Tolerance

worker:

1. heart beat to check whether worker lives
1. reduce worker renames its temporary output file to final output file **atomically**
1. re-excuted produces the same output; map tasks are re-executed, but reduce tasks not
1. after re-executing one map task, all the reduce workers would be notified

master:

1. simply assume that master will not fail, but write checkpoints periodically

#### Locality

Master use the info of where the GFS stores the replica to schedule tasks to reduce remote IO.

#### Backup Tasks

When a MapReduce opeartion is close to completion, the master schedules backup executions of the remaining *in-progress* tasks.

### Refinements

- partitioning function
- ordering guarantee
- local execution
    - alternative implementation which enables sequetial excution is developed
- status information
- counter

#### Combiner Function

We allow the user to specify an optional *Combiner* function that does partial merging of the data before it is sent over the network.

#### Side-effects

Atomic two-phase commits of multiple output files produced by a single task is not provided. Therefore, tasks that produce multiple output files with cross-file consistency requirements should be deterministic.

### Performance

#### Grep

- $10^{10}$ 100-byte records
- pattern is relatively rare, 92337 times occured
- M=15000, R=1
- 150 sec

#### Sort

- - $10^{10}$ 100-byte records, roughly TB-sort
- final sorted output is written to a set of 2-way replicated GFS files
- a pre-pass MR operation will collect a sample of the keys and use the distribution to compute the split points, which is used in Map phase
- M=15000, R=4000
- 891 sec

### Experience

- first version of MapReduce is written in Feb 2013
- significant enhancements were make in Aug 2013
    - locality optimization
    - dynamic load balancing

简单总结：

- MapReduce要求map,reduce均无后效性
- 无后效性可以隐藏：
    - 并行的细节
    - 错误处理
    - 局部性优化
    - 负载均衡
- 在map,reduce快要结束的时候，启动备份Task来避免长尾
- 网络是瓶颈

## [The Google file system](https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf)

SOSP 2003.

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

![GFS-Architecture](./resource/gfs-fig-0.svg)

A GFS cluster consists of a single *master* and multiple *chunkservers* and is accessed by multiple *clients*.

Files are divided to fixed-size *chunks*. Each chunk is identified by an immutable, globally unique 64bit *chunk handle* assigned by *master* at the creation.
For reliability, each chunk is replicated on multiple chunkservers. By default, three replicas are stored.

The master maintains all file system metadata including:

1. file and chunk namespaces
1. mapping from files to chunks
1. locations of each chunk's location

Clients interact with the master for metadata operations, but all data-bearing communication goes directly to the chunkservers.

Neither the client nor the chunkserver caches file data, but clients do cache metadata.

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
It simply pools chunkservers for that information at startup.

The operation log contains a historical record of critical metadata changes.

Operation log is critical, so it is stored reliably and changes is not visible to clients until metadata changes are made persistent.
It does not respond to a client operation until flushing to both local and remote disk completed.

The master recovers its file system state by replaying the operation log.
The master checkpoints its state whenever the log grows beyond a certain size.
The checkpoint is in a compact B-tree like from.

#### Consistency Model

##### Guarantees by GFS

File namespace mutations are atomic, they are handled exclusively by the master.

The state of a file region after a data mutation depends on the type of mutation, whether it succeeds or fails, and whether there are concurrent mutations.

1. A file region is *consistent* is all clients will always see the same data, regradless of which replicas they read from.
1. A file region is *defined* after a file data mutation if it is consistent and clients will see what mutation writes in its entirety.
1. A file region is *undefined but consistent* after concurrent successful mutations : all clients see the same data, but it may not reflect what any one mutation has written.
1. A file region is *inconsistent* hence also *undifined* after a failed mutation.

Data mutations may be *writes* or *record appends*.

1. A write causes data to be written at an application-specified file offset.
1. A record append causes data to be appended *atomically at least once*.

After a sequence of successful mutations, the mutated file region is guaranteed to be defined and contain the data written by the last mutation. GFS achieves this by:

1. applying mutations to a chunk in the same order on all its replicas
1. using chunk version numbers to detect stale replicas

### System Interactions

The master grants a chunk lease to one of the replicas, which is called *primary*.
The primary picks a serial order for all mutations to the chunk, and all replicas follow this order.
The lease machanism is designed to minimize management overhead at the master.

![GFS-WriteFlow](./resource/gfs-fig-1.svg)

Writes:

1. Client asks the master which chunkserver holds the current lease and the location of other replicas.
1. Master replies.
1. Client pushes the data to all the replicas.
1. All replicas acknowledged, then the client sends a write request to primary, which idetifies the data pushed earlier to all of the replicas. The priamry assigns consecutive serial numbers to all the mutation it receives.
1. The primary forwards the write request to all other replicas.
1. Other replicas say completion.
1. The primary replies to client.

#### Data Flow

Flow of data and flow of control are decoupled to use the network efficiently.
Data is pushed linearly, like a pipeline.
Pipelining is helpful as dull-duplex links are used.

#### Atomic Record Appends

TODO

GFS does not guarantee that all replicas are bytewise identical.

#### Snapshot

TODO

<!-- Guarantees: -->

<!-- - File namespace mutations (e.g., file creation) are atomic. -->
<!-- - The state of a file region after a data mutation depends on the type of mutation, whether it succeeds or fails, and whether there are concurrent mutations. -->

## [The Chubby lock service for loosely-coupled distributed systems](http://static.usenix.org/legacy/events/osdi06/tech/full_papers/burrows/burrows.pdf)

## [Bigtable: A Distributed Storage System for Structured Data](http://static.usenix.org/event/osdi06/tech/chang/chang.pdf)

OSDI 2006.

# 分布式一致性协议

## [In Search of an Understandable Consensus Algorithm](https://www.usenix.org/system/files/conference/atc14/atc14-paper-ongaro.pdf)

Raft is a consensus algorithm for managing a replicated log.
It produces a result equivalent to multi-Paxos.

Raft separates the key elements of consensus, such as leader election, log replication and safty.
It enforces a stronger degree of conherency to reduce the number of states.

### Replcated State Machines

Consensus algorithms typically arise in the context of *replicated state machines*.

Replicated state machines are used to solve a varity of fault tolerance problems in distributed systems, e.g. GFS, HDFS and RAMCloud.
Examples of replcated state machines include Chubby and ZooKeeper.

Replicated state machines are typically implemented using a replicated log. Keeping the replcated log consistent is the job of the consensus algorithm, which is to:

- ensure *safety* under all non-Byzantine conditions
- *available* as long as any majority of the servers operational
- do not depend on timing techniques

### Paxos

Paxos first defines a protocol capable of reaching agreement on a single decision, e.g. a single replicated log entry.
Paxos then combines multiple instances of this protocol to facilitate a series of decisions such as a log(Multi-Paxos).

### Design

Most important goal is *understandablity*.

### Raft

Raft implements consensus by first electing a distinguished *leader*.
The leader accepts log entries from clients, replicates them on other servers, and tells servers then it is safe to apply log entries to their state machines.

Raft includes:

1. leader election
1. log replication
1. safety

#### Raft Basics

A raft cluster contains several servers; five is a typical number, which allows the system to tolerate two failures.
At any time, each server is in one of the states:

1. leader
1. follower
1. candidate

Raft divides time into *terms* of arbitrary length.
Terms are numbered with consecutive integers.
Each term begins with an election.

Raft ensures that is at most one leader in a given term.

Terms act as a logical lock in Raft, which allow servers to detect obsolete information such as stale leaders.
Each server stores a *current terms* number, which increases monotonically over time.

Current terms are exchanged whenever servers communicate. If a candidate or leader discovers that its term is out of date, it immediately reverts to follower state.

Raft servers communicate using RPCs, and the basic consensus algorithm requires:

1. RequestVote RPC, sent by candidates during elections
1. AppendEntries RPC, sent by leader to replicate log entry and to provide a form of heart beat

#### Leader Election

When servers start up, they begin as followers.
A server remains in follower state as long as it receives valid RPCs from a leader or candidate.
Leaders send periodic heartbeats to all followers in order to maintain their authority.

If a follower receives no communication over a period of time called the *election timeout*, then it assumes there is no viable leader and begins an election to choose a new leader.

Raft uses randomized election timeouts to ensure that split votes are rare and that they are resolved quickly.

![Raft-ServerState]()

#### Log Replication

Once a leader has been elected, it begins servicing client requests.

Each client request contains a command to be executed by the replicated state machines.
The leader appends the command to its log as a new entry, then issues AppendEntries RPCs in parallel to each of the other servers to replicate the entry.
Only if the entry is *safely* replicated, the leader applies the entry to its state and returns the result to the client.

Each log stores:

1. state machine command
1. term number given by leader
1. log index

If an entry is replicated on majority of machines, it is called *saftly replicated* or *committed*.
This also commits all proceding entries in the leader's log, including entries created by previous leaders.
Next subsection will discusses more about why it is safe.

The leader keeps track of the highest index it knows to be commited, which is also included in the AppendEntires RPCs.
Once a follower learns an entry is commited, it applies the entry in its local machine.

There are Log Matching Property:

1. If two entries in different logs have the same index and term, they store the same command.
1. If two entries in different logs have the same index and term, logs are indentical in all preceding entries.

By:

1. The leader creates at most one entry with a given log index in a given term.
1. When sending an AppendEntries RPC, the leader includes the index and term of the entry in its log immediately precedes the new entry.

Leader crashes can leave the logs inconsistent.
In Raft, the leader will force the follower's log to duplicate its own.
Next subsection will show why it is safe when coupled with one more restriction.

To bring a follower's log into consistency with its own, the leader must find the lastest log entry where the two logs agree, delete any entries in the follower's log after that and send all the leader's entries after that.

#### Safety

This section completes the Raft algorithm by adding a restriction on which servers may be elected leader.

##### Election restriction

Raft uses the voting process to prevent a candidate from winning an election unless its log contains all committed entries.
A candidate must contact a majority of the cluster in order to be elected, which means that every committed entry must be present in at least  one of those servers.

##### Committing entries from previous terms

If a leader crashes before committing an entry, future leaders will attempt to finish replicating the entry.
However, a leader cannot immediately conclude that an entry from a previous term is commited once it is stored on majority of servers. 

Raft never commits log entries from previous terms by counting replicas.
Only log entries from the leader's current term are commited by counting replicas.

Raft incurs this extra complexity in the commitment rules as log entries retain their original term numbers when a leader replicates entries from previous terms.

##### Safety argument

TODO

#### Timing and availability

$$
broadcastTime << electionTimeout << MTBF
$$

# 分布式存储

## [Ceph: A Scalable, High-Performance Distributed File System](https://www.usenix.org/legacy/event/osdi06/tech/full_papers/weil/weil.pdf)

2006 OSDI.

## [Spanner: Google’s globally distributed database](https://dl.acm.org/ft_gateway.cfm?id=2491245&type=pdf)

OSDI 2012.

## [PolarFS: An Ultra-low Latency and Failure Resilient Distributed File System for Shared Storage Cloud Database](http://www.vldb.org/pvldb/vol11/p1849-cao.pdf)

VLDB 2018.

Recently, decoupling storage from compute has become a trend for cloud computing industry.

1. Hardware of compute / storage nodes can be different.
2. Disks on different storage nodes can form one pool for fragmentation and storage usage concerns.
3. There is no local persistent state on compute nodes(?), which makes it easier and faster for database migration.

*PolarFS* is designed and implemented to prove a better filesystem comparing to the other common products as there does exist trade-off between consistent and performance.

- Distributed filesystems like HDFS and Ceph are found with much higher latency, and have no use of new hardwares such as RMDA or NVMe.
- General-purpose filesystems like ext4 running on bare metal hardwares are lack of scalability, fault tolerence and usage effciency.

*PolarFS* do use the user space network stack and IO stack.
*PolarFS* proposes *ParallelRaft* as Raft seriously impedes the I/O scalability when using extra low latency hardware.

For NVMe and RDMA, traditional techniques such as kernel space disk drivers or kernel space TCP/IP stack become the bottleneck.

PolarFS has 2 layers:

- Layer 1 is storage layer, which manages all the disk resoures and provides a database volume for every database instance.(Q)
- Layer 2 is filesystem layer, which supports file management and is responsible for mutual exclusion and synchronization of concurrent accesses to file system metadata.

For storage layer:

1. A volumn is allocated for each database instance and is composed of chunks. The capacity of volumns ranges from 10GB to 100TB.
1. Chunks are distributed among ChunkServers, whose relica are placed to three distinct ChunkServers.
1. Size of chunks are 10GB, makes PolarSwitch to store all the metadata in memory possible.
1. A chunk is devided into blocks inside the ChunkServer, and each block is set to 64KB.
1. Large chunk makes seprating hot spot chunk impossible, which means aggregate performance of instances of databases is preferrable to max performance of one instance.

Other components:

1. PolarSwitch is daemon process running in the database server machines (compute nodes) which forwards IO requests to machine where the leader chunk locates according to local cache synchronized with PolarCtrl.
1. Each chunkserver process is bound to a dedicated CPU core and a NVMe SSD. WAL will be first written to a 3D-XPoint SSD buffer.
1. Chunkserver replicates the IO requests and form a consensus group.
1. PolarCtrl is the control plane of a PolarFS cluster, and it is depolyed on at lease 3 dedicated machines.

Q:

- `recv()`
- Layer 1
- WAL

# 分布式锁

# 分布式调度

## [Large-scale cluster management at Google with Borg](https://dl.acm.org/ft_gateway.cfm?ftid=1563633&id=2741964)
