# Inside Trees

_trees in java · from traversal to red-black_

> A tree trades the flat O(1) of a hash table for something a hash table can never give you: _order_. Everything below — balancing, rotations, the colour bits in `TreeMap` — exists to keep that trade honest.

---

## Contents

1. [Anatomy of a tree](#anatomy-of-a-tree)
2. [Binary search trees](#binary-search-trees)
3. [The four traversals](#the-four-traversals)
4. [Why balance matters](#why-balance-matters)
5. [Rotations](#rotations)
6. [Red-black trees](#red-black-trees)
7. [TreeMap and TreeSet](#treemap-and-treeset)
8. [The ordering contract](#the-ordering-contract)
9. [Trees hiding elsewhere](#trees-hiding-elsewhere)
10. [Tries](#tries)
11. [B-trees](#b-trees)
12. [Recursion and depth](#recursion-and-depth)
13. [Complexity](#complexity)
14. [Ways to get hurt](#ways-to-get-hurt)

---

## Anatomy of a tree

Nodes connected so that every node has exactly one parent, except one that has none. No cycles, no second route to anywhere.

![A labelled tree showing root, internal nodes, leaves, an edge, a subtree, and the height and depth measurements](diagrams/inside-trees-01.svg)

**Every subtree is a tree.** That self-similarity is why almost every tree algorithm is three lines of recursion — you handle one node and delegate the rest to yourself.

```java
// The whole data structure. There is nothing else to it.
class Node<T> {
    T value;
    Node<T> left, right;
    Node(T value) { this.value = value; }
}

// Counting nodes — the shape of nearly every tree method you'll write
static int size(Node<?> n) {
    return n == null ? 0 : 1 + size(n.left) + size(n.right);
}

static int height(Node<?> n) {
    return n == null ? -1 : 1 + Math.max(height(n.left), height(n.right));
}
```

## Binary search trees

Add one rule to a binary tree and it becomes searchable: **everything left is smaller, everything right is larger** — and that holds at every node, not just the root.

![A binary search tree with the path to a target value highlighted, discarding half the remaining tree at each step](diagrams/inside-trees-02.svg)

**The invariant is recursive.** It isn't "the left child is smaller" — it's that _every_ value in the left subtree is smaller. Checking only the immediate children is the classic wrong answer to "validate this BST".

```java
// Search — three lines, and the whole point of the structure
static Node find(Node n, int key) {
    if (n == null || n.value == key) return n;
    return key < n.value ? find(n.left, key) : find(n.right, key);
}

// Validating one — note the RANGE, not just the parent
static boolean isBst(Node n, long min, long max) {
    if (n == null) return true;
    if (n.value <= min || n.value >= max) return false;
    return isBst(n.left, min, n.value) && isBst(n.right, n.value, max);
}
// call with Long.MIN_VALUE, Long.MAX_VALUE
```

## The four traversals

Same tree, four visiting orders, four different uses. The only thing that changes in the first three is _when you touch the node_ relative to the recursive calls.

![One tree traversed four ways, showing the output sequence produced by in-order, pre-order, post-order and level-order](diagrams/inside-trees-03.svg)

**In-order on a BST yields sorted output.** That single fact is why `TreeMap` iterates in key order for free — no sorting step, just a walk.

```java
// Move one line and you change the traversal.
static void inOrder(Node n, Consumer<Node> visit) {
    if (n == null) return;
    inOrder(n.left, visit);
    visit.accept(n);                // ← in the middle
    inOrder(n.right, visit);
}

// Level-order needs a queue, not the call stack
static void levelOrder(Node root, Consumer<Node> visit) {
    if (root == null) return;
    Deque<Node> q = new ArrayDeque<>();
    q.add(root);
    while (!q.isEmpty()) {
        Node n = q.poll();
        visit.accept(n);
        if (n.left  != null) q.add(n.left);
        if (n.right != null) q.add(n.right);
    }
}

// Iterative in-order — an explicit stack, no recursion depth limit
Deque<Node> stack = new ArrayDeque<>();
Node cur = root;
while (cur != null || !stack.isEmpty()) {
    while (cur != null) { stack.push(cur); cur = cur.left; }
    cur = stack.pop();
    visit.accept(cur);
    cur = cur.right;
}
```

Swap the queue for a stack in `levelOrder` and you get depth-first instead — the container is the algorithm. That's the same trick as the recursion-to-iteration conversion in [**Inside Stacks and Queues**](04-inside-stacks-and-queues.html).

## Why balance matters

A BST's O(log n) is a promise about _shape_, not about the structure. Insert sorted data into a plain BST and you rebuild a linked list with extra pointers.

![The same five keys inserted in sorted order producing a degenerate chain, versus balanced insertion producing a shallow tree](diagrams/inside-trees-04.svg)

**Sorted input is the worst case, and it's common.** Importing records by id, replaying a time series, loading a sorted file — all produce exactly this. A self-balancing tree restructures as it goes so the shape can't degrade.

## Rotations

The one operation every self-balancing tree is built from. It re-hangs three pointers, changes the height, and **preserves the ordering** — which is what makes it safe.

![A left rotation moving the right child up and the parent down, with the in-order sequence unchanged before and after](diagrams/inside-trees-05.svg)

**Ordering survives, height changes.** Subtree `b` moves from C's left to P's right, which is exactly where it still belongs. AVL and red-black trees differ only in _when_ they decide to rotate.

## Red-black trees

One extra bit per node, four rules, and a guaranteed height of at most 2·log₂(n+1). This is what `TreeMap` and `TreeSet` are.

| #   | Rule                                                       | Why it's there                  |
| --- | ---------------------------------------------------------- | ------------------------------- |
| `1` | Every node is red or black                                 | The extra bit                   |
| `2` | The root is black                                          | Simplifies the cases            |
| `3` | A red node's children are black                            | No two reds in a row            |
| `4` | Every root-to-leaf path has the same number of black nodes | **The one that forces balance** |

Rules 3 and 4 together do the work. If every path has the same black count, and reds can never be adjacent, the longest possible path is at most twice the shortest — alternating red and black against an all-black path. That bound is the guarantee.

|                   | AVL                   | Red-black                                |
| ----------------- | --------------------- | ---------------------------------------- |
| Balance condition | Heights differ by ≤ 1 | The four rules above                     |
| Height bound      | ~1.44 log n — tighter | 2 log n                                  |
| Lookups           | Slightly faster       | Slightly slower                          |
| Insert / delete   | More rotations        | Fewer rotations                          |
| In the JDK        | No                    | TreeMap, TreeSet, treeified HashMap bins |

The JDK chose red-black because mixed workloads dominate: a little slower to search, meaningfully cheaper to modify. AVL wins when reads vastly outnumber writes — which is common enough in databases, and rare enough in general-purpose collections.

## TreeMap and TreeSet

The JDK's sorted collections. Everything they offer beyond `HashMap` comes from one property: the keys are in order, all the time.

| Property               | Value                       |
| ---------------------- | --------------------------- |
| **Structure**          | Red-black tree              |
| **get / put / remove** | `O(log n) always`           |
| **Iteration**          | Sorted by key               |
| **null key**           | NPE by default              |
| **TreeSet is**         | A TreeMap with dummy values |
| **Thread-safe**        | No                          |

```java
NavigableMap<Integer, String> m = new TreeMap<>();
m.put(10, "a"); m.put(20, "b"); m.put(30, "c");

// The queries a HashMap simply cannot answer
m.floorKey(25);        // 20 — greatest key ≤ 25
m.ceilingKey(25);      // 30 — least key ≥ 25
m.lowerKey(20);        // 10 — strictly less
m.higherKey(20);       // 30 — strictly greater
m.firstEntry();         // 10=a
m.headMap(20, true);  // {10=a, 20=b} — a live VIEW, not a copy
m.descendingMap();      // reversed view
m.pollFirstEntry();     // remove and return the smallest

// TreeSet is the same tree with the values thrown away
NavigableSet<String> s = new TreeSet<>(Comparator.comparing(String::length));
s.subSet("aa", "zz");
```

> **Range views are live and cheap**
>
> `headMap`, `tailMap` and `subMap` return _views_ backed by the original tree — no copying, and writes pass through in both directions. `map.headMap(cutoff).clear()` deletes that whole range from the underlying map, which is the tidiest expiry idiom in the JDK.

The full `TreeMap` treatment — including how it compares against every other Map implementation — is in [**Inside Java Maps**](01-inside-java-maps.html).

## The ordering contract

A tree map has no use for `hashCode()` or `equals()`. Two keys are "the same key" exactly when the comparison returns zero — and that catches people out.

```java
// The comparator decides identity, not equals()
TreeMap<String, Integer> ci = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
ci.put("Apple", 1);
ci.put("APPLE", 2);      // compares equal → overwrites
ci.size();                 // 1  — a HashMap would hold 2

// A comparator that only looks at part of the object silently
// collapses distinct entries:
new TreeSet<>(Comparator.comparing(Person::lastName));
// every Smith after the first is dropped — same "key"

// ✓ Make it total: break ties on something unique
Comparator.comparing(Person::lastName).thenComparing(Person::id);

// Null keys fail, because comparing null throws
m.put(null, "x");         // NullPointerException
```

> ⚠️ **Mutating a key corrupts the tree**
>
> The key's position was chosen by comparisons made at insert time. Change a field the comparator reads and the node is now in the wrong place — searches walk past it, and the entry becomes unreachable while still occupying memory. Exactly the same hazard as mutating a `HashMap` key, with a different mechanism. Use immutable keys.

## Trees hiding elsewhere

Several things you use daily are trees without saying so.

- **Treeified HashMap bins** — A bucket with 8+ collisions converts to a red-black tree, so a bad `hashCode()` degrades to O(log n) rather than O(n).

A binary heap — a tree by shape, stored in a flat array. Only the root is ordered; it is _not_ a BST.

N-ary trees. `Files.walk()` is a depth-first traversal returning a lazy stream.

The heap is covered properly in [**Inside Stacks and Queues**](04-inside-stacks-and-queues.html), and treeification in [**Inside Java Maps**](01-inside-java-maps.html).

## Tries

A prefix tree. The key isn't stored in a node — it's spelled out by the _path_ you take to reach it.

![A trie storing car, cat and dog, where the path spells the word and terminal nodes are marked](diagrams/inside-trees-06.svg)

**Prefix search is the whole point.** A `TreeMap` can do `subMap("ca", "cb")` for the same result in O(log n); a trie does it in O(m) and shares storage between common prefixes. For a dictionary of similar words, the trie wins on both.

## B-trees

The tree your database uses. Same idea as a BST, reshaped around the fact that reading from disk has a fixed minimum cost.

A binary node holds one key and costs one read. If a read costs 8KB regardless, fetching one key wastes the other 8,000 bytes. A B-tree node instead holds _hundreds_ of keys — one page worth — so the fanout is enormous and the tree is extremely shallow.

|                       | Binary tree | B-tree (fanout ~200)         |
| --------------------- | ----------- | ---------------------------- |
| Keys per node         | 1           | Hundreds                     |
| Height for 1M keys    | ~20         | ~3                           |
| Disk reads per lookup | ~20         | ~3                           |
| Built for             | Memory      | Block storage                |
| Used by               | TreeMap     | Postgres, MySQL, filesystems |

In a **B+ tree** — the variant most databases actually use — all values live in the leaves, and the leaves are linked. That gives you a shallow tree for point lookups _and_ a linked list for range scans, which is why `WHERE id BETWEEN 100 AND 200` is fast. There's no B-tree in the JDK, because the JDK's collections live in memory where the fanout argument doesn't apply.

## Recursion and depth

Tree code is naturally recursive, and recursion depth is bounded by tree height — which is fine until the tree isn't balanced.

```java
// Balanced: 1,000,000 nodes → depth ~20. Never a problem.
// Degenerate: 1,000,000 nodes → depth 1,000,000 → StackOverflowError.

// A TreeMap is safe by construction; your own BST is not.
// If the input order is untrusted, iterate instead of recursing:
Deque<Node> stack = new ArrayDeque<>();
stack.push(root);
while (!stack.isEmpty()) {
    Node n = stack.pop();
    visit(n);
    if (n.right != null) stack.push(n.right);
    if (n.left  != null) stack.push(n.left);   // pushed last → popped first
}
```

Java has no tail-call optimisation, so a recursive traversal always consumes real stack frames. The heap-allocated `ArrayDeque` above is bounded by available memory rather than by `-Xss`, which is a much larger budget. See [**Inside Stacks and Queues**](04-inside-stacks-and-queues.html) for why.

## Complexity

| Structure           | `search`   | `insert`   | `delete`            | Ordered?  |
| ------------------- | ---------- | ---------- | ------------------- | --------- |
| `BST (balanced)`    | `O(log n)` | `O(log n)` | `O(log n)`          | yes       |
| `BST (degenerate)`  | `O(n)`     | `O(n)`     | `O(n)`              | yes       |
| `TreeMap / TreeSet` | `O(log n)` | `O(log n)` | `O(log n)`          | yes       |
| `HashMap`           | `O(1)*`    | `O(1)*`    | `O(1)*`             | no        |
| `Binary heap`       | `O(n)`     | `O(log n)` | `O(log n) min only` | min only  |
| `Trie`              | `O(m)`     | `O(m)`     | `O(m)`              | by prefix |
| `B-tree`            | `O(log n)` | `O(log n)` | `O(log n)`          | yes       |

- average with a decent `hashCode()`; m = key length. Traversal is O(n) everywhere — you're visiting every node.

### Choosing

| I want to…                            | Use                 |
| ------------------------------------- | ------------------- |
| Lookup by exact key, order irrelevant | `HashMap`           |
| Iterate in sorted order               | `TreeMap`           |
| Nearest key above or below            | `NavigableMap`      |
| Everything between two keys           | `subMap`            |
| Only ever the smallest item           | `PriorityQueue`     |
| Autocomplete on a prefix              | `trie`              |
| Sorted data on disk                   | `B+ tree (your DB)` |

## Ways to get hurt

### Validating a BST by checking only the children

Comparing each node against its immediate children passes trees that are badly wrong — a node deep in the left subtree can exceed the root while still being smaller than its parent. Pass a permitted range down the recursion instead.

### Building your own BST from sorted input

It degenerates into a linked list and every operation becomes O(n). Either use `TreeMap`, which rebalances, or shuffle before inserting. The failure is silent: correct results, catastrophic performance, and only at scale.

### A comparator that isn't total

Comparing on one field means every entry sharing that field collapses into one. A `TreeSet` ordered by last name holds exactly one Smith. Always `thenComparing` down to something unique.

### Expecting `PriorityQueue` to be sorted

It's heap-ordered, not sorted — only `peek` and `poll` respect the ordering, and the iterator doesn't. Covered in detail in [Inside Stacks and Queues](04-inside-stacks-and-queues.html).

### Mutating a key that's already in a tree

Its position was fixed by comparisons at insert time. Change what the comparator reads and the entry becomes unreachable — present in `size()`, invisible to `get()`.

### Recursing over an untrusted tree

Depth equals height. A tree built from adversarial or sorted input can be a million deep, and `StackOverflowError` is not something you can meaningfully catch. Iterate with an explicit `Deque` when the shape isn't yours to guarantee.

> **The short version**
>
> A tree buys you order, and balance is what keeps that affordable. In-order traversal of a BST is sorted output — that one fact explains `TreeMap`. Reach for `HashMap` when you only need lookup, `TreeMap` the moment you need "nearest", "between", or "in order", and remember that anything you build yourself has no balancing unless you write it.
