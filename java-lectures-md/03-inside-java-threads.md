# Inside Java Threads

_java concurrency · from first principles to Loom_

> Single-threaded code runs in the order you wrote it. Concurrent code runs in _an_ order — and only the orderings you explicitly constrain are guaranteed. Everything below follows from that one sentence.

---

## Contents

1. [What a thread actually is](#what-a-thread-actually-is)
2. [Making threads run](#making-threads-run)
3. [Why concurrency is hard](#why-concurrency-is-hard)
4. [The Java Memory Model](#the-java-memory-model)
5. [Locks and coordination](#locks-and-coordination)
6. [The java.util.concurrent toolkit](#the-javautilconcurrent-toolkit)
7. [Executors — stop managing threads yourself](#executors--stop-managing-threads-yourself)
8. [CompletableFuture](#completablefuture)
9. [Virtual threads](#virtual-threads)
10. [Patterns that work](#patterns-that-work)
11. [Ways to get hurt](#ways-to-get-hurt)
12. [A learning path](#a-learning-path)

---

## What a thread actually is

A thread is an independent path of execution through your program. Your process can have many, and they all share one thing that makes everything else complicated.

When the JVM starts, it creates one thread to run `main`. Every thread you create after that gets **its own call stack** — its own local variables, its own method-call chain, its own program counter. But every thread shares **one heap**. Every object you ever allocate lives there, visible to all of them.

![Three threads each with a private stack, all reading and writing one shared heap](diagrams/inside-java-threads-01.svg)

**Private stacks, one shared heap.** Local variables are automatically thread-safe — nobody else can see them. Everything reachable from the heap is fair game for every thread simultaneously, and that is the entire source of difficulty.

So the first rule of thread safety falls out for free: **a local variable of a primitive type can never be involved in a race.** Problems begin the moment two threads can reach the same object.

### Why bother with threads at all

- **Use more than one core.** A single-threaded program on a 16-core machine uses one core. Genuinely CPU-bound work parallelises.
- **Stop wasting time on waiting.** Far more common. A thread blocked on a database call or an HTTP response is doing nothing; another thread can work meanwhile. Most server workloads are this, not the first case.
- **Keep something responsive.** UI toolkits and servers need a thread that never blocks so the application keeps answering.

## Making threads run

Three ways to start work, and a lifecycle you need to recognise in a stack dump.

```java
            // 1. A raw Thread with a lambda (Runnable is a functional interface).
Thread t = new Thread(() -> System.out.println("running"));
t.start();       // starts a NEW thread
t.run();         // ✗ just a normal method call on the CURRENT thread
t.join();        // block until t finishes

// 2. Subclassing Thread — legal, rarely right. You're inheriting
//    when you only wanted to pass behaviour.
class Worker extends Thread {
    @Override public void run() { /* … */ }
}

// 3. What you'll actually use in real code — hand work to a pool.
try (ExecutorService pool = Executors.newFixedThreadPool(4)) {
    pool.submit(() -> doWork());
}   // close() shuts down and waits — Java 19+

```

> ⚠️ **The classic beginner trap**
>
> `t.run()` compiles, runs your code, and creates no thread whatsoever. It executes synchronously on the caller. Only `t.start()` asks the JVM for a new thread — and calling `start()` twice on the same object throws `IllegalThreadStateException`. A `Thread` is single-use.

### The lifecycle

Every thread is in exactly one of six states, and `Thread.getState()` will tell you which. Reading these correctly is most of what diagnosing a hung application involves.

![Thread state machine showing transitions between new, runnable, blocked, waiting, timed waiting and terminated](diagrams/inside-java-threads-02.svg)

**Six states, and only one of them is running.** `RUNNABLE` covers both "executing right now" and "ready, waiting for a core" — the JVM can't distinguish them, so a thread dump can't either.

### Interruption: the cooperative stop signal

There is no safe way to kill a thread. `Thread.stop()` exists, is deprecated for removal, and corrupts state by releasing locks mid-update. The supported mechanism is a _request_ the target must honour.

```java
            Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            doChunkOfWork();
            Thread.sleep(100);
        } catch (InterruptedException e) {
            // sleep/wait/join CLEAR the flag when they throw,
            // so restore it and leave.
            Thread.currentThread().interrupt();
            break;
        }
    }
});
worker.start();
worker.interrupt();   // polite request, not a kill

```

> ⚠️ **Never do this**
>
> `catch (InterruptedException e) { }` — swallowing it silently discards the only cancellation signal your thread will get, and clears the flag so nothing downstream can detect it either. Either rethrow it, or re-set the flag with `Thread.currentThread().interrupt()`. An empty catch block here is a bug every single time.

Two other properties worth knowing early: a **daemon** thread (`t.setDaemon(true)`) does not keep the JVM alive — when only daemons remain, the JVM exits and kills them abruptly. And an uncaught exception in a thread kills only that thread, silently, unless you install an `UncaughtExceptionHandler`.

## Why concurrency is hard

Not one problem — three independent ones. Solving atomicity does nothing for visibility, and neither addresses ordering.

- **Atomicity** — An operation you think is one step is really several, and another thread can slip in between them.

A write by one thread may never become visible to another. Not "late" — potentially never.

The compiler and CPU reorder your instructions. Another thread can observe the results out of order.

### Atomicity: the counter that loses count

`count++` looks indivisible. It compiles to three separate operations: read the field, add one, write it back. Two threads interleaving those steps can both read the same starting value.

![Two thread timelines interleaving a read-modify-write on a counter, producing 1 instead of 2](diagrams/inside-java-threads-03.svg)

**The lost update.** Run two threads incrementing a shared `int` a million times each and you will reliably get less than two million. The interleaving above is one of many that lose data.

### Visibility: the loop that never ends

This one shocks people. Each CPU core has its own caches, and the JIT compiler is free to hoist a field read out of a loop entirely. Without a memory barrier, there is _no guarantee_ a write ever becomes visible to another thread.

```java
            class Stopper {
    private boolean stopped = false;     // ✗ not volatile

    void loop() {
        while (!stopped) { }                // may spin FOREVER
        System.out.println("stopped");
    }
    void stop() { stopped = true; }
}

// The JIT is entitled to rewrite the loop as:
//     if (!stopped) { while (true) { } }
// because within this thread, nothing changes `stopped`.
// That optimisation is legal, and it hangs your program.

// The fix is one keyword:
private volatile boolean stopped = false;

```

### Ordering: writes that arrive backwards

Compilers and CPUs reorder independent instructions for speed. Within one thread this is invisible — the language guarantees your own code _appears_ to run in order. Across threads, another observer can see the effects in a different sequence than you wrote them.

```java
            // Thread 1                    Thread 2
config = new Config();          if (ready) {
ready   = true;                      config.use();   // may see a HALF-BUILT Config
                                }
// Nothing stops the write to `ready` becoming visible
// BEFORE the writes that populate `config`.

```

### And the failure modes of coordination itself

- **Deadlock** — two threads each hold a lock the other needs. Both wait forever. Nothing recovers.
- **Livelock** — threads keep responding to each other and making no progress. Busy, not stuck; equally useless.
- **Starvation** — a thread never gets scheduled or never wins the lock, because others keep taking priority.

## The Java Memory Model

The specification that says which writes a thread is guaranteed to see. It's built from one relation: _happens-before_.

If action A **happens-before** action B, then everything A did is visible to B. If no happens-before relationship exists between two actions in different threads, the JMM makes _no promise at all_ — B might see A's work, might see part of it, might see none. That's not a bug in the JVM; it's the contract.

### What creates a happens-before edge

- Everything a thread does before `t.start()` is visible to `t`.
- Everything `t` does is visible after `t.join()` returns.
- Releasing a monitor happens-before any later acquisition of _that same_ monitor.
- A write to a `volatile` field happens-before every later read of that field.
- Anything visible to a task submitted to an executor, and anything the task did, visible to whoever reads its `Future`.
- `final` fields are safely visible after the constructor completes — provided `this` didn't escape during construction.

![A volatile write acting as a barrier that publishes all preceding writes to a second thread](diagrams/inside-java-threads-04.svg)

**A volatile write is a barrier, not just a field.** Reading a volatile variable makes every write the writer performed _before_ it visible too. This is why the flag pattern works — and why the flag must be volatile for the payload to be safe.

### What volatile does and does not do

| Guarantee                | volatile | synchronized | AtomicInteger |
| ------------------------ | -------- | ------------ | ------------- |
| Visibility of writes     | yes      | yes          | yes           |
| Prevents reordering      | yes      | yes          | yes           |
| Atomic read-modify-write | no       | yes          | yes           |
| Mutual exclusion         | no       | yes          | no            |
| Can block a thread       | no       | yes          | no            |

> ⚠️ **volatile is not a lock**
>
> `volatile int count; count++;` is still broken. Volatile guarantees you read the latest value — it does nothing to stop another thread reading the same latest value and racing you to write. For counters use `AtomicInteger`; for anything with an invariant across multiple fields, use a lock. Volatile is for _flags and single-field publication_, nothing more.

## Locks and coordination

Mutual exclusion: only one thread inside the critical section at a time. Java gives you a built-in version and a configurable one.

### synchronized — the built-in monitor

Every Java object has a **monitor**. `synchronized` acquires it on entry and releases it on exit — including when an exception unwinds, which is a real advantage. It's **reentrant**: a thread already holding a monitor can re-acquire it without deadlocking itself.

```java
            class Counter {
    private int count = 0;

    // Locks on `this` — the whole instance.
    synchronized void increment() { count++; }

    // Better: a private lock nobody outside can grab.
    private final Object lock = new Object();
    void safeIncrement() {
        synchronized (lock) { count++; }
    }
}
// A static synchronized method locks on ClassName.class instead.

```

Prefer the private lock object. `synchronized` methods lock on `this`, which is publicly reachable — any outside code can `synchronized (yourObject)` and interfere with your locking, deliberately or by accident.

### Deadlock, and the one rule that prevents it

![Two threads acquiring two locks in opposite order, forming a circular wait](diagrams/inside-java-threads-05.svg)

**Circular wait.** The JVM will not detect or break this; the threads stay parked until you kill the process. **The fix is a global lock ordering:** if every thread acquires locks in the same defined sequence, a cycle is impossible.

### wait / notify — the guarded block

Locks give exclusion. To make a thread wait _for a condition_, you need `wait()` and `notify()`, both of which must be called while holding the monitor.

```java
            synchronized (lock) {
    while (!conditionIsTrue) {   // ALWAYS while, never if
        lock.wait();             // releases the lock, parks the thread
    }
    proceed();
}

// elsewhere
synchronized (lock) {
    conditionIsTrue = true;
    lock.notifyAll();            // prefer over notify()
}

```

> ⚠️ **Why `while` and not `if`**
>
> **Spurious wakeups are permitted by the specification** — a thread can return from `wait()` with nobody having notified it. And with several waiters, another thread may consume the condition before you re-acquire the lock. Re-checking in a loop handles both. An `if` here is a latent bug that appears under load, months later.

### ReentrantLock and friends

Everything `synchronized` does, plus the things it can't: timed attempts, interruptible waiting, fairness, and multiple wait-sets on one lock.

```java
            ReentrantLock lock = new ReentrantLock();

lock.lock();
try { criticalSection(); }
finally { lock.unlock(); }        // finally is MANDATORY — no auto-release

// The capabilities synchronized simply doesn't have:
if (lock.tryLock(2, TimeUnit.SECONDS)) { … }   // give up rather than hang
lock.lockInterruptibly();                        // cancellable wait
new ReentrantLock(true);                        // FIFO fairness (slower)

// Many readers OR one writer — good for read-heavy shared state.
ReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock();    // concurrent with other readers
rw.writeLock().lock();   // exclusive

// StampedLock adds optimistic reads — no lock at all on the happy path,
// validated afterwards. Fast, but not reentrant. Read the javadoc first.

```

Default to `synchronized`: it's shorter, it can't leak a lock, and the JVM optimises it well. Reach for `ReentrantLock` when you specifically need a timeout, interruptibility, fairness, or more than one condition queue.

## The java.util.concurrent toolkit

Written by experts, tested harder than anything you will write. Reach for these before building your own.

### Atomics and compare-and-swap

Atomic classes get their safety from a CPU instruction rather than a lock. **Compare-and-swap** says: "set this to N, but only if it's still M." If another thread changed it, the swap fails and you retry with the new value. No blocking, no context switch.

![A compare and swap loop reading a value, computing a new one, and retrying when another thread intervenes](diagrams/inside-java-threads-06.svg)

**Optimistic, not pessimistic.** A lock assumes conflict and prevents it; CAS assumes no conflict and detects it. Under low contention CAS wins easily. Under heavy contention the retry loop burns CPU — which is exactly what `LongAdder` exists to fix.

```java
            AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();                  // atomic ++
counter.addAndGet(5);
counter.compareAndSet(10, 20);            // the raw primitive
counter.updateAndGet(v -> v * 2);          // CAS loop for you

AtomicReference<Config> cfg = new AtomicReference<>(initial);
cfg.updateAndGet(c -> c.withTimeout(30));

// Heavy write contention? LongAdder spreads across internal cells
// and sums on read. Much faster than AtomicLong for pure counting.
LongAdder hits = new LongAdder();
hits.increment();
hits.sum();

```

### Concurrent collections

| Use                       | Class                   | Why                                           |
| ------------------------- | ----------------------- | --------------------------------------------- |
| General shared map        | `ConcurrentHashMap`     | Per-bin locking; reads never block            |
| Sorted shared map         | `ConcurrentSkipListMap` | Lock-free, keeps keys ordered                 |
| Rarely-changing list      | `CopyOnWriteArrayList`  | Copies on every write — perfect for listeners |
| Hand work between threads | `LinkedBlockingQueue`   | Blocks when empty or full                     |
| Bounded handoff           | `ArrayBlockingQueue`    | Fixed capacity gives you backpressure         |
| Direct handoff            | `SynchronousQueue`      | Zero capacity; producer waits for a consumer  |
| Scheduled work            | `DelayQueue`            | Elements emerge only when due                 |

### Synchronizers

```java
            // CountDownLatch — wait for N things, once. Cannot be reset.
CountDownLatch ready = new CountDownLatch(3);
// workers: ready.countDown();
ready.await();                       // blocks until it hits zero

// CyclicBarrier — N threads meet at a point, then all continue. Reusable.
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("round done"));
barrier.await();

// Semaphore — at most N threads past this point. Rate limiting, pools.
Semaphore permits = new Semaphore(10);
permits.acquire();
try { callRateLimitedApi(); } finally { permits.release(); }

```

## Executors — stop managing threads yourself

Creating a thread per task doesn't scale: threads are expensive, and unbounded creation is how servers fall over. Submit tasks to a pool instead.

![Tasks entering a bounded queue, picked up by a fixed set of worker threads, with overflow going to a rejection handler](diagrams/inside-java-threads-07.svg)

**The queue is the part people get wrong.** `Executors.newFixedThreadPool(n)` uses an _unbounded_ queue — tasks pile up in memory without limit until the heap dies. For anything load-bearing, construct `ThreadPoolExecutor` directly with a bounded queue.

```java
            // The convenient factories — fine for scripts, risky for servers.
Executors.newFixedThreadPool(4);      // unbounded queue
Executors.newCachedThreadPool();       // unbounded THREADS
Executors.newSingleThreadExecutor();
Executors.newScheduledThreadPool(2);

// What production code should do — every parameter deliberate.
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    4,                                  // core threads, kept alive
    8,                                  // max threads
    60, TimeUnit.SECONDS,               // idle timeout for extras
    new ArrayBlockingQueue<>(100),      // BOUNDED
    new ThreadPoolExecutor.CallerRunsPolicy()  // natural backpressure
);

// Getting results back: Callable returns a value and may throw.
Future<Integer> f = pool.submit(() -> expensiveSum());
Integer result = f.get();          // BLOCKS until done
Integer orFail = f.get(5, TimeUnit.SECONDS);   // prefer a timeout

// Shutting down properly — a non-daemon pool keeps the JVM alive.
pool.shutdown();                                  // no new tasks; finish current
if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
    pool.shutdownNow();                           // interrupt stragglers
}

```

> **Sizing the pool**
>
> For **CPU-bound** work, roughly `Runtime.getRuntime().availableProcessors()` — more threads than cores just adds context switching. For **I/O-bound** work, many more than the core count, because most are parked waiting. The old formula is `cores × (1 + waitTime/computeTime)`. Virtual threads (Part 09) make this calculation mostly obsolete for the I/O case.

## CompletableFuture

A plain `Future` can only be blocked on. `CompletableFuture` lets you describe what happens next and walk away.

```java
            // Chain transformations — nothing blocks.
CompletableFuture.supplyAsync(() -> fetchUser(id))
    .thenApply(User::name)                 // transform the value
    .thenApply(String::toUpperCase)
    .thenAccept(System.out::println)      // consume, return nothing
    .exceptionally(ex -> {                  // recover from failure
        log.error("failed", ex);
        return null;
    });

// Combine two independent async results.
CompletableFuture<User>  user  = CompletableFuture.supplyAsync(() -> fetchUser(id));
CompletableFuture<Order> order = CompletableFuture.supplyAsync(() -> fetchOrder(id));

user.thenCombine(order, (u, o) -> new Summary(u, o))
    .thenAccept(System.out::println);

// thenCompose flattens when the next step is ITSELF async.
// (thenApply here would give you CompletableFuture<CompletableFuture<X>>.)
user.thenCompose(u -> CompletableFuture.supplyAsync(() -> loadProfile(u)));

// Wait for many.
CompletableFuture.allOf(user, order).join();
CompletableFuture.anyOf(mirrorA, mirrorB);      // first one wins

// Timeouts (Java 9+).
user.orTimeout(3, TimeUnit.SECONDS)
    .completeOnTimeout(User.GUEST, 2, TimeUnit.SECONDS);

```

> ⚠️ **Two traps worth knowing**
>
> **The default pool is the common ForkJoinPool**, sized to your core count and shared with parallel streams. Put blocking I/O in there and you starve everything else in the JVM. Pass your own executor: `supplyAsync(task, myPool)`.
>
> **Exceptions vanish if you never consume them.** A failed `CompletableFuture` with no `exceptionally`, `handle`, or terminal `join` swallows the error entirely — no stack trace, no log line, nothing. Always terminate the chain.

## Virtual threads

Final in Java 21, and the biggest change to Java concurrency in twenty years. The reason thread pools existed is largely gone.

A traditional **platform thread** maps 1:1 to an operating-system thread. It costs around a megabyte of stack and a system call to create, so you can afford thousands, not millions. That scarcity is _the_ reason for pooling — you were rationing a scarce resource.

A **virtual thread** is managed by the JVM instead. Its stack lives on the heap and grows as needed; creating one is roughly as cheap as allocating an object. Many virtual threads are multiplexed onto a small pool of **carrier** platform threads.

![Platform threads mapping one to one onto OS threads, versus many virtual threads mounting onto a few carrier threads](diagrams/inside-java-threads-08.svg)

**Blocking stops being expensive.** When a virtual thread blocks on I/O, the JVM unmounts it from its carrier and runs something else there. The blocked thread costs a heap object, not an idle OS thread — so "one thread per request" becomes viable again.

```java
            // One virtual thread per task. No pool, no sizing, no tuning.
try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Request r : requests) {
        exec.submit(() -> handle(r));     // a million of these is fine
    }
}

// Directly:
Thread vt = Thread.ofVirtual().start(() -> work());
Thread.ofVirtual().name("handler-", 0).factory();

// The code inside is ORDINARY BLOCKING CODE. That's the point —
// you get async scalability without writing async-style code.

```

> **What changes in your habits**
>
> **Don't pool virtual threads.** Pooling exists to reuse something expensive; these are cheap. A pool of virtual threads is strictly worse than one per task.
>
> **Use them for blocking I/O, not CPU work.** They don't add compute capacity — the carrier count is still bounded by your cores. For CPU-bound parallelism, a platform-thread pool or parallel streams remains the right tool.
>
> **Semaphores replace pool sizing.** If you needed a pool of 10 to limit database connections, use a `Semaphore` with 10 permits and unlimited virtual threads instead.

One historical caveat worth knowing if you're on an older runtime: in Java 21 through 23, blocking inside a `synchronized` block would **pin** the virtual thread to its carrier, defeating the mechanism — the standing advice was to prefer `ReentrantLock` in hot paths. Java 24 removed that limitation, so on current runtimes `synchronized` is fine again. Native calls and some legacy code can still pin.

### Structured concurrency and scoped values

Two companion features from the same effort. **Structured concurrency** treats a group of related tasks as a single unit — if one fails, siblings are cancelled; the scope doesn't exit until all children finish. It removes the classic leak where a failed request leaves orphaned background work running. **Scoped values** are an immutable, better-behaved alternative to `ThreadLocal` for passing context down a call tree, and they don't leak in pooled threads.

Check your JDK's release notes for exact status before relying on the structured-concurrency API — it went through several preview rounds with a changing surface, whereas scoped values landed as a final feature in Java 25.

## Patterns that work

### Immutability beats every locking scheme

An object that cannot change cannot be raced. No locks, no volatile, no reasoning about interleavings. This is the single highest-leverage habit in concurrent Java, and records make it nearly free.

```java
            // Deeply safe: final fields, no setters, defensive copy on the way in.
record Point(int x, int y) { }

record Config(String host, List<String> tags) {
    Config {
        tags = List.copyOf(tags);   // nobody can mutate it later
    }
}

// "Change" = build a new one and swap the reference atomically.
AtomicReference<Config> current = new AtomicReference<>(initial);
current.updateAndGet(c -> new Config(newHost, c.tags()));

```

### Confinement — don't share in the first place

If only one thread can reach an object, it needs no synchronisation. Give each thread its own copy, or keep mutable state local to a method.

```java
            // SimpleDateFormat is notoriously not thread-safe. Give each thread one.
static final ThreadLocal<SimpleDateFormat> FMT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

String s = FMT.get().format(new Date());

// In a POOLED thread you must clean up, or the value leaks into
// whatever task runs next on that thread.
try { … } finally { FMT.remove(); }

// Better still — DateTimeFormatter is immutable and thread-safe.
static final DateTimeFormatter F = DateTimeFormatter.ISO_DATE;

```

### Producer–consumer with a blocking queue

```java
            BlockingQueue<Job> queue = new ArrayBlockingQueue<>(100);

// Producer — blocks when full, which is the backpressure you want.
queue.put(job);

// Consumer — blocks when empty. No polling, no sleep loop.
while (!Thread.currentThread().isInterrupted()) {
    Job job = queue.take();
    process(job);
}

// A POISON PILL is the clean shutdown signal.
static final Job STOP = new Job();
// producer: queue.put(STOP);
// consumer: if (job == STOP) break;

```

### Testing and debugging

- **A passing test proves nothing.** Race conditions are timing-dependent; a green run may just mean you got lucky. Run assertions in loops with many threads, and repeat.
- **Thread dumps** — `jstack <pid>` or Ctrl+\. The JVM detects and reports deadlocks explicitly in the dump; look for "Found one Java-level deadlock".
- **jcstress** is the OpenJDK harness built specifically for finding memory-model bugs. If you're writing lock-free code, use it.
- **Name your threads.** `new Thread(task, "order-processor-1")` turns an unreadable dump into a readable one. Pools take a `ThreadFactory` for this.

## Ways to get hurt

### Double-checked locking without volatile

The canonical broken idiom. Without `volatile`, another thread can see a non-null reference to an object whose constructor hasn't finished — because the write publishing the reference may be reordered ahead of the writes initialising the fields.

```java
            // ✗ BROKEN — instance may be visible before it is fully constructed
private static Singleton instance;
static Singleton get() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) instance = new Singleton();
        }
    }
    return instance;
}

// ✓ Correct — one keyword
private static volatile Singleton instance;

// ✓✓ Better — the holder idiom. The JVM guarantees class
//     initialisation is thread-safe and lazy. No locks in your code.
private static class Holder {
    static final Singleton INSTANCE = new Singleton();
}
static Singleton get() { return Holder.INSTANCE; }

```

### Compound operations on thread-safe collections

Each individual call is atomic. Two calls in sequence are not.

```java
            ConcurrentMap<String, Integer> map = new ConcurrentHashMap<>();

// ✗ Two atomic calls do NOT make an atomic sequence.
if (!map.containsKey(k)) map.put(k, v);      // another thread can interleave
map.put(k, map.get(k) + 1);                  // lost update

// ✓ Single atomic operations
map.putIfAbsent(k, v);
map.merge(k, 1, Integer::sum);
map.computeIfAbsent(k, key -> expensive(key));

```

### The rest of the list

| Mistake                                  | What happens                                                     | Fix                                      |
| ---------------------------------------- | ---------------------------------------------------------------- | ---------------------------------------- |
| Sharing a `HashMap` across threads       | Lost entries, corrupted internals, undefined behaviour           | `ConcurrentHashMap`                      |
| Swallowing `InterruptedException`        | Cancellation silently stops working                              | Rethrow, or restore the flag             |
| Never calling `shutdown()`               | JVM never exits — non-daemon pool threads hold it open           | try-with-resources (19+), or a `finally` |
| `ThreadLocal` in a pooled thread         | Value leaks into the next task; classloader leaks in app servers | `remove()` in `finally`                  |
| Blocking inside `computeIfAbsent`        | Bin lock held during I/O; possible deadlock                      | Compute outside, then insert             |
| Blocking work on the common ForkJoinPool | Starves parallel streams and every other user                    | Pass an explicit executor                |
| Calling `get()` with no timeout          | One stuck task hangs the caller indefinitely                     | `get(n, TimeUnit…)`                      |
| Locks acquired in inconsistent order     | Deadlock under load, never in testing                            | One global lock ordering                 |
| `if` instead of `while` around `wait()`  | Proceeds on a spurious wakeup with the condition false           | Always loop                              |

## A learning path

Roughly the order that builds correctly on itself. Write code at each step — this subject does not stick from reading.

1. **Threads and the lifecycle** — Start threads, join them, watch states. Write something that interleaves visibly.
2. **Break something on purpose** — Two threads, one unsynchronised counter, a million increments each. Watch the total come out wrong. This makes the rest concrete.
3. **Fix it three ways** — `synchronized`, then `AtomicInteger`, then `LongAdder`. Understand why all three work and how they differ.
4. **Visibility and volatile** — Reproduce the non-terminating loop. Fix it with `volatile`. Learn what volatile does _not_ do.
5. **happens-before** — The mental model everything else rests on. Come back to this repeatedly.
6. **Executors and Future** — Stop creating threads by hand. Learn pool sizing and why unbounded queues are dangerous.
7. **Concurrent collections** — `ConcurrentHashMap` and `BlockingQueue`. Build a producer - consumer pipeline.
8. **Cause a deadlock** — Two locks, opposite order. Then find it in a `jstack` dump. This skill pays for itself.
9. **CompletableFuture** — Compose async work without blocking. Watch out for the common pool.
10. **Virtual threads** — Java 21+. Re-solve an earlier problem with one thread per task and compare.
11. **Immutability as strategy** — Go back and redesign something so the concurrency problem disappears rather than getting solved.

> **Worth reading**
>
> _Java Concurrency in Practice_ (Goetz et al.) is still the definitive book despite predating everything from Part 08 onward — the reasoning about the memory model, publication, and design has not aged. Pair it with the JEPs for virtual threads and structured concurrency to fill the modern gap.

### The short version

- Don't share mutable state. Immutability and confinement eliminate problems instead of managing them.
- When you must share, use `java.util.concurrent` rather than writing your own.
- When you must lock, keep the critical section small, never do I/O inside it, and always acquire in a consistent order.
- `volatile` is for visibility of a single field. It is not a lock.
- Every blocking call deserves a timeout.
- On Java 21+, one virtual thread per task beats a carefully tuned pool for I/O-bound work.
