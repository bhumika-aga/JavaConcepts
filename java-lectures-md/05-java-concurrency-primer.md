# Java Concurrency Primer

_java concurrency · the twenty-minute version_

> The short form: one fact that causes all the trouble, three distinct problems it creates, and the five tools that solve them. Everything here is expanded at length in _Inside Java Threads_.

---

## Contents

1. [The one fact](#the-one-fact)
2. [Three problems, not one](#three-problems-not-one)
3. [Five tools](#five-tools)
4. [What to reach for](#what-to-reach-for)
5. [Seven rules](#seven-rules)
6. [Where to go next](#where-to-go-next)

---

## The one fact

Every thread gets its own stack. All threads share one heap. That single asymmetry is the source of every concurrency bug you will ever write.

![Three threads with private stacks all reading and writing one shared heap](diagrams/java-concurrency-primer-01.svg)

**The corollary is the best design advice in the subject:** if nothing is shared, nothing needs synchronising. Immutability and confinement remove problems rather than managing them.

## Three problems, not one

These are independent. Fixing atomicity does nothing for visibility, and neither addresses ordering. Most confusion comes from treating them as one thing.

- **Atomicity** — An operation you think is one step is several. `count++` is read, add, write — another thread slips between them.

A write by one thread may never reach another. Not late — potentially never, because the JIT can hoist the read out of the loop.

The compiler and CPU reorder independent instructions. Another thread can observe your steps in a different sequence.

```java
            // Atomicity — two threads, both read 4, both write 5. One increment lost.
count++;

// Visibility — this can spin forever; nothing forces a re-read.
private boolean stopped = false;
while (!stopped) { }

// Ordering — the reader may see ready==true before config is built.
config = new Config();
ready  = true;

```

And three ways to be stuck rather than wrong: **deadlock** (two threads each hold what the other needs), **livelock** (both keep yielding, neither progresses), and **starvation** (one thread never gets its turn).

## Five tools

Almost all correct concurrent Java is built from these. Reach for them in roughly this order.

### 1 · Immutability — the one that removes the problem

```java
            record Point(int x, int y) { }        // cannot be raced, ever

// "Changing" it means swapping the reference atomically
AtomicReference<Config> cfg = new AtomicReference<>(initial);
cfg.updateAndGet(c -> c.withTimeout(30));

```

### 2 · `synchronized` — mutual exclusion

```java
            private final Object lock = new Object();   // private — nobody outside can grab it

void transfer() {
    synchronized (lock) { /* keep this short; never do I/O here */ }
}

```

### 3 · `volatile` — visibility for a single field

```java
            private volatile boolean stopped = false;   // the loop now terminates

// It publishes everything written BEFORE it, too — which is why
// the flag pattern works. But it is NOT a lock: volatile count++
// is still broken.

```

### 4 · Atomics — lock-free counters

```java
            AtomicInteger hits = new AtomicInteger();
hits.incrementAndGet();
hits.updateAndGet(v -> v * 2);

LongAdder busy = new LongAdder();   // better under heavy write contention

```

### 5 · `java.util.concurrent` — don't build your own

```java
            Map<String, Integer> shared = new ConcurrentHashMap<>();
shared.merge(key, 1, Integer::sum);          // atomic counter

BlockingQueue<Job> work = new ArrayBlockingQueue<>(100);   // bounded!

try (ExecutorService pool = Executors.newVirtualThreadPerTaskExecutor()) {
    pool.submit(() -> handle(request));      // Java 21+
}

```

## What to reach for

| I want to…                               | Use                  |
| ---------------------------------------- | -------------------- |
| Shared state that never changes          | `record / final`     |
| A counter many threads bump              | `LongAdder`          |
| A flag one thread sets, another reads    | `volatile`           |
| Several fields that must change together | `synchronized`       |
| A map many threads share                 | `ConcurrentHashMap`  |
| Hand work between threads                | `ArrayBlockingQueue` |
| Run many independent tasks               | `ExecutorService`    |
| Thousands of blocking I/O calls          | `virtual threads`    |
| Cap concurrent access to a resource      | `Semaphore`          |
| Wait for N tasks to finish               | `CountDownLatch`     |

| Guarantee                | `volatile` | `synchronized` | `Atomic*` |
| ------------------------ | ---------- | -------------- | --------- |
| Visibility of writes     | yes        | yes            | yes       |
| Prevents reordering      | yes        | yes            | yes       |
| Atomic read-modify-write | no         | yes            | yes       |
| Mutual exclusion         | no         | yes            | no        |
| Can block a thread       | no         | yes            | no        |

## Seven rules

1. **Don't share mutable state** — Immutability and thread confinement eliminate bugs instead of managing them. Always try this first.
2. **Never share a plain HashMap or ArrayList** — Concurrent writes corrupt internal structure, not just data. Use the concurrent collections.
3. **Two atomic calls are not one atomic operation** — `containsKey` then `put` has a gap. Use `merge`, `compute`, or `putIfAbsent`.
4. **Keep critical sections tiny** — Never perform I/O, call unknown code, or block while holding a lock. Everyone queues behind you.
5. **Always acquire locks in the same order** — The single habit that prevents deadlock. Enforce it once and a cycle becomes impossible.
6. **Every blocking call gets a timeout** — `future.get()` with no deadline lets one stuck task hang the caller forever.
7. **Never swallow InterruptedException** — Rethrow it, or restore the flag with `Thread.currentThread().interrupt()`. An empty catch here breaks cancellation everywhere.

> ⚠️ **The most common single mistake**
>
> Treating `volatile` as a lightweight lock. It guarantees you read the _latest_ value; it does nothing to stop another thread reading that same latest value and racing you to write. For anything read-modify-write, you need an atomic class or a lock.

## Where to go next

This page is the on-ramp. The full treatment — the memory model and happens-before, lock internals, the whole `java.util.concurrent` toolkit, executor tuning, `CompletableFuture`, and virtual threads — is in the companion lecture.

**Inside Java Threads →** 12 parts, timing diagrams, and a gotchas table.

> **If you remember one thing**
>
> Concurrent code runs in _an_ order, and only the orderings you explicitly constrain are guaranteed. Every tool on this page is a way of constraining one.
