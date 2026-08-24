+++
title = 'Spinlock vs Mutex'
date = '2025-12-12T10:20:40+09:00'
draft = false
translationKey = 'spinlock-vs-mutex'
slug = 'spinlock-vs-mutex-en'
aliases = ['/posts/spinlock-vs-mutex-en/']
description = 'Why a userspace spinlock spinning on CAS and a kernel-backed mutex going through futex end up suited to different workloads, explained through cache coherence.'
tags = ['Concurrency', 'Linux', 'CPU', 'Lock-Free']
categories = ['Concurrency', 'Systems']

[cover]
image = 'images/spinlock-vs-mutex/image1.png'
hiddenInSingle = true
+++

Locks split into two broad families: spinlocks, which run entirely in userspace on raw CPU instructions, and mutexes, which lean on the kernel. Both exist to guarantee mutual exclusion, but they get there differently, and that difference determines which workload each one fits.

## Spinlock

A spinlock lives entirely in userspace. The mechanism is simple: loop forever until a CAS (Compare-And-Swap) succeeds.

```c
while (!CAS(lock, 0, 1))
```

Most CPUs ship a dedicated instruction for this. On x86 it's `LOCK CMPXCHG`. The catch is that a CAS only succeeds once the core holds exclusive write access to that cache line. Getting there pulls the cache coherence protocol into play, and passing ownership of a cache line back and forth triggers frequent invalidation, what people call cache line bouncing. That round trip usually costs somewhere between 4 and 80 nanoseconds.

A cache line is the smallest unit a CPU cache moves and keeps coherent. Most modern CPUs use 64-byte lines, so reading even a single 8-byte value pulls in the full 64 bytes around it. That design pays off for spatial locality, but it also means two unrelated variables that happen to share a cache line can trigger false sharing.

Cache coherence is the inter-core protocol that keeps every core's cache in sync on a multicore machine. The most common one is MESI, which assigns each cache line one of four states: Modified (this core holds the newest value; memory may still be stale), Exclusive (only this core holds it, unmodified), Shared (multiple cores hold it read-only), and Invalid (not in this core's cache). Writing to a line requires holding it in M or E; S only permits reads. Most modern CPUs implement this through invalidation: a core sends a Read For Ownership (RFO) request, the interconnect relays it, every other core holding that line flips it to Invalid, and only then does the requesting core get ownership (M or E). These messages travel through the L1/L2 cache controllers and the interconnect.

Because the whole thing runs in userspace, a spinlock never triggers a system call. The tradeoff shows up under contention: a thread stuck spinning burns CPU the entire time it waits.

## Mutex

A mutex spans both userspace and kernel space. Most modern kernels implement it as a two-tier structure. Under no contention, it takes the same fast path as a spinlock, grabbing the lock with a single CAS. When that fails, it falls to the slow path: a system call, `futex(FUTEX_WAIT)`, that parks the thread.

Under no contention, a mutex is just as fast as a spinlock, one CAS and done. Under contention, it avoids wasting CPU by context-switching the thread out, but that protection isn't free. The system call alone costs around 500ns, and the context switch itself runs 3 to 5μs.

## Spinlock vs Mutex

Put side by side: a spinlock is cheap, 40 to 80ns, but since it spins in a loop, CPU load climbs directly with contention. That makes it a good fit for workloads where the critical section is extremely short, like a bare memory access. A mutex costs more, 3 to 5μs, but its fast path keeps it competitive with a spinlock when there's no contention. Under contention, it yields the CPU to another thread instead of burning cycles, which makes it the better choice for workloads that hold the lock longer, like disk or network access.

That split shows up in practice. Redis, which is dominated by memory access, leans on spinlocks internally for things like its job queue. PostgreSQL, which spends much of its time in I/O-bound sections, relies mainly on mutexes.

![](/images/spinlock-vs-mutex/image1.png)

## Lock-Free Data Structures

A typical thread-safe data structure grabs a spinlock or mutex internally before touching its data. Some structures skip locks entirely and get thread safety from atomic operations alone, usually the same CAS-plus-loop shape as a spinlock. These are called lock-free data structures, and the classic example is the Treiber stack.

```java
// example code - from Wikipedia
import java.util.concurrent.atomic.*;

import net.jcip.annotations.*;

/**
 * ConcurrentStack
 *
 * Nonblocking stack using Treiber's algorithm
 *
 * @author Brian Goetz and Tim Peierls
 */
@ThreadSafe
public class ConcurrentStack <E> {
    AtomicReference<Node<E>> top = new AtomicReference<Node<E>>();

    public void push(E item) {
        Node<E> newHead = new Node<E>(item);
        Node<E> oldHead;

        do {
            oldHead = top.get();
            newHead.next = oldHead;
        } while (!top.compareAndSet(oldHead, newHead));
    }

    public E pop() {
        Node<E> oldHead;
        Node<E> newHead;

        do {
            oldHead = top.get();
            if (oldHead == null)
                return null;
            newHead = oldHead.next;
        } while (!top.compareAndSet(oldHead, newHead));

        return oldHead.item;
    }

    private static class Node <E> {
        public final E item;
        public Node<E> next;

        public Node(E item) {
            this.item = item;
        }
    }
}
```

Both `push` and `pop` follow the same shape: read the current `top`, link the new node to it, then swap `top` in with a CAS. If another thread changes `top` in between, the CAS fails and the loop tries again from scratch. That's how the stack stays consistent without ever taking a lock.

## References
- [Spinlocks vs. Mutexes: When to Spin and When to Sleep](https://howtech.substack.com/p/spinlocks-vs-mutexes-when-to-spin)
- [Treiber stack](https://en.wikipedia.org/wiki/Treiber_stack)
