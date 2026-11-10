# PingHaz:  Signal-Based vs. Asymmetric-Fence Memory Reclamation
### Team: Akanksha Akkihal (aakkihal), Lilly Yang (qingqiny)

## Summary

We are going to implement the Publish-on-Ping (POP)[1] technique inside Meta's Folly library as an alternative to its current memory reclamation, which is based on asymmetric fences. We will compare the performance of textbook hazard pointers, Folly's hazard pointers, our POP implementation, and epoch-based reclamation on lock-free data structures such as lock-free lists and Folly's `ConcurrentHashMap`, under both read-heavy and update-heavy workloads.

## Background

Lock-free data structures let many threads share data without locks, but in languages without garbage collection they need a safe way to free memory. A removed node can't be freed right away, because another thread might still be reading it.

**Hazard pointers (HP)** solve this by having readers announce each node before using it, so writers only free nodes no one has announced. This is safe even if a thread gets stuck, but readers pay for a memory fence on every node they visit. **Epoch-based reclamation (EBR)** is faster for readers, but one stuck thread can stop all memory from being freed.

Two newer designs keep HP's safety without the per-read fence:

- **Folly's hazard pointers** (Meta) move the fence to the reclaimer, which calls Linux `membarrier()` before scanning.
- **Publish-on-Ping (POP)** [1] keeps readers' reservations private, and the reclaimer uses signals to ask threads to publish them.

We will implement POP in Folly and compare the two approaches.

## The Challenge

Reclamation cost depends on memory ordering, cache coherence, and OS scheduling all at once, so we can't predict which scheme wins without building and measuring them.

- **Workload:** threads run random lookups, inserts, and deletes on a shared lock-free list or hash table. We vary the read/write mix, key range, and thread count.
- **Correctness:** with no locks, a reader's announcement and a reclaimer's scan must happen in the right order, which is easy to get wrong.
- **Memory and communication:** pointer chasing has poor locality, and shared hazard slots and retired lists cause cache coherence traffic.
- **System constraints:** x86 can reorder a store with a later load, which every scheme must handle. POP's signals add their own challenges: handlers must be async-signal-safe, threads that aren't running delay reclamation, and Folly is a large codebase to work in.

## Resources

## Resources

- [Folly](https://github.com/facebook/folly)
- [Publish on Ping (Singh and Brown, PPoPP 2025)](https://arxiv.org/abs/2501.04250)


## Goals and Deliverables

We plan to reimplement POP, integrate it into Folly, and compare it against production and textbook baselines.

### Plan to achieve

- **Reimplement POP:** build a test harness with a lock-free list and hash table, and implement POP alongside EBR, textbook hazard pointers, and a never-free baseline.
- **Integrate into Folly:** first swap `membarrier` for signals to measure their cost, then implement full POP, keeping Folly's existing API.
- **Evaluation:** measure throughput as threads increase, memory use when a thread stalls, tail latency, and oversubscription, using perf counters to explain the results.

### Hope to achieve

- EpochPOP and Hazard Eras
- Folly's `ConcurrentHashMap` as a real-world workload
- A pull request to Folly with our changes and benchmarks

## Questions We Will Answer

- Are signals or `membarrier` cheaper, and does that change with more threads or more frequent reclamation?
- How do these schemes affect tail latency, not just throughput?
- How do POP and Folly behave when threads aren't always running?
- How much of POP's benefit comes from removing the fence vs. keeping reservations private?
- Does shifting cost from readers to reclaimers make the system faster overall, or just move the slowdown elsewhere?

## Platform Choice

We'll develop on our local Linux machines and benchmark on PSC, whose many cores let us see how costs change as we add threads. We'll use [Google Benchmark](https://github.com/google/benchmark) to measure runtime and memory usage.

## Schedule

**Week 1 (11/9 – 11/15)**
- Build a lock-free list and our reclamation implementations.
- Set up the Folly build and benchmarks, and trace the `hazptr_holder` and `membarrier()` code.

**Week 2 (11/16 – 11/22)**
- Integrate POP into Folly's hazard pointers.
- Stress test on Folly's `ConcurrentHashMap` to check for use-after-free and ABA bugs.

**Week 3 (11/23 – 11/29)**
- Profile with perf: oversubscription tests, context switches, signal latency, and CPU stalls.
- Make throughput and tail-latency graphs, draft the final report, and prepare the upstream PR.

**Week 4 (11/30 - 12/6)**
- Finalize the final report, the website, poster, and other visualization demo for the presentation.

## References

[1] Ajay Singh and Trevor Brown. 2025. Publish on Ping: A Better Way to Publish Reservations in Memory Reclamation for Concurrent Data Structures. In *PPoPP '25*, Las Vegas, NV, USA. ACM. https://doi.org/10.1145/3710848.3710890

