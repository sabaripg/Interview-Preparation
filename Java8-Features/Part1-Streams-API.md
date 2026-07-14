# Part 1 — Streams API

> What a Stream actually is (not a data structure), creating streams, sorting, intermediate vs terminal operations and lazy evaluation, a full deep-dive on `flatMap` (why it exists, every common shape it solves), primitive streams (`IntStream`/`DoubleStream`/`LongStream`, `mapToInt`/`mapToDouble`, `sum`/`average`/`max`/`min`), `takeWhile`/`dropWhile`, the Collectors toolkit (`groupingBy`/`partitioningBy`/`joining`/`mapping`), `Optional`, and parallel streams — including exactly when not to reach for them. Interview Q&A at the end.

## What a Stream Actually Is

**What it does:** a `Stream` is a one-time-use pipeline for processing a sequence of elements declaratively — you describe *what* transformation you want (map, filter, sum), not *how* to loop and accumulate it yourself. It is **not** a data structure: a stream holds no elements of its own, doesn't store anything, and can't be reused once a terminal operation has consumed it.

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Dave");
List<String> longNames = names.stream()
    .filter(n -> n.length() > 3)
    .map(String::toUpperCase)
    .collect(Collectors.toList());
System.out.println(longNames);
```
**Output:**
```
[ALICE, CHARLIE, DAVE]
```
> ⚠️ **Pitfall:** a stream can only be consumed **once**. Calling a second terminal operation on an already-consumed stream throws `IllegalStateException: stream has already been operated upon or closed` — a stream is a single-pass pipeline, not something you can iterate twice like a `List`. If you need the same data processed two different ways, either build two separate stream pipelines from the source collection, or collect the intermediate result into a `List` first.

## Creating Streams

```java
Stream.of("a", "b", "c");                          // from varargs
List.of(1, 2, 3).stream();                          // from any Collection
Arrays.stream(new int[]{1, 2, 3});                   // from an array -- returns IntStream
IntStream.range(0, 5);                               // 0,1,2,3,4 -- exclusive upper bound
IntStream.rangeClosed(0, 5);                         // 0,1,2,3,4,5 -- inclusive
Stream.iterate(1, n -> n * 2).limit(5);               // 1,2,4,8,16 -- infinite, must be bounded
Stream.generate(Math::random).limit(3);               // infinite, must be bounded
```
> ⚠️ **Pitfall:** `Stream.iterate()` and `Stream.generate()` produce **infinite** streams — forgetting `.limit(n)` on one hangs the program (or eventually throws `OutOfMemoryError` if a terminal operation like `collect()` tries to exhaust it). Always pair an infinite source with a bounding intermediate operation.

## Sorting Streams — Natural Order, Reverse, Comparator, By Attribute

```java
List<Integer> numbers = Arrays.asList(3, 45, 5, 2, 5, 3, 22, 40, 4);

numbers.stream().sorted().collect(Collectors.toList());                       // natural order: needs Comparable
numbers.stream().sorted(Comparator.reverseOrder()).collect(Collectors.toList()); // reverse natural order

Comparator<Integer> comp = (i1, i2) -> i1.compareTo(i2);
numbers.stream().sorted(comp).collect(Collectors.toList());                   // explicit comparator, same effect as natural order here
```
**Sorting objects by an attribute** — this is the shape that actually comes up in interviews, not sorting bare integers:
```java
record Student(String name, int age) {}
List<Student> students = List.of(new Student("Tim", 30), new Student("Sim", 22), new Student("Sam", 28));

students.stream().sorted(Comparator.comparingInt(Student::age)).toList();                 // ascending by age
students.stream().sorted(Comparator.comparing(Student::name).reversed()).toList();          // descending by name
students.stream()
    .sorted(Comparator.comparing(Student::age).thenComparing(Student::name))               // multi-key sort
    .toList();
