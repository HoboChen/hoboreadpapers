## [The Chubby lock service for loosely-coupled distributed systems](http://static.usenix.org/legacy/events/osdi06/tech/full_papers/burrows/burrows.pdf)

OSDI 2006.

### Introduction

Chubby is intended for use within a loosely-coupled distributed system consisting of moderately large numbers of small machines connected by a low latency network.
The purpose of the lock service is to allow its clients to synchronize their activities and to agree on basic information about their environment.
Chubby’s client interface is similar to that of a simple file system that performs whole-file reads and writes, augmented with advisory locks and with notification of various events such as file modification.

GFS uses a Chubby lock to appoint a GFS master server, and Bigtable uses Chubby in several ways, to elect a master, to allow the master to discover the servers it controls and to permit clients to find the server.
Before Chubby was deployed, most distributed systems at Google used ad hoc methods for primary election (when work could be duplicated without harm), or required operator intervention (when correctness was essential).
In the former case, Chubby allowed a small saving in computing effort. In the latter case, it achieved a significant improvement in availability in systems that no longer required human intervention on failure.

Building Chubby was an engineering effort required to fill the needs mentioned above; it was not research.
We claim no new algorithms or techniques. The purpose of this paper is to describe what we did and why, rather than to advocate it.

### Design

#### Ratinale

One might argue that we should have built a library embodying Paxos, rather than a library that accesses a centralized lock service, even a highly reliable one.

A client Paxos library would depend on no other servers (besides the name service), and would provide a standard framework for programmers, assuming their services can be implemented as state machines.
Indeed, we provide such a client library that is independent of Chubby.

A lock service has some advantages over a client library:

1. Our developers sometimes do not plan for high availability in the way one would wish.
2. Many of our services that elect a primary or that partition data between their components need a mechanism for advertising the results, which suggests that we should allow clients to store and fetch small quantities of data, that is, to read and write small files.
3. A lock-based interface is more familiar to our programmers.

And, consensus service, which is not limited to lock service, is not better in above situations (but much more complex).

So design decisions are:

1. We chose a lock service, as opposed to a library or service for consensus.
2. We chose to serve small-files to permit elected primaries to advertise themselves and their parameters, rather than build and maintain a second service.
3. A service advertising its primary via a Chubby file may have thousands of clients. Therefore, we must allow thousands of clients to observe this file, preferably without needing many servers.
4. Clients and replicas of a replicated service may wish to know when the service’s primary changes. This suggests that an event notification mechanism would be useful to avoid polling.
5. Even if clients need not poll files periodically, many will; this is a consequence of supporting many developers. Thus, caching of files is desirable.
6. Our developers are confused by non-intuitive caching semantics, so we prefer consistent caching.
7. To avoid both financial loss and jail time, we provide security mechanisms, including access control.

Chubby itself usually has five replicas in each cell, of which three must be running for the cell to be up.
We expect the locks are coarse-grained rather than fine-grained. These two styles of use suggest different requirements from a lock server.

Chubby is intended to provide only coarse-grained locking. Fortunately, it is straightforward for clients to implement their own fine-grained locks tailored to their application. 

#### System Structure

![System Structure](./resource/chubby-structure.png)

The replicas use a distributed consensus protocol to elect a master; the master must obtain votes from a majority of the replicas, plus promises that those replicas will not elect a different master for an interval of a few seconds known as the *master lease*.

The replicas maintain copies of a simple database, but only the master initiates reads and writes of this database.
All other replicas simply copy updates from the master, sent using the consensus protocol.

Clients find the master by sending master location requests to the replicas listed in the DNS.
Read requests are satisfied by the master alone; this is safe provided the master lease has not expired.
If a master fails, the other replicas run the election protocol when their master leases expire; a new master will typically be elected in a few seconds.

If a replica fails and does not recover for a few hours, a simple replacement system selects a fresh machine from a free pool and starts the lock server binary on it.
It then updates the DNS tables, replacing the IP address of the failed replica with that of the new one.

#### Files, directories, and handles

Chubby exports a file system interface similar to, but simpler than that of UNIX; it is a tree.

The name space contains only files and directories, collectively called *nodes*. Every such node has only one name within its cell.

Nodes may be either permanent or ephemeral.
Any node may be deleted explicitly, but ephemeral nodes are also deleted if no client has them open.

The per-node meta-data includes four monotonically increasing 64-bit numbers that allow clients to detect changes easily:

1. an instance number; greater than the instance number of any previous node with the same name
2. a content generation number (files only); this increases when the file’s contents are written
3. a lock generation number; this increases when the node’s lock transitions from *free* to *held*
4. an ACL generation number; this increases when the node’s ACL names are written


#### Locks and sequencers

Each Chubby file and directory can act as a reader-writer lock.
Like the mutexes known to most programmers, locks are advisory.
For why we reject *mandatory* locks:

- Chubby locks not only protect the file
- To make debugging and administrative operations easier
- Our developers perform error checking in the conventional way

In Chubby, acquiring a lock in either mode requires write permission so that an unprivileged reader cannot prevent a writer from making progress.

Locking is complex in distributed systems because communication is typically uncertain, and processes may fail independently.

It is costly to introduce sequence numbers into all the interactions in an existing complex system.
Instead, Chubby provides a means by which sequence numbers can be introduced into only those interactions that make use of locks.
At any time, a lock holder may request a *sequencer*, an opaque byte-string that describes the state of the lock immediately after acquisition.
The chubby client passes the sequencer to servers which requires the operations to be protected by the lock. The recipent of the sequencer is expected to test whether the sequencer is still valid and has the appropriate mode.

