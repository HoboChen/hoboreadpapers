---
linkcolor: red
urlcolor: purple
citecolor: blue
...

# G-3

## [Websearch for a Planet: The Google Cluster Architecture](http://www.eecs.harvard.edu/~dbrooks/cs246-fall2004/google.pdf)

TODO

Google’s architecture features clusters of more than 15,000 **commodity** class PCs with fault-tolerant software. This architecture achieves superior performance at a fraction of the cost of a system built from fewer, but more expensive, high-end servers.

**Energy efficiency and price-performance** ratio are the most important factors which influence the design.
For energy efficiency, power consumption and cooling issues are taxing the limits of available data center power densities. Google is an example of a **throughput-oriented, parallel friendly** workload.

Reliability is provided in software rather than in server-class hardware, and aggregate request throughput rather than peak server response time are tailored for design.

To provide sufficient capacity to handle query traffic, google service consists of multiple clusters distributed worldwide.
Each cluster has around a few thousand machines, and the geographically distributed setup protects us against catastrophic data center failures.
A DNS-based load-balancing system selects a cluster by accounting for the user’s geographic proximity to each physical cluster.

There isn’t that much exploitable instruction-level parallelism (ILP) in the workload.
Measurements suggest that the level of aggressive out-of-order, speculative execution present in modern processors is already beyond the point of diminishing performance returns for such programs.

Google使用很多廉价服务器组成集群来提供服务，架构设计的考量中最重要的是**能效、性价比**。Google是一种典型的**吞吐敏感、易于并行**的服务。相比于使用更贵的企业级硬件，可靠性由软件层保证；相比于单个请求延迟，优化目标是吞吐量。

## [MapReduce: Simplified Data Processing on Large Clusters](https://www.usenix.org/legacy/events/osdi04/tech/full_papers/dean/dean.pdf)

OSDI 2004。

## [Bigtable: A Distributed Storage System for Structured Data](http://static.usenix.org/event/osdi06/tech/chang/chang.pdf)

OSDI 2006。

## [The Google file system](https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf)

SOSP 2003。

### Introduction

Key observations of our application workloads and technological environment:

- Component failures are the norm rather than the exception.
- Files are huge.
- Most files are mutated by appending rather than overwriting.

### Design Review

#### Architecture

A GFS cluster consists of a single *master* and multiple *chunkservers* and is accessed by multiple *clients*.

Files are divided to fixed-size *chunks*. Each chunk is identified by an immutable, globaly unique 64bit *chunk handle* assigned by *master* at the creation.

The master maintains all file system metadata. The master periodically communicates with each chunkserver in *HeartBeat* messages.

#### Single Master

#### Consistency Model

Guarantees:

- File namespace mutations (e.g., file creation) are atomic.
- The state of a file region after a data mutation depends on the type of mutation, whether it succeeds or fails, and whether there are concurrent mutations.

# 分布式存储

## [Ceph: A Scalable, High-Performance Distributed File System](https://www.usenix.org/legacy/event/osdi06/tech/full_papers/weil/weil.pdf)

CEPH, 2006年OSDI。

## [Spanner: Google’s globally distributed database](https://dl.acm.org/ft_gateway.cfm?id=2491245&type=pdf)

OSDI 2012。

## [PolarFS: An Ultra-low Latency and Failure Resilient Distributed File System for Shared Storage Cloud Database](http://www.vldb.org/pvldb/vol11/p1849-cao.pdf)

VLDB 2018。

# 分布式锁

## [The Chubby lock service for loosely-coupled distributed systems](http://static.usenix.org/legacy/events/osdi06/tech/full_papers/burrows/burrows.pdf)

# 分布式调度

## [Large-scale cluster management at Google with Borg](https://dl.acm.org/ft_gateway.cfm?ftid=1563633&id=2741964)
