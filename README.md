# PingHaz:  Signal-Based vs. Asymmetric-Fence Memory Reclamation
### Team: Akanksha Akkihal (aakkihal), Lilly Yang (qingqiny)

## Summary

We are going to implement the Publish-on-Ping (POP)[1] technique inside Meta's Folly library as an alternative to its current memory reclamation, which is based on asymmetric fences. We will compare the performance of textbook hazard pointers, Folly's hazard pointers, our POP implementation, and epoch-based reclamation on lock-free data structures such as lock-free lists and Folly's `ConcurrentHashMap`, under both read-heavy and update-heavy workloads.

## Background

Lock-free data structures let many threads read and update shared data without locks, which is why they scale on multi-core machines. In languages without garbage collection (C, C++, Rust), they need a safe memory reclamation (SMR) scheme. When a writer removes a node, it cannot free it immediately, because another thread may still be reading it.

**Hazard pointers (HP)** fix this by having readers announce each node before using it:
```
protect(src):
    do:
        p = src.load()
        my_hazard_slot = p          // announce
        MFENCE                      // make it visible before re-checking
    while p != src.load()           // still reachable?
    return p
```

When a writer removes a node, it doesn't free it right away. It puts the node on a retired list and later frees only the nodes that no thread has in its hazard slot. This makes HP robust, since a stuck thread can only hold onto a few nodes. The downside is that readers need a fence for every node they visit, which gets expensive when most operations are reads. **Epoch-based reclamation (EBR)** is much cheaper for readers, but if even one thread gets stuck, no memory can be freed, so memory use can keep growing.

Two newer designs keep hazard pointers' safety but remove the fence from the read path:

- **Folly's hazard pointers** (from Meta, used in production code like `ConcurrentHashMap`) use asymmetric fences. Readers only use a cheap compiler barrier. Before scanning hazard slots, the reclaimer calls Linux `membarrier()`, which forces every thread in the process to execute a memory barrier.
- **Publish-on-Ping (POP)** [1] has readers keep their reservations private, with no fences at all. When a reclaimer wants to scan, it sends a POSIX signal to every thread, and each thread's signal handler publishes its reservations.

Both designs make reads cheap and push the cost onto reclamation, which happens rarely. They just use different OS mechanisms to do it, and comparing them is the focus of our project. We will implement POP in Meta's open-source Folly library, and then compare it against Folly's existing `membarrier` approach and the classic schemes.


## The Challenge

The cost of memory reclamation comes from several things at once: memory ordering, cache coherence, and how the OS schedules threads. Because these interact, we can't predict which scheme will win without building and measuring them. We want to learn how these costs trade off as the number of threads and the workload change.

### Workload Characteristics

- **Workload:** Each thread runs a loop of random lookups, inserts, and deletes on a shared lock-free list or hash table. We vary the read/write mix, the key range, and the number of threads. A smaller key range means more threads hit the same nodes, so there is more contention.
- **Dependencies:** Readers and writers run at the same time without locks. The code is only correct if a reader's announcement and a reclaimer's scan happen in the right order, which is easy to get wrong.
- **Memory access and communication:** Traversing a list means following pointers to scattered memory, so there is little locality. Hazard slots, epoch counters, and retired lists are shared between threads, so every publish and scan causes cache coherence traffic. Each thread's slots also need padding to avoid false sharing.

### System Constraints

- On x86, a store followed by a load can be reordered by the CPU. This is exactly what hazard pointers need to prevent, so every scheme must handle it correctly.
- With POP, a reclaimer has to wait for every thread to run its signal handler, so a thread that isn't currently running can delay reclamation.
- Signal handlers can only use a limited set of safe operations, and a signal can arrive in the middle of a traversal. We also need to keep track of which threads are alive, even if a thread exits while a reclaimer is signaling it.
- Folly is a large production codebase that is hard to build, and our POP code has to work within it.


