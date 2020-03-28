## [Frangipani: A Scalable Distributed File System](https://zoo.cs.yale.edu/classes/cs422/2014/bib/thekkath97frangipani.pdf)

### Abstract & Introduction

Frangipani is a new file system. The lower layer is Petal, a distributed storage service that provides incrementally scalable, highly available, automatically managed virtual disks.
In the upper layer, multiple machines run the same Frangipani file system code on top of a shared Petal virtual disk, using a distributed lock service to ensure coherence.

Frangipani has a very simple internal structure, and features:

1. Strong consistency.
2. Scale out.
3. Decouple user with concrete store.
4. Online backup.
5. Partial failure tolerant.

Petal provides virtual disks.

![Figure 1](../resource/frangipani-0.pdf)

Multiple interchangeable Frangipani servers provide access to the same files by running on top of a shared Petal virtual disk, coordinat ing their actions with locks to ensure coherence. 

### System Structure

User programs access Frangipani through the standard operating system call interface.

Programs running on different machines all see the same files, and their views are coherent.

Programs get essentially the same semantic guarantees as on a local Unix file system: changes to file contents are staged through the local kernel buffer pool and are not guaranteed to reach nonvolatile storage until the next applicable fsync or sync system call, but metadata1 changes are logged and can optionally be guaranteed non-volatile by the time the system call returns. In a small departure from local file system semantics, Frangipani maintains a file’s last-accessed time only approximately, to avoid doing a metadata write for every data
read.