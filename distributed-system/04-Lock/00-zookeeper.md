# ZooKeeper: Wait-free coordination for Internet-scale systems

## Abstract and Introduction

In this paper, we describe ZooKeeper, a service for coordinating processes of distributed applications.
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

The data model of ZooKeeper is essentially a file system with a simplified API and only full data reads and writes.