```
> ⚠️ **Pitfall — `sorted()` with no comparator requires `Comparable`:** calling the no-argument `sorted()` on a stream of a custom type that does **not** implement `Comparable<T>` throws `ClassCastException` at runtime (`class Student cannot be cast to class java.lang.Comparable`) — not a compile-time error, because the compiler can't verify comparability for an arbitrary generic `Stream<T>`. Either implement `Comparable` on the type, or (the far more common real-world fix) always pass an explicit `Comparator` via `Comparator.comparing(...)` instead of relying on natural ordering existing.

## Intermediate vs Terminal Operations — and Why Laziness Matters

**Intermediate operations** (`map`, `filter`, `sorted`, `distinct`, `limit`, `skip`, `flatMap`, `peek`) return a new `Stream` and are **lazy** — they don't actually run anything, they just build up a pipeline description. **Terminal operations** (`collect`, `reduce`, `forEach`, `count`, `anyMatch`/`allMatch`/`noneMatch`, `findFirst`/`findAny`) trigger the actual execution, and only then does the pipeline run, element by element, through every intermediate stage.

```java
Stream<String> pipeline = names.stream()
    .filter(n -> { System.out.println("filtering " + n); return n.length() > 3; })
    .map(n -> { System.out.println("mapping " + n); return n.toUpperCase(); });
System.out.println("Pipeline built, nothing has printed yet.");
pipeline.collect(Collectors.toList()); // ONLY NOW does filtering/mapping actually run
```
**Output:**
```
Pipeline built, nothing has printed yet.
filtering Alice
filtering Bob
filtering Charlie
mapping Charlie
filtering Dave
mapping Dave
```
Notice the order: it's not "filter everything, then map everything" — each element flows through the **entire** pipeline (filter → map) before the next element starts. This is why streams can short-circuit efficiently: `findFirst()` after a `filter()` can stop at the very first match without evaluating the rest of the source at all.

> ⚠️ **Pitfall — the classic "why didn't my code run" bug:** building a stream pipeline with no terminal operation at the end does **nothing** — no exception, no output, just silently no-op code, because nothing ever triggered execution. This is the single most common stream mistake for developers new to the API: forgetting the terminal operation entirely and wondering why `peek()`/`map()` side effects never happened.

## `flatMap` — Why It Exists, and Every Shape It Solves

**The core problem `flatMap` exists to solve:** `map()` is strictly 1-to-1 — one input element produces exactly one output element. The moment an input element itself produces a *collection* of things (zero, one, or many), `map()` leaves you with a **stream of streams** (or a stream of lists) — a nested, awkward shape you'd then have to unwrap by hand. `flatMap()` does the 1-to-many transformation **and** the flattening in one step: it maps each element to its own stream, then merges every one of those streams into a single, flat result stream.

```
map():      [A, B, C]  --f-->  [f(A), f(B), f(C)]                     -- still one output per input
flatMap():  [A, B, C]  --f-->  f(A) ++ f(B) ++ f(C)  (all concatenated into ONE flat stream)
```

### Case 1 — Flattening a List of Lists (the textbook case)
```java
List<List<Integer>> nested = List.of(List.of(1, 2, 3), List.of(4, 5), List.of(6));
List<Integer> flat = nested.stream()
    .flatMap(List::stream)
    .collect(Collectors.toList());
System.out.println(flat); // [1, 2, 3, 4, 5, 6]
```
Without `flatMap`, `nested.stream().map(List::stream)` would produce a `Stream<Stream<Integer>>` — a stream *of streams* — which is almost never what you actually want to work with.

### Case 2 — One Input, Many Outputs (splitting/expanding each element)
```java
List<String> sentences = List.of("hello world", "streams are powerful");
List<String> words = sentences.stream()
    .flatMap(sentence -> Arrays.stream(sentence.split(" ")))
    .collect(Collectors.toList());
