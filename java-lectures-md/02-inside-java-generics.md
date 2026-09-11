# Inside Java Generics

_java generics · from the cast to the wildcard_

> Generics are a _compile-time_ promise about what a container holds. Understanding them means understanding exactly how much of that promise survives to runtime — which is almost none of it.

---

## Contents

1. [Why generics exist](#why-generics-exist)
2. [The vocabulary](#the-vocabulary)
3. [Generic classes](#generic-classes)
4. [Generic methods](#generic-methods)
5. [Bounded type parameters](#bounded-type-parameters)
6. [Invariance](#invariance)
7. [Wildcards and PECS](#wildcards-and-pecs)
8. [Wildcard capture](#wildcard-capture)
9. [Type erasure](#type-erasure)
10. [What erasure forbids](#what-erasure-forbids)
11. [Arrays vs generics](#arrays-vs-generics)
12. [Raw types](#raw-types)
13. [Cheat sheet](#cheat-sheet)
14. [Ways to get hurt](#ways-to-get-hurt)

---

## Why generics exist

Before Java 5, every collection held `Object`. You knew what was inside; the compiler didn't. So you cast on the way out, and found out you were wrong in production.

```java
            // Java 1.4 — legal, and a trap
List names = new ArrayList();
names.add("Ada");
names.add(42);                       // nothing stops this

String s = (String) names.get(1);   // ClassCastException, at runtime

```

The cast is the tell. Every cast is a place where you know something the compiler doesn't, and every such place is a possible crash. Generics let you say the thing you know, once, in the type.

![Without generics an error surfaces at runtime after a cast; with generics the same error is caught at compile time](diagrams/inside-java-generics-01.svg)

**Generics buy you one thing: earlier failure.** They add no speed and no runtime capability — the entire benefit is that a whole class of bug becomes uncompilable.

```java
            // Same code, typed
List<String> names = new ArrayList<>();
names.add("Ada");
names.add(42);                // ✗ won't compile

String s = names.get(0);      // no cast — the compiler already knows

```

## The vocabulary

Four words that get used interchangeably in conversation and mean quite different things in the spec.

| Term                 | What it is                                    | Example        |
| -------------------- | --------------------------------------------- | -------------- |
| `type parameter`     | The placeholder in the declaration            | `class Box<T>` |
| `type argument`      | The real type you supply at the use site      | `Box<String>`  |
| `parameterized type` | The combination of the two                    | `Box<String>`  |
| `raw type`           | The generic type used with no argument at all | `Box`          |

### The conventional single letters

Nothing enforces these, but every Java codebase follows them, and breaking the convention makes your code read as unfamiliar for no gain.

- `T` — type. The generic default.
- `E` — element. Used throughout the collections framework.
- `K`, `V` — key and value, for anything map-shaped.
- `R` — result, usually the return type of a function.
- `S`, `U` — second and third type when you need more.

## Generic classes

One declaration, any number of concrete types. The type parameter behaves like a field of the class — visible to every instance method.

| Property           | Value                |
| ------------------ | -------------------- |
| **Declared with**  | `class Name<T>`      |
| **Scope of T**     | The whole class body |
| **Static context** | T not available      |
| **Runtime cost**   | None                 |

```java
            class Box<T> {
    private T item;

    void put(T item) { this.item = item; }
    T    take()          { return item; }

    // ✗ won't compile — statics belong to the class, which has no T
    // static T shared;
}

Box<String> b = new Box<>();   // diamond: inferred from the left
b.put("hello");
String s = b.take();               // no cast

// More than one parameter is ordinary
class Pair<K, V> {
    private final K key;
    private final V value;
    Pair(K key, V value) { this.key = key; this.value = value; }
}

```

![One generic class declaration substituting three different type arguments to produce three distinct compile-time types](diagrams/inside-java-generics-02.svg)

**Substitution, not inheritance.** `Box<String>` and `Box<Integer>` are as unrelated to each other as `String` and `Integer` are — a fact that surprises people in Part 6.

## Generic methods

A method can declare its own type parameter, independent of its class. The declaration goes between the modifiers and the return type — the position everyone forgets.

```java
            //     ↓ the declaration lives here
static <T> T firstOrNull(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}

// T is inferred from the argument — you rarely name it
String s = firstOrNull(listOfStrings);

// but you can, when inference can't get there
List<String> empty = Collections.<String>emptyList();

// Several parameters, and one that relates two types
static <K, V> Map<V, K> invert(Map<K, V> in) { /* … */ }

```

> **A static method can be generic — a static field cannot**
>
> This trips people constantly. `static <T> T pick(...)` is fine, because the method declares its _own_ `T`, resolved per call. `static T field;` is not, because a class-level `T` only exists per-instance, and statics have no instance to get it from.

## Bounded type parameters

An unbounded `T` is only known to be an `Object`, so you can barely do anything with it. A bound trades away some flexibility for capability.

```java
            // Upper bound: T is at least a Number, so Number's methods are usable
static <T extends Number> double sum(List<T> nums) {
    double total = 0;
    for (T n : nums) total += n.doubleValue();   // allowed by the bound
    return total;
}

// Multiple bounds — a class first (if any), then interfaces, joined by &
<T extends Number & Comparable<T> & Serializable>

// Recursive bound — "T is comparable to its own kind".
// Looks alarming, is idiomatic, and is how Enum is declared.
static <T extends Comparable<T>> T max(List<T> list) {
    T best = list.get(0);
    for (T item : list) if (item.compareTo(best) > 0) best = item;
    return best;
}

```

Note that `extends` here means "is a subtype of", covering both classes and interfaces. There is no `implements` in a bound — the keyword is always `extends`, even for interfaces.

## Invariance

Every apple is a fruit. A `List<Apple>` is nonetheless **not** a `List<Fruit>`. This is the single most confusing rule in generics, and it exists for a reason you can see in four lines.

![If a list of apples could be assigned to a list of fruit, a banana could legally be added, breaking the original list's type](diagrams/inside-java-generics-03.svg)

**The assignment is banned to protect the writes.** Anyone holding a `List<Fruit>` reference is entitled to add any fruit to it — so a `List<Apple>` must never be reachable through one.

```java
            List<Apple> apples = new ArrayList<>();
List<Fruit> fruit  = apples;     // ✗ compile error — the guard
fruit.add(new Banana());            // would have been legal
Apple a = apples.get(0);           // would have exploded

```

Generic types in Java are therefore **invariant**: `Box<A>` is a subtype of `Box<B>` only when `A` and `B` are the same type. Wildcards are how you get controlled flexibility back.

## Wildcards and PECS

Two loosenings that are exact opposites. One makes a type readable, the other makes it writable. Neither gives you both, and that restriction is the whole point.

| Property           | Value       |
| ------------------ | ----------- |
| **? extends T**    | Read as T   |
| **Write into it**  | Only null   |
| **? super T**      | Write T     |
| **Read out of it** | Only Object |

![An extends wildcard permits reads and blocks writes, while a super wildcard permits writes and degrades reads to Object](diagrams/inside-java-generics-04.svg)

**PECS — Producer Extends, Consumer Super.** If the parameter hands data to you, use `extends`. If it swallows data you supply, use `super`. If it does both, use a plain `T`.

```java
            // The canonical PECS signature, straight out of the JDK
public static <T> void copy(List<? super T> dest,     // consumer → super
                              List<? extends T> src) {  // producer → extends
    for (T item : src) dest.add(item);
}

// Why it matters — this call only compiles because of the wildcards
List<Object> dest = new ArrayList<>();
List<Integer> src = List.of(1, 2, 3);
copy(dest, src);                     // ✓

// The unbounded wildcard: "a list of something". Read-only in practice.
static void printAll(List<?> list) {
    for (Object o : list) System.out.println(o);
    // list.add(anything);   ✗ — except literal null
}

```

> **Where wildcards belong**
>
> Use them on **method parameters**, to accept more callers. Do _not_ use them as **return types** — you push the awkwardness onto every caller, who then has to deal with a wildcard they can't name. And never on fields.

## Wildcard capture

Occasionally the compiler refuses something obviously safe, because it can't prove two wildcards refer to the same type. The fix is a private helper — an idiom worth recognising.

```java
            // Swap two elements. Looks fine. Doesn't compile.
static void swap(List<?> list, int i, int j) {
    Object tmp = list.get(i);
    list.set(i, list.get(j));      // ✗ can't add to List<?>
    list.set(j, tmp);              // ✗
}

// The capture helper: give the wildcard a NAME, once.
static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}
private static <T> void swapHelper(List<T> list, int i, int j) {
    T tmp = list.get(i);          // now T is a real, single type
    list.set(i, list.get(j));
    list.set(j, tmp);
}

```

The public signature keeps the wildcard so callers stay unconstrained; the private method captures it as a concrete `T` so the body type-checks. You'll see `…Helper` methods like this throughout the JDK for exactly this reason.

## Type erasure

The compiler checks every type argument — then **deletes them**. At runtime there is no such thing as a `List<String>`; there is only a `List`.

![Source code carrying type arguments compiles to bytecode where the parameters are replaced by their bounds and casts are inserted](diagrams/inside-java-generics-05.svg)

**Why it works this way:** Java 5 had to run existing Java 1.4 bytecode unchanged. Erasure was the price of that compatibility — and every restriction in the next section is a consequence of it.

```java
            List<String>  a = new ArrayList<>();
List<Integer> b = new ArrayList<>();

a.getClass() == b.getClass();       // true — both are just ArrayList
System.out.println(a.getClass());   // class java.util.ArrayList

```

### What survives erasure

Not quite everything is deleted. Type arguments that appear in a **class or method signature** are kept in the class file as metadata, which is how reflection and frameworks recover them.

```java
            // Erased at runtime — a local variable's type argument is gone
List<String> local = new ArrayList<>();

// Retained — a field's declared generic type is readable via reflection
class Holder { List<String> names; }
Type t = Holder.class.getDeclaredField("names").getGenericType();
// → java.util.List<java.lang.String>

// This is the trick behind the "super type token" you see in Jackson, Guice…
new TypeReference<List<User>>() {}   // anonymous subclass ⇒ signature retained

```

## What erasure forbids

Seven rules that look arbitrary in isolation and are all the same rule: _the type argument is not there at runtime._

| You can't write                      | Because                              | Do instead                  |
| ------------------------------------ | ------------------------------------ | --------------------------- |
| `new T()`                            | No class object to instantiate       | `pass a Supplier<T>`        |
| `new T[10]`                          | Arrays need a reified component type | `(T[]) new Object[10]`      |
| `x instanceof List<String>`          | Nothing to test against              | `x instanceof List<?>`      |
| `List<int>`                          | Primitives aren't Objects            | `List<Integer>`             |
| `static T field;`                    | No instance to resolve T             | make it an instance field   |
| `class E<T> extends Exception`       | catch needs an exact runtime type    | use a non-generic exception |
| `f(List<String>) + f(List<Integer>)` | Both erase to f(List)                | rename one method           |

### Heap pollution and `@SafeVarargs`

A generic varargs parameter secretly creates an array of a non-reifiable type, which the compiler cannot fully guarantee. Hence the warning — and the annotation that silences it once you've checked.

```java
            // warning: possible heap pollution from parameterized vararg type T
static <T> List<T> listOf(T... items) { return Arrays.asList(items); }

// Safe ONLY if you never write into the array and never let it escape.
@SafeVarargs
static <T> List<T> listOf(T... items) { return List.of(items); }

```

> ⚠️ **The annotation is a promise, not a check**
>
> `@SafeVarargs` suppresses the warning without verifying anything. If the method stores the array somewhere or hands it out, you've silenced a real warning and bought yourself a `ClassCastException` in unrelated code later. It's allowed only on `static`, `final`, or `private` methods — precisely because those can't be overridden into something unsafe.

## Arrays vs generics

They behave in opposite ways, and mixing them is where the sharpest edges live. Arrays remember their type at runtime; generics don't. Arrays are covariant; generics aren't.

![An array of strings assigned to an object array accepts a wrong write and fails at runtime, while the generic equivalent is rejected at compile time](diagrams/inside-java-generics-06.svg)

**Don't create arrays of generic types.** `new List<String>[10]` is illegal outright; the `(T[]) new Object[n]` cast is the standard workaround, and it's what `ArrayList` itself does internally.

## Raw types

Using a generic type with no type argument. It compiles, it warns, and it switches off checking for the whole expression — including parts that look unrelated.

```java
            List raw = new ArrayList();       // raw — legal, for 1.4 compatibility only
raw.add("anything");
raw.add(42);                        // nobody objects

List<String> typed = raw;          // unchecked warning — the poison enters here
String s = typed.get(1);            // ClassCastException at runtime

// If you genuinely mean "a list of unknown element type", say so:
List<?> unknown = new ArrayList<String>();   // safe, and checked

```

The difference matters: `List` means "I have opted out of type checking"; `List<?>` means "the element type is unknown, so protect me from writing the wrong thing". The second is safe, the first is not.

> **Treat unchecked warnings as errors**
>
> Compile with `-Xlint:unchecked`. Every unchecked warning marks a spot where the compiler has stopped being able to help you, and where a `ClassCastException` can appear in code that looks entirely innocent. Where a cast is genuinely provable, narrow the suppression to the smallest possible scope and leave a comment saying why it's safe.

```java
            // Narrow, justified, documented — the only acceptable form
@SuppressWarnings("unchecked")   // safe: the map only ever holds String values
List<String> result = (List<String>) cache.get(key);

```

## Cheat sheet

### Choosing a form

| I want to…                                 | Use                       |
| ------------------------------------------ | ------------------------- |
| A container of one known type              | `Box<T>`                  |
| A parameter I only read from               | `? extends T`             |
| A parameter I only write into              | `? super T`               |
| A parameter I read _and_ write             | `plain T`                 |
| I don't care about the element type at all | `List<?>`                 |
| T needs a capability                       | `T extends Number`        |
| T compared against its own kind            | `T extends Comparable<T>` |
| Two types that vary together               | `<K, V>`                  |

#### Compile time vs runtime

| Question                              | At compile time    | At runtime                |
| ------------------------------------- | ------------------ | ------------------------- |
| Is this a `List<String>`?             | known and enforced | unknowable                |
| Can I add an `Integer` to it?         | refused            | nothing checks            |
| What is `T` here?                     | a real type        | its bound, usually Object |
| Are the casts gone?                   | you wrote none     | compiler inserted them    |
| Does a field's type argument survive? | yes                | yes — in metadata         |
| Does a local's type argument survive? | yes                | no                        |

## Ways to get hurt

### Assuming `Box<Dog>` is a `Box<Animal>`

The most common generics compile error, and the one people fight rather than understand. It isn't the compiler being pedantic — see the banana in Part 6. When you want the flexibility, ask for it explicitly with a wildcard.

### Reaching for a wildcard when a type parameter is right

```java
            // ✗ Two independent unknowns — nothing ties them together
static void copy(List<?> dst, List<?> src) { /* can't write into dst */ }

// ✓ One named type relates both sides
static <T> void copy(List<? super T> dst, List<? extends T> src) { … }

```

Rule of thumb: if a type appears in **one** place in the signature, a wildcard is fine. If it appears in **two** places that must agree, you need a named type parameter.

### Silencing warnings instead of reading them

A class-level `@SuppressWarnings("unchecked")` hides every unchecked operation in the file, including ones added years later by someone else. Put it on the smallest unit that works — ideally a single local variable declaration.

### Expecting overloads to distinguish type arguments

`process(List<String>)` and `process(List<Integer>)` are the same method after erasure, so the file won't compile. Give them different names — `processNames` and `processIds` — which reads better anyway.

### Returning a wildcard

`List<? extends Fruit> getFruit()` forces every caller into wildcard handling for no benefit. Return the concrete parameterized type and keep the wildcards on your parameters.

> **The short version**
>
> Generics are checked thoroughly and then thrown away. Write the strongest types you can at the boundaries, use `extends` and `super` to widen what callers may pass, never suppress a warning you haven't personally proven safe — and remember that at runtime, every one of your carefully typed collections is just a collection.
