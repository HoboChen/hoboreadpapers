# Fault-Tolerant

## [The Design of a Practical System for Fault-Tolerant Virtual Machines]

SIGOPS 2010.

A commercial enterprise-grade system for providing fault-tolerant virtual machines is implemented, which based on the approach of replicating the execution of a primary virtual machine via a backup virtual machine on another server.

It is implemented in VMware vSphere 4.0, which runs on commodity servers.

- reduces performance of real applications by less than 20%
- needs as low as 20mbps to keep primary/backup in lockstep

### Introduction

One way of replicating the state on the backup server is to ship changes to all state of parimary, including CPU, memory, and I/O devices, to the backup nearly continuously.

Another way is referred to as the state machine approach; a VM can be considered a well-defined state machine whose operations are the operations of the machine being virtualized.

Since the hypervisor has full control over the execution of a VM, including delivery of all inputs, the hypervisor is able to capture all the necessary information about non-deterministic operations on the primary and then replay on the backup.

At this time, the production version supports only uni-processor VM.

Only fail-stop failures are dealed.

### Basic FT Design

TODO Figure 1

All input that the primary receives is sent to the backup VM via a network connection known as the *logging channel*.
For server workloads, the dominant input traffic is network and disk.
Additional information is transmitted as necessary to ensure that the backup VM always executes identically to the primary VM.

#### Deterministic Replay Implementation

TODO

简单总结：

- 主备，自动切换；vSphere 4.0商用
- 将VM看作确定性自动机；Hypervisor拿到所有非确定性操作的信息
- VM最多只能有一个vCPU