System.out.println(words); // [hello, world, streams, are, powerful]
```
Each sentence becomes *multiple* words — `map()` alone could only ever give you one output per sentence (e.g., the array itself, not its individual elements); `flatMap` is what actually explodes each sentence into its constituent words and merges them all into one flat stream of words.

### Case 3 — Flattening a Stream of Custom Objects' Nested Collections
```java
record Employee(String name, List<String> projects) {}
List<Employee> employees = List.of(
    new Employee("Alice", List.of("Alpha", "Beta")),
    new Employee("Bob", List.of("Beta", "Gamma"))
);
List<String> allDistinctProjects = employees.stream()
    .flatMap(e -> e.projects().stream())
    .distinct()
    .collect(Collectors.toList());
System.out.println(allDistinctProjects); // [Alpha, Beta, Gamma]
```
This is the real-world shape that shows up constantly: each employee "contains" a list of projects; you want one flat list of every project across every employee, deduplicated. `flatMap` plus `distinct()` is the standard combination for exactly this "find all the unique children across all parents" pattern.

### Case 4 — Flattening an `Optional<T>` (a lesser-known but real use)
```java
Optional<String> maybeName = Optional.of("Alice");
Stream<String> asStream = maybeName.stream(); // Optional.stream() -- 0 or 1 elements
List<Optional<String>> maybeNames = List.of(Optional.of("Alice"), Optional.empty(), Optional.of("Bob"));
List<String> presentNames = maybeNames.stream()
    .flatMap(Optional::stream) // Java 9+ Optional.stream(): empty Optionals contribute nothing, present ones contribute their value
    .collect(Collectors.toList());
System.out.println(presentNames); // [Alice, Bob] -- empties silently dropped
```
`Optional.stream()` (added in Java 9) turns an `Optional` into a stream of zero or one elements — combined with `flatMap`, this is a clean, idiomatic way to run a stream of computations that may or may not produce a value, then collect only the ones that did, without any manual null-checking or `isPresent()` branching.

> ⚠️ **Pitfall — the practical benefit, stated plainly:** using `map()` where `flatMap()` was actually needed produces a `Stream<List<Integer>>` (or `Stream<Stream<Integer>>`) instead of the flattened result you wanted — a very common early mistake, and the compiler won't catch it, since `Stream<List<Integer>>` is a perfectly valid, compilable type, just not the shape you meant to end up with. The tell that you need `flatMap` instead of `map`: the transformation function you're about to write returns a *collection or stream*, not a single value.

## Primitive Streams — `IntStream`, `DoubleStream`, `LongStream`

**Why they exist:** a regular `Stream<Integer>` or `Stream<Double>` stores **boxed** wrapper objects — every element is a full heap object (`Integer`, `Double`), not a raw primitive (see `core-java/StringBuilder-Immutability.md`-adjacent boxing-overhead numbers in the core-java Wrapper Classes section — `Integer` costs roughly 4x an `int` in memory, and boxing/unboxing costs real CPU too). `IntStream`, `LongStream`, and `DoubleStream` are specialized stream types that hold **raw primitives directly**, avoiding both the memory overhead and the boxing/unboxing cost on every operation — and they add numeric-specific terminal operations (`sum()`, `average()`, `max()`, `min()`, `summaryStatistics()`) that a generic `Stream<T>` simply doesn't have, because summing or averaging arbitrary objects doesn't make general sense.

```java
List<Employee> employees = Employee.employeeData(); // assume salary is a double field

// mapToDouble converts a Stream<Employee> into a DoubleStream -- unwrapping the salary field AS A PRIMITIVE
double totalSalary = employees.stream()
    .mapToDouble(Employee::getSalary)
    .sum(); // sum() only exists on DoubleStream/IntStream/LongStream, NOT on Stream<Double>

OptionalDouble avgSalary = employees.stream()
    .mapToDouble(Employee::getSalary)
    .average(); // average() -- also primitive-stream-only

DoubleSummaryStatistics stats = employees.stream()
    .mapToDouble(Employee::getSalary)
    .summaryStatistics(); // min/max/average/sum/count ALL IN ONE PASS
