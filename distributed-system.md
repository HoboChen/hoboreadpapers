---
linkcolor: red
urlcolor: purple
citecolor: blue
...

# 早年

## [MapReduce: Simplified Data Processing on Large Clusters](https://www.usenix.org/legacy/events/osdi04/tech/full_papers/dean/dean.pdf)

OSDI 2004。

## [Bigtable: A Distributed Storage System for Structured Data](http://static.usenix.org/event/osdi06/tech/chang/chang.pdf)

OSDI 2006。

## [The Google file system](http://www.academia.edu/download/28578273/the.google.file.system.pdf)

2003年SOSP的论文。

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
- 

# 近生代

## [Spanner: Google’s globally distributed database](https://dl.acm.org/ft_gateway.cfm?id=2491245&type=pdf)

OSDI 2012。
