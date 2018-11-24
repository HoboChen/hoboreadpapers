---
linkcolor: red
urlcolor: purple
citecolor: blue
...

# G-3

## [MapReduce: Simplified Data Processing on Large Clusters](https://www.usenix.org/legacy/events/osdi04/tech/full_papers/dean/dean.pdf)

OSDI 2004。

## [Bigtable: A Distributed Storage System for Structured Data](http://static.usenix.org/event/osdi06/tech/chang/chang.pdf)

OSDI 2006。

## [The Google file system](http://www.academia.edu/download/28578273/the.google.file.system.pdf)

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