System.out.println(stats.getMin() + " / " + stats.getMax() + " / " + stats.getAverage());
```
**The exact gotcha this trips people up with:**
```java
// WRONG -- .map() keeps you in the boxed Stream<Double> world
OptionalDouble avg = employees.stream()
    .map(Employee::getSalary)   // Stream<Double> -- if getSalary() returns double, this actually won't even compile cleanly without boxing
    .average();                 // COMPILE ERROR -- average() does not exist on Stream<Double>

// RIGHT -- .mapToDouble() converts to the primitive DoubleStream, which DOES have average()
OptionalDouble avg = employees.stream()
    .mapToDouble(Employee::getSalary)
    .average();
```
> ⚠️ **Pitfall — this is a real, frequent compile-time trap, not a style nitpick:** `Stream<T>` (the generic/boxed stream) does **not** have `sum()`, `average()`, or `summaryStatistics()` at all — those methods only exist on `IntStream`/`LongStream`/`DoubleStream`. The fix is always to switch to the primitive stream via `mapToInt()`/`mapToDouble()`/`mapToLong()` *before* calling the numeric terminal operation, not to try to call it on the boxed stream. Going back the other way — from a primitive stream to a boxed one — uses `.boxed()` (`IntStream.range(0,5).boxed()` → `Stream<Integer>`), needed specifically when a later operation (like `collect(Collectors.toList())`) requires the boxed `Stream<T>` API.

**Getting a max/min value directly on a boxed stream (the non-primitive alternative):**
```java
int max = numbers.stream().mapToInt(Integer::intValue).max().getAsInt();      // primitive-stream style
Integer max2 = numbers.stream().max(Comparator.naturalOrder()).orElseThrow(); // boxed-stream style, using max(Comparator) instead
```
Both are legitimate — the primitive-stream version avoids boxing overhead and is idiomatic for pure numeric data; the boxed `max(Comparator)`/`min(Comparator)` form is what you reach for on a `Stream<T>` of custom objects where you're comparing by some field rather than the elements themselves.

## `takeWhile()` and `dropWhile()` (Java 9+)

**What they do:** both take a `Predicate` and are specifically designed for **already-ordered/sorted** data, letting you slice a stream around the point the predicate stops holding — **without** evaluating the entire stream, unlike `filter()`.

```java
List<Integer> sortedCalories = List.of(120, 300, 350, 400, 530, 800, 900); // must be SORTED for these to behave sensibly

List<Integer> under400 = sortedCalories.stream()
    .takeWhile(cal -> cal < 400)     // takes elements FROM THE START, stops at the FIRST element that fails the predicate
    .collect(Collectors.toList());
System.out.println(under400); // [120, 300, 350]

List<Integer> fromFourHundredOn = sortedCalories.stream()
    .dropWhile(cal -> cal < 400)     // DROPS elements from the start while the predicate holds, keeps everything from the first failure onward
    .collect(Collectors.toList());
System.out.println(fromFourHundredOn); // [400, 530, 800, 900]
```
**Why not just use `filter()` for this?** `filter()` evaluates the predicate against **every single element** in the stream, regardless of position. `takeWhile()` **stops evaluating entirely** the moment it hits the first element that fails the predicate — for a large, already-sorted stream, that's a real, measurable difference: `filter(cal -> cal < 400)` on a million-element sorted list still touches all one million elements; `takeWhile(cal -> cal < 400)` stops at the first failure, potentially touching only a handful.

> ⚠️ **Pitfall — the exact trap the source material for this section calls out:** `takeWhile()`/`dropWhile()` have **no bearing on whether the stream is actually sorted** — they blindly stop (or start) at the *first* element that fails the predicate, in encounter order, whatever that order happens to be. Running `takeWhile(cal -> cal < 800)` against an **unsorted** list can silently return a wrong, incomplete-looking result (e.g., just the first matching run before an early failure), even though later elements further down the unsorted list would also have satisfied the predicate. **Always `sorted()` the stream first** if you intend to use `takeWhile`/`dropWhile` for a genuine threshold cutoff — they are not a substitute for `filter()` on arbitrary unordered data, only a more efficient tool for a specific "cut off at this point in already-ordered data" scenario.

## Collectors — the Real Toolkit Beyond `toList()`

```java
record Employee(String name, String department, double salary) {}
List<Employee> employees = List.of(
    new Employee("Alice", "Engineering", 95000),
    new Employee("Bob", "Engineering", 87000),
    new Employee("Charlie", "Sales", 72000),
    new Employee("Dave", "Sales", 68000)
);

