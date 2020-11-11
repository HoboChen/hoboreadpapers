# ZooKeeper: Wait-free coordination for Internet-scale systems

## Abstract and Introduction

In this paper, we describe ZooKeeper, a service for coordinating processes of distributed applications.
It incorporates elements from group messaging, shared registers, and distributed lock services in a replicated, centralized service.

When designing our coordination service, we moved away from implementing specific primitives on the server side, and instead we opted for exposing an API that enables application developers to implement their own primitives.

Our system, Zookeeper, hence implements an API that manipulates simple waitfree data objects organized hierarchically as in file systems. Implementing wait-free data objects, however, differentiates ZooKeeper significantly from systems based on blocking primitives such as locks.

Guarantee:

- wait-free
- FIFO client ordering
- linearizable