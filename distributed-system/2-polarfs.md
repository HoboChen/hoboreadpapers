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