// groupingBy -- Map<K, List<V>>
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department));

// groupingBy + downstream collector -- Map<K, D>
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department, Collectors.averagingDouble(Employee::salary)));

// groupingBy + mapping -- extract ONE field per group instead of whole objects
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department, Collectors.mapping(Employee::name, Collectors.toList())));
// {Engineering=[Alice, Bob], Sales=[Charlie, Dave]}

// partitioningBy -- always exactly TWO groups, keyed by boolean
Map<Boolean, List<Employee>> aboveAvg = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.salary() > 80000));

// joining -- building a delimited String
String names = employees.stream().map(Employee::name).collect(Collectors.joining(", ", "[", "]"));
// "[Alice, Bob, Charlie, Dave]"

// toMap -- careful with duplicate keys
Map<String, Double> nameToSalary = employees.stream()
    .collect(Collectors.toMap(Employee::name, Employee::salary));

// counting, summingDouble, summarizing
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department, Collectors.counting()));
Map<String, Double> totalSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department, Collectors.summingDouble(Employee::salary)));
DoubleSummaryStatistics stats = employees.stream()
    .collect(Collectors.summarizingDouble(Employee::salary)); // min/max/avg/sum/count in one pass
```
> ⚠️ **Pitfall — `Collectors.mapping()` is asked specifically to test this distinction:** a plain `groupingBy(Employee::department)` gives you `Map<String, List<Employee>>` — the **whole object** in each group. If you only want one field (say, just the names), the naive-but-wrong instinct is to `map()` *before* grouping — but you can't `map()` before `groupingBy` and still group by the original field cleanly in one pass. `Collectors.mapping(extractorFn, downstreamCollector)` is the correct tool: it's a **downstream collector** that transforms each element *within* each group, after grouping has already happened, letting you group by one field while collecting a different, extracted field.

> ⚠️ **Pitfall — `toMap()` with duplicate keys:** `Collectors.toMap(keyFn, valueFn)` throws `IllegalStateException: Duplicate key` at runtime if two elements produce the same key — not a compile-time-visible risk, and easy to miss until production data has a collision dev data didn't. Fix by supplying a third **merge function** argument: `Collectors.toMap(keyFn, valueFn, (existing, replacement) -> existing)` to explicitly decide what happens on collision, rather than letting it throw.

**`groupingBy` vs `partitioningBy` — the real distinction:** `groupingBy` can produce any number of groups, keyed by whatever the classifier function returns. `partitioningBy` is a specialized, more efficient case that **always** produces exactly two groups (`true`/`false`), because the classifier is a `Predicate`, not an arbitrary function — reach for `partitioningBy` specifically when the split is genuinely binary, since it's a clearer signal of intent than `groupingBy` with a boolean-returning classifier.

## `Optional` — Avoiding Null Checks in a Stream Pipeline

```java
Optional<Employee> highestPaid = employees.stream()
    .max(Comparator.comparingDouble(Employee::salary));

String result = highestPaid
    .map(Employee::name)
    .orElse("No employees found");
