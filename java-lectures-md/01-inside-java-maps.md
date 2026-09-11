# Inside Java Maps

_java.util.Map · explained from zero_

> Every Map is a machine for answering one question fast: _where did I put that?_ Eleven implementations, eleven different machines. Here is what each one actually does under the lid.

---

## Contents

1. [What a Map is](#what-a-map-is)
2. [The one trick behind hashing](#the-one-trick-behind-hashing)
3. [HashMap](#hashmap)
4. [LinkedHashMap](#linkedhashmap)
5. [Hashtable](#hashtable)
6. [TreeMap](#treemap)
7. [ConcurrentHashMap](#concurrenthashmap)
8. [ConcurrentSkipListMap](#concurrentskiplistmap)
9. [EnumMap](#enummap)
10. [IdentityHashMap](#identityhashmap)
11. [WeakHashMap](#weakhashmap)
12. [Map.of and the immutable maps](#mapof-and-the-immutable-maps)
13. [Compare them all](#compare-them-all)
14. [Which do I pick?](#which-do-i-pick)
15. [Ways to get hurt](#ways-to-get-hurt)

---

## What a Map is

Imagine a wall of cubbies. Each cubby has a name tag on the front and something inside. That's it. That's a Map.

![Four cubbies, each with a name tag on top and a value inside](diagrams/inside-java-maps-01.svg)

**A Map is a wall of cubbies.** The name tag is the key. What's inside is the value. Give it a tag, get the contents back.

The whole interface is small. You put things in, you get things out, and you ask whether something is there.

```java
            Map<String, Integer> ages = new HashMap<>();

ages.put("alice", 31);        // hang a tag, fill the cubby
ages.get("alice");            // → 31
ages.get("nobody");           // → null (no such tag)
ages.containsKey("bob");     // → false
ages.remove("alice");         // → 31, and the cubby empties
ages.size();                    // → 0

```

Two rules hold for _every_ Map in Java, no matter which one you pick:

- **Keys are unique.** Putting `"alice"` twice doesn't make two cubbies. The second put overwrites the first and hands you back the old value.
- **Values are not.** Ten different keys can all hold the number `31`. Nobody minds.

`Map` itself is just an **interface** — a promise about what the methods do, with no machinery behind it. The machinery is the implementation, and choosing one is the entire subject of this page. They differ in how they find a cubby, whether the cubbies stay in any order, and what happens when two threads reach for the wall at once.

## The one trick behind hashing

Searching a thousand cubbies one at a time is slow. So most Maps don't search at all — they _calculate_ which cubby to open.

Every object in Java can produce a number about itself, via `hashCode()`. Equal objects must produce equal numbers. That number is a fingerprint — big, arbitrary, and useless as an address on its own, because you don't have four billion cubbies. So you fold it down to fit.

![A key is hashed, spread, masked, and turned into a bucket index](diagrams/inside-java-maps-02.svg)

**Address arithmetic, not searching.** The key computes its own cubby number. That is why a `HashMap` lookup costs the same whether it holds ten entries or ten million.

### Why the extra stirring step?

The final `& (n - 1)` is a cheap replacement for a modulo, and it works only because table sizes are always powers of two. But it has a flaw: it throws away every bit above the low few. If a set of keys differs only in its _high_ bits, they'd all land in the same cubby.

![The high 16 bits of a hash are XOR-ed into the low 16 bits before masking](diagrams/inside-java-maps-03.svg)

**One XOR buys a lot of safety.** `h ^ (h >>> 16)` mixes the discarded high half into the half that survives, so keys that differ only up top still scatter. It costs a single instruction.

> **The contract you must not break**
>
> If `a.equals(b)`, then `a.hashCode() == b.hashCode()` — always. Break it and your key vanishes: you store with one fingerprint, look up with another, and the Map opens an empty cubby. This is the single most common Map bug in Java, and it is completely silent.

## HashMap

The default. An array of buckets, a chain in each bucket, and a red-black tree if a chain gets embarrassing.

| Property              | Value                  |
| --------------------- | ---------------------- |
| **Backing structure** | Array + chains + trees |
| **Iteration order**   | None guaranteed        |
| **null key**          | One allowed            |
| **null values**       | Allowed                |
| **Thread-safe**       | No                     |
| **get() cost**        | `O(1) average`         |

### The shape of it

Inside is one array, `Node<K,V>[] table`. Each slot is a **bucket**. Each bucket holds a chain of nodes, and each node carries four things: the cached hash, the key, the value, and a pointer to the next node in the chain.

![A bucket array of eight slots, with a three-node collision chain hanging off slot 5](diagrams/inside-java-maps-04.svg)

**Collisions are normal, not exceptional.** Different keys legitimately land in the same bucket. The Map then walks the short chain, comparing cached hashes first (cheap) and only calling `equals()` on a hash match.

### When a chain gets too long, it becomes a tree

A long chain destroys the O(1) promise — walking 500 nodes is just a slow linked list. So since Java 8, `HashMap` watches its chains. When one reaches **8 nodes** _and_ the table is at least **64** slots long, that bucket converts into a **red-black tree**, and the worst case improves from O(n) to O(log n).

![A chain of eight nodes converting into a balanced red-black tree](diagrams/inside-java-maps-05.svg)

**Treeify at 8, untreeify at 6.** The gap between the two thresholds is deliberate hysteresis — without it, a bucket hovering at the boundary would convert back and forth on every insert and remove.

> **The table ≥ 64 condition matters**
>
> If a bucket hits 8 nodes while the table is still small, `HashMap` **resizes instead of treeifying**. A crowded small table is a spreading problem, and doubling the array is the cheaper fix. Trees are reserved for genuinely adversarial hash distributions.

### Growing: load factor and the doubling trick

A `HashMap` starts with capacity **16** and a load factor of **0.75**, giving a threshold of 12. Insert the 13th entry and the table doubles to 32. (The array itself is allocated lazily, on the first `put` — a `new HashMap<>()` that you never write to costs almost nothing.)

Resizing sounds expensive, and here is the elegant part: **nothing is re-hashed**. Because capacity is always a power of two, doubling adds exactly one bit to the mask. Every entry in old bucket _j_ either stays at _j_ or moves to _j + oldCapacity_ — decided by a single bit test, `(hash & oldCap) == 0`. Each chain splits cleanly into a "low" list and a "high" list in one pass.

![A bucket chain splitting into a low list and a high list when the table doubles](diagrams/inside-java-maps-06.svg)

**Doubling is a bit test, not a re-hash.** This is the reason `HashMap` insists on power-of-two capacities — pass `new HashMap<>(1000)` and it quietly rounds up to 1024.

### Seeing it work

```java
            // Two keys engineered to collide: "Aa" and "BB" have the SAME hashCode.
System.out.println("Aa".hashCode());   // 2112
System.out.println("BB".hashCode());   // 2112

Map<String, String> m = new HashMap<>();
m.put("Aa", "first");
m.put("BB", "second");

// Same bucket, but BOTH survive — equals() tells them apart.
System.out.println(m.size());        // 2
System.out.println(m.get("Aa"));   // "first"

// Sizing up front avoids repeated doubling.
// Want room for 1000 without a resize? 1000 / 0.75 = 1334.
Map<String, String> sized = new HashMap<>(1334);

// Java 19+ does the arithmetic for you:
Map<String, String> better = HashMap.newHashMap(1000);

```

The methods worth knowing, because they turn three lines into one and do it atomically-ish in a single bucket visit:

```java
            Map<String, List<String>> index = new HashMap<>();

// The grouping idiom — creates the list only if absent.
index.computeIfAbsent("fruit", k -> new ArrayList<>()).add("apple");
index.computeIfAbsent("fruit", k -> new ArrayList<>()).add("pear");
// → {fruit=[apple, pear]}

Map<String, Integer> counts = new HashMap<>();
for (String word : words) {
    counts.merge(word, 1, Integer::sum);   // the counting idiom
}

counts.getOrDefault("missing", 0);          // no null check needed
counts.putIfAbsent("seed", 0);              // only if not there

```

## LinkedHashMap

A HashMap that also remembers what order things arrived in — by threading a second, independent list through the same nodes.

| Property              | Value                 |
| --------------------- | --------------------- |
| **Backing structure** | HashMap + linked list |
| **Iteration order**   | Insertion, or access  |
| **null key**          | One allowed           |
| **null values**       | Allowed               |
| **Thread-safe**       | No                    |
| **get() cost**        | `O(1) average`        |

It literally `extends HashMap`. All the bucket machinery above is inherited unchanged. The only addition is two extra pointers per node — `before` and `after` — plus a `head` and `tail` on the map itself. The buckets decide _where things live_; the linked list decides _what order you see them in_. The two are completely independent.

![Bucket array with a separate doubly-linked list threading through the entries in insertion order](diagrams/inside-java-maps-07.svg)

**Two structures, one set of nodes.** A plain `HashMap` iterating a mostly-empty table has to walk every slot. `LinkedHashMap` just follows its own list, so iteration cost tracks the number of entries, not the capacity.

### Access order: the free LRU cache

Flip one constructor flag and the list reorders itself on every `get`, moving the touched entry to the end. Combine that with an override of `removeEldestEntry` and you have a bounded least-recently-used cache in about six lines.

```java
            // Insertion order (default) — predictable iteration.
Map<String, Integer> ordered = new LinkedHashMap<>();
ordered.put("c", 3); ordered.put("a", 1); ordered.put("b", 2);
System.out.println(ordered);        // {c=3, a=1, b=2} — always

// An LRU cache holding at most 3 entries.
// args: initialCapacity, loadFactor, accessOrder=true
Map<String, Integer> lru = new LinkedHashMap<>(16, 0.75f, true) {
    @Override protected boolean removeEldestEntry(Map.Entry<String, Integer> eldest) {
        return size() > 3;      // evict once we exceed 3
    }
};

lru.put("a", 1); lru.put("b", 2); lru.put("c", 3);
lru.get("a");                         // "a" jumps to the most-recent end
lru.put("d", 4);                      // over the limit → evict eldest, which is "b"
System.out.println(lru);            // {c=3, a=1, d=4}

```

> **Watch out**
>
> In access-order mode, `get()` is a **structural modification** — it mutates the list. Two threads calling nothing but `get()` can still corrupt this map, and iterating while calling `get()` throws `ConcurrentModificationException` on a single thread.

## Hashtable

The 1996 original. Still in the JDK, still compiles, and you should not use it.

| Property              | Value           |
| --------------------- | --------------- |
| **Backing structure** | Array + chains  |
| **Iteration order**   | None guaranteed |
| **null key**          | Throws NPE      |
| **null values**       | Throws NPE      |
| **Thread-safe**       | Yes — badly     |
| **get() cost**        | `O(1) average`  |

It predates the collections framework entirely, and it shows. Three differences from `HashMap` tell the whole story:

- **Every method is `synchronized`** on the whole table. Two threads reading unrelated keys still queue behind each other. This is the reason it's obsolete — the safety is real but the granularity is hopeless.
- **Capacity is 11 and grows to `2n + 1`**, not a power of two. So the index is a real modulo — `(hash & 0x7FFFFFFF) % table.length` — where the mask exists only to force the value positive. Integer division per lookup, versus `HashMap`'s single AND.
- **No nulls anywhere.** Neither keys nor values.

It never got the tree optimization either — a degenerate bucket stays a linked list forever.

```java
            // Don't. But if you inherit it, this is what it does:
Hashtable<String, Integer> t = new Hashtable<>();
t.put("a", 1);
t.put(null, 1);      // NullPointerException
t.put("b", null);    // NullPointerException

// Need a thread-safe map? Use this instead:
Map<String, Integer> good = new ConcurrentHashMap<>();

```

`Properties` extends `Hashtable`, which is why that class also behaves oddly. That's the main reason the JDK can never quite delete it.

## TreeMap

No hashing at all. Keys live in a self-balancing binary search tree, permanently sorted, and that unlocks a whole category of questions the hash maps simply cannot answer.

| Property              | Value             |
| --------------------- | ----------------- |
| **Backing structure** | Red-black tree    |
| **Iteration order**   | Sorted by key     |
| **null key**          | NPE by default    |
| **null values**       | Allowed           |
| **Thread-safe**       | No                |
| **get() cost**        | `O(log n) always` |

Each node holds a key, a value, pointers to left child, right child, and parent, plus one bit of colour. Everything smaller than a node sits in its left subtree; everything larger sits to the right. Lookup is descent: compare, go left or right, repeat.

![A red-black tree of names with the search path to a key highlighted](diagrams/inside-java-maps-08.svg)

**Sorted by construction.** Iterating a `TreeMap` is an in-order walk, so entries come out in key order for free. The price is O(log n) on every single operation, including `get` — there is no O(1) path.

### What sorting buys you

This is the real reason to reach for `TreeMap`. Because keys are ordered, you can ask _relative_ questions — nearest, next, everything between — that a hash map has no way to answer short of scanning everything.

```java
            TreeMap<Integer, String> grades = new TreeMap<>();
grades.put(60, "D"); grades.put(70, "C");
grades.put(80, "B"); grades.put(90, "A");

// Nearest-match lookups — the killer feature.
grades.floorEntry(85);      // 80=B  (greatest key ≤ 85)
grades.ceilingEntry(85);    // 90=A  (least key ≥ 85)
grades.lowerKey(80);        // 70    (strictly less)
grades.higherKey(80);       // 90    (strictly greater)

// Ends and ranges.
grades.firstKey();            // 60
grades.lastEntry();           // 90=A
grades.headMap(80);         // {60=D, 70=C}      — a live view, not a copy
grades.subMap(70, true, 85, true);  // {70=C, 80=B}
grades.descendingMap();       // {90=A, 80=B, 70=C, 60=D}

// Queue-like removal from either end.
grades.pollFirstEntry();      // removes and returns 60=D

// Custom ordering — the comparator replaces natural order entirely.
TreeMap<String, Integer> ci =
    new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
ci.put("Apple", 1);
ci.get("APPLE");              // 1 — comparator decides equality, not equals()

```

> **Ordering defines identity here**
>
> `TreeMap` never calls `equals()` or `hashCode()`. Two keys are "the same key" precisely when `compareTo` (or your comparator) returns `0`. A comparator inconsistent with `equals` gives you a Map that misbehaves in ways no debugger will make obvious — as the case-insensitive example above shows, where `"APPLE"` finds an entry stored under `"Apple"`.

The null rule follows from the same fact: sorting a key means comparing it, and comparing `null` throws. So `put(null, v)` is an NPE unless you supplied a comparator that tolerates nulls.

## ConcurrentHashMap

The one to reach for when threads are involved. It doesn't lock the map — it locks a single bucket, and often not even that.

| Property              | Value                  |
| --------------------- | ---------------------- |
| **Backing structure** | Array + chains + trees |
| **Iteration order**   | None guaranteed        |
| **null key**          | Throws NPE             |
| **null values**       | Throws NPE             |
| **Thread-safe**       | Yes — per bucket       |
| **get() cost**        | `O(1), lock-free`      |

Structurally it's a `HashMap`: same bucket array, same chains, same treeify-at-8. The difference is entirely in how writes are coordinated.

- **Reads take no lock at all.** The table and the node `next`/`value` fields are `volatile`, so `get()` is a plain read that sees a consistent, recent value. Readers never block, and never block writers.
- **Writing into an empty bucket takes no lock either.** It's a single CAS — compare-and-swap — to install the first node. If the CAS loses a race, the thread simply retries.
- **Writing into an occupied bucket locks that bucket only**, via `synchronized` on the node already at the head. Threads working on any of the other buckets are unaffected.

![Four threads writing to four different buckets, only one of which needs a lock](diagrams/inside-java-maps-09.svg)

**Lock striping taken to its limit.** Java 7 divided the table into 16 fixed `Segment` locks. Java 8 threw that away — the lock granularity is now a single bin, so it gets finer as the map grows.

### Why null is banned

This looks like an arbitrary restriction until you think about a concurrent `get`. In a `HashMap`, a `null` return is ambiguous — either the key is absent, or it's present with a `null` value — and you resolve it by calling `containsKey`. In a concurrent map that second call is worthless: another thread may have changed the answer in between, and there's no way to make the pair atomic. Banning null values makes `null` mean exactly one thing: _not there_.

```java
            ConcurrentMap<String, Integer> counts = new ConcurrentHashMap<>();

// These are atomic — the whole read-modify-write holds the bin lock.
counts.merge("hits", 1, Integer::sum);       // safe concurrent counter
counts.computeIfAbsent("key", k -> expensive(k));  // computed at most once
counts.putIfAbsent("seed", 0);
counts.replace("hits", 10, 11);                  // CAS: only if currently 10

// This is NOT atomic — two threads can both read 5 and both write 6.
counts.put("hits", counts.get("hits") + 1);      // ✗ lost update

// Parallel bulk operations. The threshold is the number of elements
// below which it stays single-threaded.
counts.forEach(16, (k, v) -> System.out.println(k + "=" + v));
Integer total = counts.reduceValues(16, Integer::sum);
String found = counts.search(16, (k, v) -> v > 100 ? k : null);

```

> ⚠️ **Two sharp edges**
>
> **`size()` is an estimate.** The count is striped across cells to avoid a single contended counter, so `size()` sums them without locking. In a quiet moment it's exact; under concurrent writes it's approximate. Never use it in a correctness check.
>
> **Don't do slow or reentrant work inside `computeIfAbsent`.** The bin lock is held for the duration of your lambda. Blocking I/O in there stalls every other writer to that bin, and touching the same map from inside the lambda can deadlock outright.

Iterators are **weakly consistent**: they never throw `ConcurrentModificationException`, they reflect the map at some point at or after creation, and they may or may not show changes made during the walk. That's the deliberate trade for never blocking.

## ConcurrentSkipListMap

What you use when you need _both_ sorted keys and thread safety. It's a TreeMap's capability with a completely different machine underneath.

| Property              | Value               |
| --------------------- | ------------------- |
| **Backing structure** | Skip list           |
| **Iteration order**   | Sorted by key       |
| **null key**          | Throws NPE          |
| **null values**       | Throws NPE          |
| **Thread-safe**       | Yes — lock-free     |
| **get() cost**        | `O(log n) expected` |

Balanced trees are miserable to make concurrent — a rebalance rewrites pointers all over the structure, so you end up locking large regions. A **skip list** sidesteps the problem: it's a sorted linked list with randomly-built express lanes above it. Every insert flips coins to decide how many lanes its node joins. No rebalancing ever happens, so every update is a local pointer change that CAS can handle without locks.

![A three-level skip list with express lanes, showing the search path for a key](diagrams/inside-java-maps-10.svg)

**Randomness instead of rebalancing.** Skip lists trade a guaranteed bound for a probabilistic one, and get lock-free concurrent updates in exchange. `size()` here is O(n) — it walks the bottom lane — so avoid it in loops.

```java
            ConcurrentNavigableMap<Long, String> events = new ConcurrentSkipListMap<>();
events.put(System.currentTimeMillis(), "started");

// Full NavigableMap vocabulary, safe across threads.
events.headMap(cutoff).clear();          // expire everything older
events.firstEntry();                     // oldest surviving event
events.pollFirstEntry();                 // atomic take-from-front

```

## EnumMap

When your keys are an enum, hashing is wasted work. The compiler already numbered them for you.

| Property              | Value                  |
| --------------------- | ---------------------- |
| **Backing structure** | Plain array            |
| **Iteration order**   | Enum declaration order |
| **null key**          | Throws NPE             |
| **null values**       | Allowed                |
| **Thread-safe**       | No                     |
| **get() cost**        | `O(1) — a real one`    |

Every enum constant has an `ordinal()` — its position in the declaration, starting at zero. `EnumMap` allocates one array slot per constant and indexes it directly. No hashing, no collisions, no chains, no load factor, no resizing. Ever. It's the fastest Map in the JDK and also the smallest.

![Enum constants mapping directly onto array slots by ordinal](diagrams/inside-java-maps-11.svg)

**Array indexing, dressed as a Map.** Because slots are ordered by ordinal, iteration comes out in declaration order automatically — often exactly the order you wanted.

```java
            enum Day { MON, TUE, WED, THU, FRI }

// Needs the Class object — that's how it knows how big the array must be.
EnumMap<Day, String> plan = new EnumMap<>(Day.class);
plan.put(Day.WED, "swim");
plan.put(Day.MON, "gym");

// Always declaration order, regardless of insertion order.
System.out.println(plan);     // {MON=gym, WED=swim}

```

The rule of thumb is simple: if the key type is an enum, use `EnumMap`. There is no case where `HashMap` beats it.

## IdentityHashMap

Deliberately breaks the Map contract: it compares keys with `==` instead of `equals()`. Rarely what you want, occasionally exactly what you need.

| Property              | Value                 |
| --------------------- | --------------------- |
| **Backing structure** | Array, linear probing |
| **Key comparison**    | Reference `==`        |
| **null key**          | Allowed               |
| **null values**       | Allowed               |
| **Thread-safe**       | No                    |
| **get() cost**        | `O(1) average`        |

Two differences from everything above. It hashes with `System.identityHashCode()` — the object's own identity, not its contents — and it resolves collisions by **linear probing** rather than chaining: on a clash it just walks forward to the next free pair of slots. Keys and values interleave in a single flat array, keys at even indices, values at odd.

![A flat array with keys at even indices and values at odd, showing a linear probe to the next free slot](diagrams/inside-java-maps-12.svg)

**Open addressing, not chaining.** Everything lives in one contiguous array, which is cache-friendly but means a full table degrades sharply — it resizes well before that point.

```java
            String a = new String("hello");
String b = new String("hello");   // a.equals(b) but a != b

Map<String, Integer> normal = new HashMap<>();
normal.put(a, 1); normal.put(b, 2);
System.out.println(normal.size());     // 1 — equal keys collapse

Map<String, Integer> ident = new IdentityHashMap<>();
ident.put(a, 1); ident.put(b, 2);
System.out.println(ident.size());      // 2 — distinct objects, distinct keys

```

The legitimate uses are narrow and real: graph or object-graph traversal where you must track "have I visited _this exact object_", serialization frameworks preserving reference topology, and attaching metadata to objects whose `equals()` is expensive or lies. Outside those, reaching for it is almost always a mistake.

## WeakHashMap

A Map whose entries delete themselves. It holds its keys weakly, so an entry lives exactly as long as someone else is still using the key.

| Property              | Value                 |
| --------------------- | --------------------- |
| **Backing structure** | Array + chains        |
| **Key references**    | Weak — GC-collectable |
| **null key**          | Allowed               |
| **null values**       | Allowed               |
| **Thread-safe**       | No                    |
| **get() cost**        | `O(1) average`        |

Normally, putting an object into a Map keeps it alive — the Map's reference is enough to stop the garbage collector. `WeakHashMap` wraps each key in a `WeakReference`, which the GC is free to ignore. Once nothing else in your program refers to a key, the collector reclaims it and the entry silently disappears from the map.

![An entry surviving while a strong reference exists, then vanishing after that reference is dropped and GC runs](diagrams/inside-java-maps-13.svg)

**Cleanup is not instant.** Cleared keys land on a `ReferenceQueue`, and the map drains it during ordinary operations. Entries vanish at some unpredictable point after collection — never treat the timing as deterministic.

> ⚠️ **The classic leak**
>
> **Values are held strongly.** If a value refers back to its own key — directly, or through a field — the value keeps the key alive, the key keeps the entry alive, and nothing is ever collected. You have built a memory leak out of the tool you chose to prevent one. Wrap such values in a `WeakReference`, or use a different design.

```java
            Map<Object, String> meta = new WeakHashMap<>();
Object key = new Object();
meta.put(key, "some metadata");
System.out.println(meta.size());   // 1

key = null;                          // last strong reference dropped
System.gc();                        // a hint, not a command
// eventually → meta.size() == 0, entry expunged on a later operation

```

It's the right tool for associating extra data with objects you don't own the lifecycle of — a cache keyed by `Class` objects, listener registries that shouldn't pin their subjects, canonicalizing maps.

## Map.of and the immutable maps

Small, fixed, unchangeable — and deliberately unpredictable in iteration order, to stop you depending on it.

| Property              | Value                  |
| --------------------- | ---------------------- |
| **Backing structure** | Probed flat array      |
| **Iteration order**   | Randomized per JVM run |
| **null key or value** | Throws NPE             |
| **Duplicate keys**    | Throws IAE             |
| **Thread-safe**       | Yes — immutable        |
| **Mutation**          | UnsupportedOperation   |

Since Java 9, `Map.of()` builds a compact immutable map with no buckets and no chains — just a flat probe table sized about twice the entry count. Up to ten pairs inline; beyond that use `Map.ofEntries`.

```java
            Map<String, Integer> fixed = Map.of("a", 1, "b", 2, "c", 3);
fixed.put("d", 4);       // UnsupportedOperationException

// More than 10 pairs:
Map<String, Integer> bigger = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2)
);

// Defensive copy — also immutable.
Map<String, Integer> snapshot = Map.copyOf(mutableMap);

// The old way: a VIEW, not a copy. Changes to the
// original still show through this wrapper.
Map<String, Integer> view = Collections.unmodifiableMap(mutableMap);

```

> **The randomized order is a feature**
>
> Iteration order of `Map.of()` is salted differently on each JVM start, so the same code produces a different order between runs. That's intentional: it makes accidental order-dependence fail loudly during testing instead of silently in production. Never write code that relies on it.

## Compare them all

### Structure and cost

| Implementation          | Under the hood                   | `get`      | `put`      | Iteration order     |
| ----------------------- | -------------------------------- | ---------- | ---------- | ------------------- |
| `HashMap`               | Array + chains + red-black trees | `O(1)*`    | `O(1)*`    | Unspecified         |
| `LinkedHashMap`         | HashMap + doubly-linked list     | `O(1)*`    | `O(1)*`    | Insertion or access |
| `Hashtable`             | Array + chains, one global lock  | `O(1)*`    | `O(1)*`    | Unspecified         |
| `TreeMap`               | Red-black tree                   | `O(log n)` | `O(log n)` | Sorted by key       |
| `ConcurrentHashMap`     | Array + per-bin locks + CAS      | `O(1)*`    | `O(1)*`    | Unspecified         |
| `ConcurrentSkipListMap` | Lock-free skip list              | `O(log n)` | `O(log n)` | Sorted by key       |
| `EnumMap`               | Plain array indexed by ordinal   | `O(1)`     | `O(1)`     | Enum declaration    |
| `IdentityHashMap`       | Flat array, linear probing       | `O(1)*`    | `O(1)*`    | Unspecified         |
| `WeakHashMap`           | Array + chains, weak keys        | `O(1)*`    | `O(1)*`    | Unspecified         |
| `Map.of(…)`             | Immutable probe table            | `O(1)*`    | `—`        | Randomized          |

- average case with a decent `hashCode()`. `HashMap` and `ConcurrentHashMap` degrade to O(log n), not O(n), once a bucket treeifies.

#### Nulls and thread safety

| Implementation          | null key | null value | Thread-safe | Notes                                         |
| ----------------------- | -------- | ---------- | ----------- | --------------------------------------------- |
| `HashMap`               | one      | yes        | no          | null key always lands in bucket 0             |
| `LinkedHashMap`         | one      | yes        | no          | access-order mode makes `get` a mutation      |
| `Hashtable`             | no       | no         | coarse      | every method synchronized on the whole map    |
| `TreeMap`               | no       | yes        | no          | unless a comparator tolerates null            |
| `ConcurrentHashMap`     | no       | no         | yes         | null must mean "absent", unambiguously        |
| `ConcurrentSkipListMap` | no       | no         | yes         | `size()` is O(n) — it walks the list          |
| `EnumMap`               | no       | yes        | no          | uses an internal sentinel for null values     |
| `IdentityHashMap`       | yes      | yes        | no          | compares with `==`, breaking the Map contract |
| `WeakHashMap`           | yes      | yes        | no          | entries disappear without you removing them   |
| `Map.of(…)`             | no       | no         | yes         | immutable, so trivially safe                  |

## Which do I pick?

Ninety percent of the time the answer is `HashMap`. Here is the rest.

| I want to…                                      | Use                     |
| ----------------------------------------------- | ----------------------- |
| I just need a map                               | `HashMap`               |
| Threads will touch it                           | `ConcurrentHashMap`     |
| I need keys sorted, or "nearest key" queries    | `TreeMap`               |
| Sorted _and_ concurrent                         | `ConcurrentSkipListMap` |
| Iteration must follow insertion order           | `LinkedHashMap`         |
| I want a bounded LRU cache                      | `LinkedHashMap`         |
| My keys are an enum                             | `EnumMap`               |
| Entries should vanish when keys are unreachable | `WeakHashMap`           |
| I need reference identity, not equality         | `IdentityHashMap`       |
| A small constant lookup table                   | `Map.of(…)`             |
| I'm maintaining 1998 code                       | `Hashtable`             |

## Ways to get hurt

### Mutating a key after you've stored it

The Map filed the entry under the key's hash _at insertion time_. Change a field that `hashCode()` reads, and the key now computes a different bucket. The entry is still in the table, holding memory, but unreachable through the front door.

```java
            List<String> key = new ArrayList<>(List.of("a"));
Map<List<String>, String> m = new HashMap<>();
m.put(key, "stored");

key.add("b");                  // hashCode just changed

m.get(key);                    // null — wrong bucket now
m.containsKey(key);            // false
m.size();                      // 1 — it's still in there, just lost

```

Use immutable keys. `String`, boxed primitives, enums, and records are all safe by construction.

### Sharing a HashMap across threads

An unsynchronized `HashMap` under concurrent writes doesn't just lose updates — it can corrupt its own structure. On Java 7 a concurrent resize could weave a chain into a cycle, and a later `get()` would spin forever at 100% CPU. Java 8's resize is no longer prone to that particular loop, but entries still vanish and internal state still tears. It's undefined behaviour; don't reason about which symptom you'll get.

```java
            // ✗ Undefined behaviour under concurrent writes
Map<String, Integer> bad = new HashMap<>();

// ✓ Correct, and fast
Map<String, Integer> good = new ConcurrentHashMap<>();

// ~ Correct but coarse: one lock for the whole map,
// and compound operations STILL need external sync.
Map<String, Integer> meh = Collections.synchronizedMap(new HashMap<>());
synchronized (meh) {                  // required — iteration is not covered
    for (var e : meh.entrySet()) { /* … */ }
}

```

### Assuming iteration order

`HashMap` order is not insertion order, not sorted order, and not stable — it changes when the table resizes, and it differs across Java versions. Code that passes today because the order happened to be convenient will break on a resize you didn't predict. If order matters, say so in the type: `LinkedHashMap` or `TreeMap`.

### Modifying a map while iterating it

```java
            // ✗ ConcurrentModificationException
for (String k : map.keySet()) {
    if (k.startsWith("tmp")) map.remove(k);
}

// ✓ Removal through the iterator is legal
Iterator<String> it = map.keySet().iterator();
while (it.hasNext()) {
    if (it.next().startsWith("tmp")) it.remove();
}

// ✓ Or just say what you mean
map.keySet().removeIf(k -> k.startsWith("tmp"));

```

### Forgetting that computeIfAbsent rejects null returns

If your mapping function returns `null`, no entry is recorded — the call returns `null` and the map is untouched. That's usually what you want, but it means `computeIfAbsent` can be called repeatedly for the same key without ever caching anything, which looks like a mysterious performance problem rather than a bug.

> **One habit worth keeping**
>
> Declare variables as `Map`, not the implementation — `Map<String, Integer> m = new HashMap<>()`. Swapping `HashMap` for `ConcurrentHashMap` six months from now then costs one line instead of a refactor. The exception is when the implementation's extra methods are the point: a `TreeMap` declared as `Map` loses `floorKey` and every other navigation method, so declare that one as `NavigableMap`.
