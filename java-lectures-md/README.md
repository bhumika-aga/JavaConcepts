# Java Lectures

Nine references on how Java actually works underneath, sharing one structure:
a lead-in, mechanism diagrams, real code, comparison tables, and a closing
"ways to get hurt" section.

| #   | Lecture                                                    | Covers                                                                                                                   |
| --- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| 01  | [Inside Java Maps](01-inside-java-maps.md)                 | Hashing, collision chains, treeify, resize — then every implementation from `LinkedHashMap` to `WeakHashMap`             |
| 02  | [Inside Java Generics](02-inside-java-generics.md)         | Type parameters, bounds, invariance, wildcards and PECS, and what type erasure leaves behind                             |
| 03  | [Inside Java Threads](03-inside-java-threads.md)           | The memory model and happens-before, locks, `java.util.concurrent`, executors, virtual threads                           |
| 04  | [Inside Stacks and Queues](04-inside-stacks-and-queues.md) | The two memory regions and three container shapes that share their names, plus the binary heap                           |
| 05  | [Java Concurrency Primer](05-java-concurrency-primer.md)   | The short form of 03 — one fact, three problems, five tools, seven rules                                                 |
| 06  | [Inside Kafka](06-inside-kafka.md)                         | The log vs the queue, partitions and ordering, acks and durability, consumer groups, offsets, rebalancing, exactly-once  |
| 07  | [Inside Spring](07-inside-spring.md)                       | Inversion of control, the bean lifecycle, proxies and AOP, `@Transactional`'s defaults, MVC and data access              |
| 08  | [Inside Spring Boot](08-inside-spring-boot.md)             | Auto-configuration internals, starters, the property precedence ladder, the fat jar, Actuator, AOT and native            |
| 09  | [Inside Trees](09-inside-trees.md)                         | Binary search trees, the four traversals, balance and rotations, red-black rules, TreeMap and TreeSet, tries and B-trees |

## Diagrams

All 53 diagrams live in [`diagrams/`](diagrams/) as standalone SVG. Each carries
its own palette and a `prefers-color-scheme` block, so they render correctly on
light and dark backgrounds without any external stylesheet.

## Publishing these elsewhere

The Markdown is plain CommonMark plus pipe tables — it pastes into GitHub,
Hashnode, Dev.to and Obsidian as-is. Two caveats:

- **Substack and Medium** don't accept SVG. Convert the diagrams to PNG first
  (`rsvg-convert -w 1400 diagrams/x.svg -o x.png`) and upload them as images.
- **Relative image paths** assume the `diagrams/` folder travels with the
  Markdown. On platforms that host images separately, rewrite the paths.
