# Java Lectures

Nine self-contained references on how Java actually works underneath the API —
the mechanisms, not the syntax. Each one follows the same shape: a lead-in,
mechanism diagrams, real code, comparison tables, and a closing
"ways to get hurt" section.

Open **[`index.html`](index.html)** to browse them, or open any lecture directly.

| #   | Lecture                                                      | Covers                                                                                                                                      | Sections | Diagrams |
| --- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------- |
| 01  | [Inside Java Maps](01-inside-java-maps.html)                 | Hashing, collision chains, treeify at 8, the power-of-two resize trick — then every implementation from `LinkedHashMap` to `WeakHashMap`    | 15       | 13       |
| 02  | [Inside Java Generics](02-inside-java-generics.html)         | Type parameters, bounds, invariance, wildcards and PECS, wildcard capture, and exactly what type erasure leaves behind                      | 14       | 6        |
| 03  | [Inside Java Threads](03-inside-java-threads.html)           | The memory model and happens-before, locks and deadlock, `java.util.concurrent`, executors, `CompletableFuture`, virtual threads            | 12       | 8        |
| 04  | [Inside Stacks and Queues](04-inside-stacks-and-queues.html) | The two memory regions and three container shapes that share their names, plus the binary heap inside `PriorityQueue`                       | 12       | 5        |
| 05  | [Java Concurrency Primer](05-java-concurrency-primer.html)   | The short form of 03 — one fact, three problems, five tools, seven rules                                                                    | 6        | 1        |
| 06  | [Inside Kafka](06-inside-kafka.html)                         | Why it's a log and not a queue, partitions and ordering, acks and durability, consumer groups, offsets, rebalancing, exactly-once           | 15       | 5        |
| 07  | [Inside Spring](07-inside-spring.html)                       | Inversion of control, the bean lifecycle, why proxy-based annotations silently do nothing, `@Transactional`'s defaults, MVC and data access | 14       | 5        |
| 08  | [Inside Spring Boot](08-inside-spring-boot.html)             | Auto-configuration internals, starters, the property precedence ladder, the startup sequence, the fat jar, Actuator, AOT and native         | 14       | 4        |
| 09  | [Inside Trees](09-inside-trees.html)                         | Binary search trees, the four traversals, balance and rotations, red-black rules, `TreeMap` and `TreeSet`, tries and B-trees                | 14       | 6        |

116 sections and 53 diagrams in total.

## How they're built

Every lecture is a **single HTML file with no build step and no JavaScript.**
Open one from disk and it works.

- **One shared stylesheet.** All nine carry a byte-identical `<style>` block —
  the same tokens, type scale and components (`.spec` strips, `.note` callouts,
  `.pick` decision lists, tables). `index.html` carries that same sheet plus its
  own card styles.
- **Diagrams are inline SVG**, hand-authored against the stylesheet's CSS
  variables. Nothing is rasterised, so they stay sharp at any zoom and recolour
  themselves with the theme.
- **Light and dark.** Colours are defined as tokens in three states — a bare
  `:root`, a `prefers-color-scheme` block, and an explicit `[data-theme]`
  override — so the pages follow your system setting.
- **Fonts** come from Google Fonts (Bricolage Grotesque, Newsreader, JetBrains
  Mono). Offline they fall back to system serif, sans and mono and every page
  still reads correctly.
- **Responsive.** The sidebar table of contents collapses to a scrolling chip
  row on narrow screens; wide tables and diagrams scroll inside their own
  containers rather than the page.

## Publishing

These are static files, so any static host will serve them as-is. GitHub Pages
is the least work — push the repo, enable Pages on the branch root, and
`index.html` becomes the landing page.

Before making them public, consider adding `description` and Open Graph meta
tags: without them, links shared on Slack or LinkedIn preview as bare
filenames.

## Conventions

- Lectures are numbered and named `NN-inside-<topic>.html`.
- New lectures reuse the shared stylesheet verbatim and get a card in
  `index.html`.
- Colour carries meaning inside diagrams and is consistent across a lecture —
  it is notation, not decoration.
