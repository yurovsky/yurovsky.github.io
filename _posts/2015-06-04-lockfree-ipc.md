---
layout: post
title: Simple lockfree IPC using shared memory and C11
---

I have an interesting Linux application that requires a process with one or
more real-time threads to communicate with other processes on the system. The
use case is single producer with single consumer per IPC channel.

There are several ways of implementing IPC in Linux and I chose the simple
POSIX shared memory approach:
1. One process (the producer) creates a POSIX shared memory object and sizes it
according to the IPC API contract.
2. Another process (the consumer) opens this shared memory object.

Both processes use `mmap()` to map a window into this shared memory and can now
communicate:

![shared memory](/assets/lfs_proc.png)

We then need a data structure such as a stack or queue that both processes can
use to move data around in this window and this in turn cannot be dynamically
allocated (instead is mapped into the window) and we cannot pass around
pointers since each process sees the same window in its own address space. In
addition, the data structure must support concurrent access without expensive
operations such as locking.

The approach I took involves placing the data structure itself (or the
"management" portion) into the start of the shared window, followed by a buffer
that the data structure manages.  This works just like an implementation of a
stack or list that has all of its nodes allocated up front except here we also
place everything in a set location in memory (or rather cast a particular
memory location).

I chose a simple stack data structure and based my design on the [C11 Lock free Stack](http://nullprogram.com/blog/2014/09/02/) written by Chris Wellons.  This
does use the C11 `stdatomic.h` primitives so GCC 4.9 or newer is required to
compile and GCC should be told to be in C11 mode (for example with `-std=c11`).
GCC switched to C11 by default in version 5.0 so that is now the default if you
do not specify `-std=`.

This stack maintains two pointers: a `head` and `free` which point into the
pool of stack nodes.  Data is added by pushing onto the head (the new node
comes from the free stack) and is removed by popping off the head (and thereby
returning the node to the free stack). There are, however, no real pointers
since we are not directly working with physical memory.  In place of pointers I
am using indexes or offsets which in turn resolve to a memory location when
needed (that is, we manually implement base plus offset addressing).  The
memory window is represented as follows:

![memory window](/assets/lfs.png)

There is a one to one mapping of stack nodes to buffer pool locations (offsets)
and, by knowing the starting address of the data structure, we can always
calculate a pointer to some offset in the buffer pool.
