## [Dynamo: Amazon's Highly Available Key-value Store](http://www.read.seas.harvard.edu/~kohler/class/cs239-w08/decandia07dynamo.pdf)

Dynamo sacrifices consistency under certain failure scenarios.
It makes extensive use of object versioning and application-assited conflict resolution in a manner that provides a novel interface for developers to use.

### Introduction

There are always a small but significant number of server and network compnents that are failing at any given time.

Dynamo is used to manage the state of services that have very high reliability requirements and need tight control over the tradeoffs between availability, consistency, cost-effectiveness and performance.

Data is partitioned and replicated using consistent hashing, and consistency is facilitated by object versioning. The consistency among replicas during updates is maintained by a quorum-like technique and a decentralized replica synchoronization protocol.

### Background

Most of the services only store and retrieve data by primary key and do not require the complex querying and management functionality offered by an RDBMS.

Query Model: Key-Value, targets applications that need to store objects that are relatively small (usually less than 1MB).

ACID: Dynamo targets applications that operate with weaker consistency (C in ACID) if this results in high availability. Dynamo does not provide any isolation guarantees.

Efficiency: TODO

Other assumptions: Internal service, targets a scale of up to hundreds of storage hosts.

At Amazon, SLAs are expressed and measured at the 99.9th percentile of the distribution.

Dynamo is designed to be an eventually consistent data store.

An important design consideration is to decide *when* to perform the process of resolving update conflicts.
The next design choice is *who* performs the process of conflict resolution.

Other key principles embraced the design:

1. incremental scalability
1. symmetry
1. decentralization
1. heterogeneity

### Related Work

TODO

### System Architecture

#### System Interface

- `get(key)`
- `put(key, context, object)`

#### Partitioning Algorithm

Dynamo's partitioning scheme relies on consistent hashing, the output range of a hash function is treated as fixed circular space or "ring".

The principle advantage of consistent hashing is that departure or arrival of a node only affects its intermediate neighbors and other nodes remain unaffected.

Each node of Dynamo gets assigned to multiple points in the ring, which are called "virtual nodes". Advantages:

1. The load will disperse across all current nodes when a node join or leave.
2. Number of virtual nodes of one machine can be ajusted for heterogeneity.

#### Replication



简单总结：

- Dynamo 是一个最终一致的分片KV数据库，用一致性换可用性；选了CP的道路。
- Dynamo 把解决冲突（consistency）放在了读的时候，并且把处理逻辑上推给应用层。