If sequencer is not available, then *lock-delay* is simple and easy to prove correctness.

#### Events

Chubby clients may subscribe to a range of events when they create a handle.

#### API

#### Caching

To reduce read traffic, Chubby clients cache file data and node meta-data (including file absence) in a consistent, write-through cache held in memory.
The cache is maintained by a lease mechanism described below, and kept consistent by invalidations sent by the master, which keeps a list of what each client may be caching.

When file data or meta-data is to be changed, the modification is blocked while the master sends invalidations for the data to every client that may have cached it; this mechanism sits on top of KeepAlive RPCs, discussed more fully in the next section.

#### Sessions and KeepAlives

A Chubby session is a relationship between a Chubby cell and a Chubby client; it exists for some interval of time, and is maintained by periodic handshakes called KeepAlives.

#### Failover



At any time, a lock holder may request a sequencer, an opaque byte-string that describes the state of the lock immediately after acquisition.
It contains the name of the lock, the mode in which it was acquired (exclusive or shared), and the lock generation number.
The client passes the sequencer to services if it expects the operation to be protected by the lock.

Although we find sequencers simple to use, important protocols evove slowly. Chubby therefore provides an imperfect but easier mechanism called lock-delay.

#### Events

Chubby clients may subscribe to a range of events when they create a handle.
Events are delivered after the corresponding action has taken place.

#### API

- `Open()`
	- how the handle will be used
	- events should be delivered
	- lock delay
	- whether a new file or directory should (or must) be created
- `Close()`
- `GetContentAndStat()`
- `SetContents()`
- `SetACL()`
- `Delete()`
- `Acquire()`, `TryAcquire()`, `Release()`
- `GetSequencer()`
- `SetSequencer()`
- `CheckSequencer()`

Clients can use this API to perform primary election as follows: All potential primaries open the lock file and attempt to acquire the lock.
One succeeds and becomes the primary, while the others act as replicas.
The primary writes its identity into the lock file with SetContents() so that it can be found by clients and replicas, which read the file with GetContentsAndStat(), perhaps in response to a file-modification event.

#### Caching

To reduce read traffic, Chubby clients cache file data and node meta-data (including file absence) in a consistent, write-through cache held in memory.
The cache is maintained by a lease mechanism.
The protocol ensures that clients see either a consistent view of Chubby state, or an error.

When file data or meta-data is to be changed, the modification is blocked while the master sends invalidations for the data to every client that may have cached it; this mechanism sits on top of KeepAlive RPCs.

Despite the overheads of providing strict consistency, we rejected weaker models because we felt that programmers would find them harder to use.

#### Sessions and KeepAlives

A Chubby session is a relationship between a Chubby cell and a Chubby client; it exists for some interval of time, and is maintained by periodic handshakes called KeepAlives.

Each session has an associated lease an interval of time extending into the future during which the master guarantees not to terminate the session unilaterally.

The master advances the lease timeout in three circumstances:

- on creation of the session
- when a master fail-over occurs
- it responds to a KeepAlive RPC from the client; default 12 seconds

The client maintains a local lease timeout that is a conservative approximation of the master’s lease timeout.

If a client’s local lease timeout expires, it becomes unsure whether the master has terminated its session.
The client empties and disables its cache, and we say that its session is in *jeopardy*.
The client waits a further interval called the grace period, 45s by default (election time).

#### Failover

![Figure 2]()

A newly elected master proceeds:

1. TODO

#### Database Implementation

The first version of Chubby used the replicated version of Berkeley DB.

And then Chubby implements a database for itself.

While Berkeley DB’s maintainers solved the problems we had, we felt that use of the replication code exposed us to more risk than we wished to take.
Chubby used few of the features of Berkeley DB, and so this rewrite allowed significant simplification of the system as a whole; for example, while we needed atomic operations, we did not need general transactions.

### Mechanisms for scaling

- We can create an arbitrary number of Chubby cells。
- The master may increase lease times from the default 12s up to around 60s when it is under heavy load。

#### Proxies

A proxy can reduce server load by handling both KeepAlive and read requests; it cannot reduce write traffic, which passes through the proxy’s cache.
But even with aggressive client caching, write traffic constitutes much less than one percent of Chubby’s normal workload

#### Partitioning

If enabled, a Chubby cell would be composed of N partitions, each of which has a set of replicas and a master. Every node D/C in directory D would be stored on the partition `P(D/C) = hash(D) mod N`.

### Use, surprises and design errors

TODO

简单总结：

1. Chubby是一个分布式锁服务，CA。
1. 设计之初就是给系统逐步演进使用，所以不是一致性库，也不是一致性服务；而选择了一致性服务中的特殊一种——锁服务。
2. 单纯的给粗粒度锁准备；一个锁至少会持有数个小时。在Chubby之上可以构建细粒度锁服务，而且看起来没有额外的语义/性能问题。
3. 客户端有保证了一致性的缓存。
3. 可以通过Proxy和Partition来提高性能，前者是因为读写比极高（所以KeepAlive是系统绝大多数RPC）后者则更像一个强C弱AP的系统。
4. 第一版的状态机直接使用了BDB，后来自己维护了一个有WAL和snapshot的DB，因为只需要原子操作而不需要事务。
