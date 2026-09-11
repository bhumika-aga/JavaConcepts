# Inside Stacks and Queues

_stacks · queues · heaps · and the memory they live in_

> Java uses “stack” for two unrelated things and “heap” for two more. Half the confusion in this topic is vocabulary; the other half is knowing which container gives you which guarantee, and what it costs.

---

## Contents

1. [Two words, four things](#two-words-four-things)
2. [The call stack](#the-call-stack)
3. [The heap](#the-heap)
4. [When each one breaks](#when-each-one-breaks)
5. [Stack — last in, first out](#stack--last-in-first-out)
6. [Queue — first in, first out](#queue--first-in-first-out)
7. [Deque — open at both ends](#deque--open-at-both-ends)
8. [PriorityQueue](#priorityqueue)
9. [The binary heap](#the-binary-heap)
10. [BlockingQueue](#blockingqueue)
11. [Compare them all](#compare-them-all)
12. [Ways to get hurt](#ways-to-get-hurt)

---

## Two words, four things

Before anything else, separate the memory regions from the data structures. They share names and share nothing else.

| The word | Sense 1 — a memory region                                         | Sense 2 — a data structure                                              |
| -------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `stack`  | Per-thread store of method frames and locals. The JVM manages it. | A LIFO container you create and push onto.                              |
| `heap`   | The shared region where every object lives. The GC manages it.    | A partially-ordered tree that always yields its smallest element first. |

> **They are not related**
>
> The memory stack happens to behave like the LIFO structure — frames push and pop — which is where the shared name came from. The memory heap and the binary heap share _nothing_ but a word; the memory sense means "an unordered pile of storage", the structure sense means "a specific ordered tree". Keep them apart and this whole topic gets easier.

## The call stack

Every thread gets its own. Each method call pushes a **frame** holding that call's local variables, parameters, and return address. Return, and the frame pops.

| Property         | Value                |
| ---------------- | -------------------- |
| **Scope**        | One per thread       |
| **Holds**        | Frames, locals, refs |
| **Cleanup**      | Automatic            |
| **Shared**       | Never                |
| **Typical size** | 512KB – 1MB          |
| **Overflow**     | StackOverflowError   |

![A call stack with three frames, the newest on top, each holding its own locals](diagrams/inside-stacks-and-queues-01.svg)

**Why locals are automatically thread-safe.** The stack is private to one thread by construction, so nothing on it needs synchronising. Only what the frames _point at_ — objects on the heap — can be shared.

```java
            // The classic overflow: recursion with no base case
static int depth(int n) { return depth(n + 1); }
depth(0);                       // StackOverflowError in milliseconds

// Stack size is a JVM flag, not an API:  java -Xss2m MyApp
// Raising it treats the symptom. Deep recursion usually wants a loop
// plus an explicit Deque instead.

```

## The heap

One region, shared by every thread, holding every object you ever allocate. You never free anything — the garbage collector reclaims what nothing can still reach.

| Property       | Value             |
| -------------- | ----------------- |
| **Scope**      | One per JVM       |
| **Holds**      | Every object      |
| **Cleanup**    | Garbage collector |
| **Shared**     | By all threads    |
| **Sized with** | -Xmx              |
| **Exhaustion** | OutOfMemoryError  |

![Two thread stacks holding references that both point at one shared object on the heap](diagrams/inside-stacks-and-queues-02.svg)

**Variables hold references, not objects.** Assigning one variable to another copies the reference — both then point at the same heap object, and a change through either is visible through both.

## When each one breaks

- **StackOverflowError** — Too many nested frames. Nearly always unbounded recursion — a missing or unreachable base case.

Live objects exceed `-Xmx`. Either a genuine working set that's too big, or a leak holding references it shouldn't.

Not the heap at all — each platform thread needs its own stack, and the OS ran out.

Both are `Error`, not `Exception`: don't catch them to continue. A `StackOverflowError` is sometimes survivable, but an `OutOfMemoryError` usually means the JVM is already unable to do useful work.

## Stack — last in, first out

Push onto the top, pop off the top. The most recently added element is the only one you can reach.

| Property         | Value            |
| ---------------- | ---------------- |
| **Use**          | ArrayDeque       |
| **push / pop**   | `O(1) amortised` |
| **Order out**    | Reverse of in    |
| **Allows null**  | No               |
| **Thread-safe**  | No               |
| **Legacy class** | Avoid Stack      |

![A stack where three elements are pushed and popped from the same top end](diagrams/inside-stacks-and-queues-03.svg)

**The call stack is this structure.** So are undo buttons, browser history, bracket matching, and every iterative rewrite of a recursive algorithm.

```java
            // ✓ The modern way
Deque<String> stack = new ArrayDeque<>();
stack.push("A");          // addFirst
stack.push("B");
stack.peek();               // "B" — look, don't remove
stack.pop();                // "B" — removeFirst

// ✗ The 1996 way. Synchronized on every call, extends Vector,
//   and iterates BOTTOM-UP — the opposite of pop order.
Stack<String> old = new Stack<>();

// Turning recursion into iteration — the standard use
Deque<Node> todo = new ArrayDeque<>();
todo.push(root);
while (!todo.isEmpty()) {
    Node n = todo.pop();
    visit(n);
    for (Node child : n.children()) todo.push(child);
}

```

> ⚠️ **The `Stack` class iterates the wrong way round**
>
> `java.util.Stack` extends `Vector`, so iterating it walks from the _bottom_ — you see insertion order, not pop order. Code that pops in one place and iterates in another quietly disagrees with itself. `ArrayDeque` iterates head-first, matching `pop()`.

## Queue — first in, first out

Add at the tail, remove from the head. Arrival order is preserved exactly, which is what makes it fair.

![A queue where elements enter at the tail and leave from the head in arrival order](diagrams/inside-stacks-and-queues-04.svg)

### Two method families, different tempers

Every `Queue` operation comes in two forms: one that throws on failure and one that returns a sentinel. Mixing them is a common source of surprise.

| Operation | Throws on failure | Returns special value |
| --------- | ----------------- | --------------------- |
| Insert    | `add(e)`          | `offer(e) → false`    |
| Remove    | `remove()`        | `poll() → null`       |
| Examine   | `element()`       | `peek() → null`       |

On an unbounded queue `add` and `offer` behave identically, so the distinction only bites once you switch to a bounded implementation — typically in production, under load.

## Deque — open at both ends

A double-ended queue. Because you can add and remove at either end, it can act as a stack, a queue, or both — which is why it replaced two older classes.

```java
            Deque<String> d = new ArrayDeque<>();

// explicit, unambiguous — prefer these in shared code
d.addFirst("x");   d.addLast("y");
d.pollFirst();      d.pollLast();
d.peekFirst();      d.peekLast();

// stack aliases  → push/pop/peek act on the FIRST element
// queue aliases  → offer/poll/peek act on first-in-first-out

```

| Implementation          | Backing             | null allowed | Notes                                                  |
| ----------------------- | ------------------- | ------------ | ------------------------------------------------------ |
| `ArrayDeque`            | Circular array      | no           | The default. Faster than LinkedList for both roles.    |
| `LinkedList`            | Doubly-linked nodes | yes          | Also a List. Node overhead per element, poor locality. |
| `ConcurrentLinkedDeque` | Lock-free nodes     | no           | Thread-safe, non-blocking.                             |

> **Why `ArrayDeque` rejects null**
>
> It isn't an oversight. `poll()` and `peek()` return `null` to mean "empty" — so a stored `null` would make that answer ambiguous. `LinkedList` permits nulls and inherits exactly that ambiguity, which is one more reason to reach for `ArrayDeque`.

## PriorityQueue

A queue that ignores arrival order entirely. Whatever is _smallest_ by the ordering comes out next — regardless of when it went in.

| Property                 | Value           |
| ------------------------ | --------------- |
| **Backing**              | Binary min-heap |
| **offer / poll**         | `O(log n)`      |
| **peek**                 | `O(1)`          |
| **contains / remove(o)** | `O(n)`          |
| **Iteration order**      | Unspecified     |
| **Allows null**          | No              |

```java
            // Natural ordering — smallest first
Queue<Integer> pq = new PriorityQueue<>();
pq.addAll(List.of(5, 1, 9, 3));
pq.poll();    // 1
pq.poll();    // 3

// Largest first — reverse the comparator
new PriorityQueue<Integer>(Comparator.reverseOrder());

// Real use: most urgent task next
Queue<Task> work =
    new PriorityQueue<>(Comparator.comparingInt(Task::urgency).reversed());

// The top-K idiom: keep a MIN-heap of size k, evict the smallest
PriorityQueue<Integer> topK = new PriorityQueue<>();
for (int v : values) {
    topK.offer(v);
    if (topK.size() > k) topK.poll();   // O(n log k), not O(n log n)
}

```

> ⚠️ **It is not a sorted collection**
>
> Only `peek` and `poll` respect the ordering. `toString()`, `forEach`, and the iterator walk the **internal array**, whose middle is only partially ordered. Printing a `PriorityQueue` and seeing `[1, 3, 9, 5]` is not a bug. If you need every element in order, drain it with repeated `poll()`, or use a `TreeSet`.

## The binary heap

The structure inside `PriorityQueue`. It is deliberately only _half_ sorted — and that's precisely why it's fast.

The single invariant: **every parent is ≤ both of its children.** Siblings are never compared to each other. That's weak enough to maintain cheaply and strong enough to guarantee the minimum is always at the root.

![A binary min-heap drawn as a tree and as the flat array that actually stores it, with the parent and child index formulas](diagrams/inside-stacks-and-queues-05.svg)

**A complete tree fits an array exactly.** No gaps, so position alone encodes the structure — no child pointers to store or follow, and excellent cache behaviour.

### The two operations

- **Insert — sift up.** Put the new element at the end of the array, then swap it with its parent while it's smaller. At most `log n` swaps.
- **Remove min — sift down.** Take the root, move the last element into its place, then swap it with its smaller child until the invariant holds. Again `log n`.

Both touch a single root-to-leaf path, which is why a heap beats a fully sorted structure for this job: you pay only to find the next winner, never to order the losers.

## BlockingQueue

A queue that makes threads wait instead of failing. The backbone of every producer–consumer pipeline, and what thread pools use internally.

```java
            BlockingQueue<Job> q = new ArrayBlockingQueue<>(100);

q.put(job);              // blocks while FULL   — natural backpressure
Job j = q.take();        // blocks while EMPTY  — no polling loop

q.offer(job, 2, TimeUnit.SECONDS);   // or give up
q.poll(2, TimeUnit.SECONDS);

```

| Implementation          | Capacity                                      | Use it when                                       |
| ----------------------- | --------------------------------------------- | ------------------------------------------------- |
| `ArrayBlockingQueue`    | Fixed, set at construction                    | You want a hard limit and real backpressure       |
| `LinkedBlockingQueue`   | Optionally bounded — **unbounded by default** | Higher throughput; always pass a capacity         |
| `SynchronousQueue`      | Zero                                          | Direct handoff; the producer waits for a consumer |
| `PriorityBlockingQueue` | Unbounded                                     | Urgency ordering across threads                   |
| `DelayQueue`            | Unbounded                                     | Elements become available only once due           |

> ⚠️ **The unbounded default is a production hazard**
>
> `new LinkedBlockingQueue<>()` has capacity `Integer.MAX_VALUE`. `put` then never blocks, so a fast producer piles work into the heap until the JVM dies — and `Executors.newFixedThreadPool` uses exactly this queue. Always pass a capacity for anything load-bearing.

```java
            // Producer–consumer with a clean shutdown signal
static final Job POISON = new Job();

// consumer
while (true) {
    Job j = q.take();
    if (j == POISON) { q.put(POISON); break; }   // pass it on
    process(j);
}

```

## Compare them all

| Type                    | Order out     | `add`      | `remove`   | `peek` | null | Safe   |
| ----------------------- | ------------- | ---------- | ---------- | ------ | ---- | ------ |
| `ArrayDeque (stack)`    | Reverse of in | `O(1)*`    | `O(1)`     | `O(1)` | no   | no     |
| `ArrayDeque (queue)`    | Arrival       | `O(1)*`    | `O(1)`     | `O(1)` | no   | no     |
| `LinkedList`            | Arrival       | `O(1)`     | `O(1)`     | `O(1)` | yes  | no     |
| `PriorityQueue`         | By comparator | `O(log n)` | `O(log n)` | `O(1)` | no   | no     |
| `ArrayBlockingQueue`    | Arrival       | `O(1)`     | `O(1)`     | `O(1)` | no   | yes    |
| `LinkedBlockingQueue`   | Arrival       | `O(1)`     | `O(1)`     | `O(1)` | no   | yes    |
| `ConcurrentLinkedQueue` | Arrival       | `O(1)`     | `O(1)`     | `O(1)` | no   | yes    |
| `Stack (legacy)`        | Reverse of in | `O(1)*`    | `O(1)`     | `O(1)` | yes  | locked |

- amortised — an array-backed structure occasionally doubles its capacity.

### Picking one

| I want to…                     | Use                       |
| ------------------------------ | ------------------------- |
| I need last-in-first-out       | `ArrayDeque`              |
| I need first-in-first-out      | `ArrayDeque`              |
| Most urgent item next          | `PriorityQueue`           |
| Hand work between threads      | `ArrayBlockingQueue`      |
| Sorted _and_ fully traversable | `TreeSet / TreeMap`       |
| Both ends, lock-free           | `ConcurrentLinkedDeque`   |
| Top K of a large stream        | `PriorityQueue of size k` |

## Ways to get hurt

### Printing a PriorityQueue and believing it

The array is heap-ordered, not sorted. Only the head is guaranteed. This is the single most common misunderstanding in this topic, and it survives testing because small heaps often _look_ sorted by accident.

### Using `java.util.Stack`

Synchronized on every operation whether you need it or not, extends `Vector` so it exposes index-based mutation that has no business on a stack, and iterates in the opposite order from `pop()`. `ArrayDeque` is faster and better behaved in every respect.

### Assuming `LinkedBlockingQueue` is bounded

It isn't, unless you say so. Combined with a fixed thread pool, an unbounded queue converts a throughput problem into an `OutOfMemoryError` — and hides the backlog until the moment it kills you.

### Mutating an element that's already in a PriorityQueue

```java
            task.setUrgency(10);      // the heap does NOT re-order itself
// The invariant is now broken and poll() may return the wrong task.

// ✓ remove, mutate, re-insert
pq.remove(task);            // O(n)
task.setUrgency(10);
pq.offer(task);             // O(log n)

```

### Catching StackOverflowError to "handle" recursion

The stack is already exhausted, so your handler has very little room to run in, and any state built during the descent is in an unknown condition. Fix the recursion or convert it to an explicit `Deque` loop.

### Reaching for `LinkedList` out of habit

It's a fine `Deque` and a poor `List`. Every element is a separate node with two pointers, so it uses far more memory than `ArrayDeque` and scatters across the heap. Unless you specifically need `null` elements or the `List` interface, prefer `ArrayDeque`.

> **The short version**
>
> Two memory regions and three container shapes, sharing two unfortunate names. Locals live on a private stack and are safe by construction; objects live on the shared heap and are not. For containers: `ArrayDeque` for both LIFO and FIFO, `PriorityQueue` when order of importance beats order of arrival, and a bounded `BlockingQueue` whenever two threads are involved.
