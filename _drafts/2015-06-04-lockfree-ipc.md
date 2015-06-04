---
layout: post
title: Simple lockfree IPC using shared memory and C11
---

I have an interesting Linux application that requires a process with one or
more real-time threads to communicate with other processes on the system. The
use case is single producer with single consumer per IPC channel.

The real-time requirements of the application (this runs on a kernel with the
PREEMPT RT patch set applied) requires that the IPC channel is lock-free and
presents minimal overhead. That is, a real-time producer thread or process must
be able to quickly write something and keep executing without locking or
context switches.

## Shared Memory

There are several ways of implementing IPC in Linux and I chose the simple
POSIX shared memory approach to represent IPC "channels":
1. One process (the producer) creates a POSIX shared memory object and sizes it
according to the IPC API contract.
2. Another process (the consumer) opens this shared memory object.

Both processes use `mmap()` to map a window into this shared memory and can now
communicate:

![shared memory](/assets/lfs_proc.png)

## Stack Structure

We then need a data structure such as a stack or list that both processes can
use to move data around in this window and this in turn cannot be dynamically
allocated (instead is mapped into the window) and we cannot pass around
pointers since each process sees the same window in its own address space. In
addition, the data structure must support concurrent access.

The approach I took involves placing the data structure itself (or the
"management" portion) into the start of the shared window, followed by a buffer
that the data structure manages.  This works just like an implementation of a
stack or list that has all of its nodes allocated up front except here we also
place everything in a set location in memory (or rather cast a particular
memory location).

I chose a simple stack data structure and based my design on the [C11 Lock free Stack](http://nullprogram.com/blog/2014/09/02/) written by Chris Wellons.  This
does use the [C11 `stdatomic.h` primitives](http://en.cppreference.com/w/c/atomic) so GCC 4.9 or newer is required to
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

## Machine capabilities and limitations

This approach assumes that the machine in question is capable of compare and
swap (CAS) and most of them (modern ARM, PowerPC, x86, etc) are.  The C11
generic function [`atomic_compare_exchange_weak`](http://en.cppreference.com/w/c/atomic/atomic_compare_exchange) will generate suitable CAS instructions.

Machines do vary in the CAS size they support (among other things), that is how
many bytes can be atomically compared and swapped.  There is a set of
[Atomic lock-free constants](http://en.cppreference.com/w/c/atomic/ATOMIC_LOCK_FREE_consts) that should be checked to determine what the machine is capable of.

The CPU I am targeting for example does not support "double CAS" and therefore
cannot compare and swap a pointer and ABA counter in one shot.  On the other
hand, I am using base plus offset
addressing rather than pointers and I have a finite number of "slots" that I
need to address so the size of the "pointer" (offset) can be adjusted such that
it, plus an ABA counter, can "fit" into the CAS capability of the machine.

## Data Structures

The machine I am targeting can CAS four bytes so this can represent a stack
"head" pointer:

        struct lfs_head {
            uint16_t inode; /* offset into the node pool */
            uint16_t count; /* ABA counter */
        };

The "nodes" themselves are nothing more than a next "pointer" which in turn is
just a 16-bit value representing another offset into the node pool,

        struct lfs_node {
            uint16_t inext; /* offset into the node pool */
        };

The stack "descriptor" itself contains a number of elements including the stack
head and free "pointers". It is set up at initialization and then modified as
needed by `push` and `pop` operations. We would normally have pointers here to
the stack node pool and buffer pool, however with the shared memory window
design this is not possible and, given the fixed layout in memory, we can easily
calculate those locations as offsets from the location of the descriptor (at
the start of the buffer).

        typedef struct {
            size_t depth; /* Number of nodes (and corresponding data buffers) */
            size_t data_nb; /* Size of each data buffer, in bytes */
            _Atomic struct lfs_head shead, sfree; /* stack and free stack */
            _Atomic size_t size; /* Number of nodes on the stack */
        } __attribute__((packed)) lfs_t;

This descriptor implies that
* The node pool is located at the address of this descriptor plus the size of the descriptor.
* The Nth node is located at the address of the node pool plus the size of a node multiplied by `N`
* The buffer pool is located at the address of the node pool plus the size of the node pool (which in turn is the size of a node multiplied by the `depth`).
* The Nth buffer is located at the address of the buffer pool plus the size of one buffer, `data_nb`, multiplied by `N`.

## API

The lockfree stack (LFS) itself must be initialized and this in turn needs to
know where the descriptor is, the desired stack depth, and how big the buffers
in the buffer pool are (the latter is used to calculate data pointers from the
internally managed offsets).

        void lfs_init(lfs_t *lfs, size_t depth, size_t data_nb);

From there, arbitrary data can be pushed to the stack as long as there are one
or more free nodes.  The data is copied into the corresponding buffer and the
size is known from initialization time.

        bool lfs_push(lfs_t *lfs, void *value);

Data can be popped off the stack as well:

        void *lfs_pop(lfs_t *lfs);

And the number of nodes on the stack is tracked and available:

        size_t lfs_size(lfs_t *lfs);
