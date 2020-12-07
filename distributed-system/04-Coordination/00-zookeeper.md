# [ZooKeeper: Wait-free coordination for Internet-scale systems](https://static.usenix.org/event/atc10/tech/full_papers/Hunt.pdf)

## Abstract and Introduction

ZooKeeper, a service for coordinating processes of distributed applications.
It incorporates elements from group messaging, shared registers, and distributed lock services in a replicated, centralized service.
The interface exposed by ZooKeeper has the wait-free aspects of shared registers with an event-driven mechanism similar to cache invalidations of distributed file systems to provide a simple, yet powerful coordination service.

2:1 to 100:1 W/R ratio, 10-100K TPS.

When designing our coordination service, we moved away from implementing specific primitives on the server side, and instead we opted for exposing an API that enables application developers to implement their own primitives.
Such a choice led to the implementation of a *coordination kernel* that enables new primitives without requiring changes to the service core.

Our system, Zookeeper, hence implements an API that manipulates simple waitfree data objects organized hierarchically as in file systems. Implementing wait-free data objects, however, differentiates ZooKeeper significantly from systems based on blocking primitives such as locks.

Guarantee:

- wait-free
- FIFO client ordering
- linearizable; see [linearizability and serializability](https://stackoverflow.com/questions/4179587/what-is-the-difference-between-linearizability-and-serializability)

## Zookeeper Service

### Overview

Clients submit requests to ZooKeeper through a client API using a ZooKeeper client library.

ZooKeeper provides to its clients the abstraction of a set of data nodes (`znode`s), organized according to a hierarchical name space.

All `znode`s store data, and all znodes, except for ephemeral znodes, can have children.

#### Data Model

The data model of ZooKeeper is essentially a file system with a simplified API and only full data reads and writes or a k/v table with hierarchical keys.

#### Sessions

A client connects to ZooKeeper and initiates a session. There is associated timeout.

### Client API

```cpp
create(path, data, flags)

delete(path, version)

exists(path, watch)

getData(path, watch)

setData(path, data, version)

getChildren(path, watch)

sync(path) // the path is currently ignored
```

All methods have both sync and async API. For async APIs, it guarantess that corresponding callbacks for each API call are invoked in order.

Because only update requests are A-linearizable, ZooKeeper processes read requests locally at each replica.

Q: "This problem is solved by the ordering guarantee for the notifications: if a client is watching for a change, the client will see the notification event before it sees the new state of the system after the change is made."

Another problem can arise when clients have their own communication channels in addition to ZooKeeper.
For example, consider two clients A and B that have a shared configuration in ZooKeeper and communicate through a shared communication channel.

Using the above guarantees B can make sure that it sees the most up-to-date information by issuing a write before re-reading the configuration.

To handle this scenario more efficiently ZooKeeper provides the `sync` request: when followed by a read, constitutes a *slow read*.

### Examples of primitives

The ZooKeeper service knows nothing about these more powerful primitives since they are entirely implemented at the client using the ZooKeeper client API.

#### Simple Lock

Q: What is herd effect?

## ZooKeeper Implementation

### Request Processor

When the leader receives a write request, it calculates what the state of the system will be when the write is applied and transforms it into a transaction that captures this new state.

The future state must be calculated because there may be outstanding transactions that have not yet been applied to the database.

### Atomic Broadcast

But since Zab does not persistently record the id of every message delivered, Zab may redeliver a message during recovery; but it is harmless as all the operations are idempotent.

### Replicated Database

They do not lock the ZooKeeper when snapshotting, so the snapshot is called *fuzzy* snapshot.
As all the operations are idempotent, so not locking is good.

### Client

Only the server that a client is connected to tracks and triggers notification for that client.

This primitive executes asynchronously and is ordered by the leader after all pending writes to its local replica.

If the client has a more recent view than the server, the server does not reestablish the session with the client until the server has caught up.

## 简单总结：

1. Wait-free coordination, watch, idempotent.
2. Zab协议，。
3. 只提供wait-free语义，实用一些的语义（比如锁）由client实现，server并不知道。
4. 本地读，如果需要强一致语义，有一个专门的`sync` API。