```
`Optional` represents "a value that might not be present" as a real type the compiler forces you to handle, rather than a `null` that silently propagates until some unrelated line throws `NullPointerException`. Key methods: `isPresent()`/`isEmpty()` (check without extracting), `orElse(default)` (eager default, always evaluated even if not used), `orElseGet(supplier)` (lazy default, only evaluated if actually needed), `orElseThrow()` (convert absence into an explicit exception), `map()`/`filter()` (chain transformations that short-circuit to empty if the Optional is already empty).

> ⚠️ **Pitfall — `orElse()` vs `orElseGet()` is not just style:** `orElse(expensiveDefaultValue())` **always** evaluates `expensiveDefaultValue()` eagerly, even when the `Optional` already has a value and the default is never used — a real, easy-to-miss performance cost if the default involves a database call or heavy computation. `orElseGet(() -> expensiveDefaultValue())` only evaluates the supplier lazily, when actually needed. Also: `Optional.get()` without checking presence first defeats the entire purpose of `Optional` — it throws `NoSuchElementException`, which is exactly the unchecked-null-access problem `Optional` exists to prevent, just with a different exception name.

> ⚠️ **Pitfall — `Optional` is for return types, not fields:** declaring a class field as `Optional<String> department;` is a well-known anti-pattern — `Optional` was designed as a *return type* signal ("this method might not produce a value"), not as a general-purpose null-safety wrapper for object state. It adds serialization complications (`Optional` isn't `Serializable`), extra indirection for no real benefit over just checking a nullable field directly, and most static-analysis/style guides (and `Optional`'s own Javadoc) explicitly discourage it as a field type.

## `forEach()` Is Not for Accumulating State

```java
// BAD -- using forEach to accumulate external mutable state defeats the point of a stream pipeline
double total = 0;
employees.stream()
    .filter(e -> e.salary() > 50000)
    .forEach(e -> total += e.salary()); // won't even compile as-is (total isn't effectively final) -- and the pattern is wrong even if worked around

// GOOD -- let the stream do the accumulation via a proper terminal/reduction operation
double total2 = employees.stream()
    .filter(e -> e.salary() > 50000)
    .mapToDouble(Employee::salary)
    .sum();
```
> ⚠️ **Pitfall:** streams are designed around **stateless, side-effect-free** operations — `forEach` mutating an external variable (especially under `parallelStream()`, where it becomes an outright data race) is exactly the anti-pattern the API is trying to steer you away from. If you find yourself reaching for an external accumulator inside `forEach`, there's almost always a proper collector or numeric terminal operation (`sum`, `collect`, `reduce`) that expresses the same intent without a mutable side variable — reach for those first, and reserve `forEach` for genuine side effects (logging, printing) that aren't building up a result.

## Parallel Streams — and Why You Probably Don't Want One by Default

```java
long count = employees.parallelStream()
    .filter(e -> e.salary() > 80000)
    .count();
```
**What it actually does:** splits the source into chunks and processes them concurrently using the **common `ForkJoinPool`** (`ForkJoinPool.commonPool()`, sized to `Runtime.getRuntime().availableProcessors() - 1` by default) — the same shared pool used by parallel `Arrays.parallelSort()` and, unless explicitly configured otherwise, `CompletableFuture.supplyAsync()` too.

> ⚠️ **Pitfall — the real reasons parallel streams are not a free performance win:** (1) **Shared pool contention** — because `parallelStream()` uses the JVM-wide common pool by default, a CPU-heavy parallel stream running in one part of the application can starve unrelated `CompletableFuture` work elsewhere in the same JVM competing for the same threads (see the Multhithreading notes' `ForkJoinPool.commonPool()` section for the full mechanics). (2) **Overhead for small or I/O-bound workloads** — splitting, coordinating, and merging has real cost; for anything but genuinely large, CPU-bound, easily-partitionable workloads, a sequential stream is often faster. (3) **Ordering and shared mutable state** — using `forEach` with a shared, non-thread-safe collection inside a parallel stream is a silent data-race waiting to happen (see the `forEach` section above — this is exactly why that anti-pattern is worse under `parallelStream()`); if order matters, `forEachOrdered` gives it back but also removes most of the parallelism benefit. The senior-level answer to "would you use `parallelStream()` here" is not a reflexive yes — it's "only after measuring, on a large CPU-bound collection, with awareness of what else is sharing the common pool."

---

## Interview Q&A

**Q: Why can't a stream be reused after a terminal operation?**
A stream isn't a data structure — it's a single-pass pipeline description with no backing storage. Once a terminal operation consumes it, the pipeline is spent; calling another terminal operation throws `IllegalStateException`. Build a fresh stream from the source collection if you need to process the same data a second way.

**Q: Explain why streams are lazy, and what breaks if you forget the terminal operation.**
Intermediate operations (`map`, `filter`, etc.) only build up a description of the pipeline; nothing executes until a terminal operation (`collect`, `forEach`, `count`, etc.) is called. Forgetting the terminal operation means the pipeline silently never runs — no error, just no-op code, which is one of the most common early mistakes with the API.

**Q: Why does `flatMap` exist — isn't `map` enough?**
`map()` is strictly one-to-one. The moment an element itself produces a collection of things (a list of projects, the words in a sentence, an `Optional` that may or may not hold a value), `map()` leaves you with a nested shape (a stream of lists, or a stream of streams) instead of the flat result you actually want. `flatMap` performs the one-to-many transformation *and* the flattening into one combined stream in a single step.

**Q: Give a concrete real-world case where you'd reach for `flatMap` beyond "flattening a list of lists."**
Extracting every distinct project across a list of employees, where each employee has their own list of projects: `employees.stream().flatMap(e -> e.getProjects().stream()).distinct().collect(toList())`. This "flatten each parent's children into one deduplicated list" shape comes up constantly with nested domain objects.

**Q: Why does `stream.map(Employee::getSalary).average()` fail to compile, and what's the fix?**
`average()`, `sum()`, and `summaryStatistics()` only exist on the primitive stream types (`IntStream`/`LongStream`/`DoubleStream`), not on a boxed `Stream<Double>`. The fix is `mapToDouble(Employee::getSalary).average()` — converting to the primitive stream *before* calling the numeric terminal operation, not after.

**Q: `takeWhile()` vs `filter()` — what's the actual difference, and what's the gotcha with unsorted data?**
`filter()` evaluates the predicate against every element regardless of position. `takeWhile()` stops evaluating entirely at the first element that fails the predicate, which is far cheaper on large already-sorted data — but it has no awareness of whether the data is actually sorted, so running it on unsorted data can silently cut off at the first failure even though later elements further down would also have matched. Always sort first if you intend `takeWhile`/`dropWhile` as a genuine threshold cutoff.

**Q: `groupingBy` alone gives you `Map<K, List<FullObject>>` — how do you group by one field while collecting just a different field, like names instead of whole objects?**
`Collectors.groupingBy(classifierFn, Collectors.mapping(extractorFn, Collectors.toList()))` — `Collectors.mapping()` is a downstream collector that transforms each element within a group after grouping has already happened, letting you extract a different field than the one you grouped by.

**Q: Why is using `forEach()` to accumulate a running total into an external variable considered an anti-pattern?**
Streams are designed around stateless operations; mutating an external variable inside `forEach` reintroduces exactly the imperative, side-effecting style streams exist to move away from — and under `parallelStream()` it becomes an outright data race. A proper collector or numeric terminal operation (`sum()`, `collect()`, `reduce()`) expresses the same accumulation without a mutable side variable.

**Q: `orElse()` vs `orElseGet()` on an `Optional` — is there a real difference?**
Yes — `orElse(value)` always evaluates its argument eagerly, even when the Optional already holds a value and the default is discarded. `orElseGet(supplier)` only invokes the supplier lazily, when the Optional is actually empty. If the default is expensive to compute (a DB call, heavy work), using `orElse()` pays that cost unconditionally.

**Q: Would you use `parallelStream()` for a given piece of code — how do you decide?**
Not reflexively. Parallel streams share the JVM-wide common `ForkJoinPool`, so CPU-heavy parallel stream work can starve unrelated concurrent work (including `CompletableFuture` tasks) competing for the same pool. They pay real coordination overhead, so they only help on large, CPU-bound, easily-partitioned workloads — and using shared mutable state inside one via `forEach` is a real data-race risk. Measure before reaching for it.
