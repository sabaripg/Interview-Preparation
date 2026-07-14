# Part 3 — Method References

> The four kinds of method references, when they genuinely read clearer than the equivalent lambda (and when they don't), and the ambiguous-overload-resolution gotcha that trips people up. Interview Q&A at the end.

## What a Method Reference Is

**What it does:** a method reference is shorthand for a lambda that does nothing but call one existing method — `Class::method` instead of `x -> Class.method(x)` (or `x -> x.method()`). It's not a different mechanism from a lambda under the hood; it still targets a functional interface and still compiles via the same `invokedynamic`/`LambdaMetafactory` path — it's purely a more concise way to express "call this existing method" instead of writing a new lambda body that just forwards its arguments.

## The Four Kinds

```java
// 1. Static method reference -- Class::staticMethod
Function<String, Integer> parse = Integer::parseInt;
// equivalent lambda: s -> Integer.parseInt(s)

// 2. Instance method reference on a PARTICULAR, already-existing object -- instance::method
String greeting = "Hello";
Supplier<Integer> length = greeting::length;
// equivalent lambda: () -> greeting.length()

// 3. Instance method reference on an ARBITRARY object of a type, supplied later -- Class::instanceMethod
Function<String, Integer> lengthFn = String::length;
// equivalent lambda: s -> s.length()   -- the argument BECOMES the receiver

// 4. Constructor reference -- Class::new
Supplier<ArrayList<String>> listFactory = ArrayList::new;
Function<Integer, ArrayList<String>> sizedListFactory = ArrayList::new; // overload resolved by target functional interface's shape
```
**Kind 2 vs Kind 3 — the distinction that actually gets tested:** in kind 2, the object already exists at the point the method reference is created, and the functional interface's parameter(s) become the method's *arguments*. In kind 3, there's no object yet — the functional interface's *first* parameter becomes the *receiver* the method is called on, and any remaining parameters become the method's arguments.

```java
List<String> names = List.of("Charlie", "alice", "Bob");
names.stream().sorted(String::compareToIgnoreCase); // kind 3: first arg becomes receiver, second is compareToIgnoreCase's argument
// equivalent lambda: (a, b) -> a.compareToIgnoreCase(b)
```
> ⚠️ **Pitfall:** this is precisely why `String::compareToIgnoreCase` works as a two-argument `Comparator<String>` even though `compareToIgnoreCase` is declared as a **one-argument instance method** (`int compareToIgnoreCase(String str)`) — the *first* parameter the `Comparator` interface supplies becomes the receiver (`a.compareToIgnoreCase(...)`), and the *second* becomes the actual method argument. Miscounting which parameter becomes the receiver versus an argument is the most common source of confusion between kinds 2 and 3.

## When a Method Reference Is Genuinely Clearer — and When It Isn't

```java
// GOOD -- method reference reads cleanly, no added logic
names.forEach(System.out::println);
names.stream().map(String::toUpperCase);

// QUESTIONABLE -- the lambda would have needed extra logic anyway, so the method reference saves little
names.stream().map(n -> processName(n, someExtraConfig)); // can't be a plain method reference -- extra argument doesn't fit
```
The rule of thumb: reach for a method reference when the lambda body would do **nothing except call one existing method with the exact arguments already available** — the moment you need to add logic, reorder arguments, or supply an extra fixed argument the target method doesn't already expect in the right position, a lambda is clearer than contorting the code to force a method reference to fit.

> ⚠️ **Pitfall — ambiguous overload resolution:** if a class has multiple overloaded methods with the same name, `Class::method` can be genuinely ambiguous to the compiler depending on the target functional interface's shape, occasionally producing a confusing compile error that a plain lambda would have sidestepped entirely (because a lambda's parameter types are explicit in its body, disambiguating immediately). When a method reference produces an unclear compiler error about ambiguity, falling back to an explicit lambda is a legitimate, pragmatic fix — not a failure to "do it the modern way."

## Constructor References With Streams — a Common Real Pattern

```java
record Employee(String name) {}
List<String> names = List.of("Alice", "Bob");
List<Employee> employees = names.stream()
    .map(Employee::new) // constructor reference -- one arg (name) matches Employee's single-arg constructor
    .collect(Collectors.toList());
```
This is one of the most common real-world uses of constructor references — converting a stream of raw data into a stream of domain objects without writing `n -> new Employee(n)` for a mapping that's otherwise purely mechanical.

---

## Interview Q&A

**Q: Are method references a different mechanism from lambdas, or just shorthand?**
Just shorthand — a method reference still targets a functional interface and compiles through the same `invokedynamic`/`LambdaMetafactory` path as a lambda. It's purely a more concise way to write a lambda whose entire body is "call this one existing method."

**Q: What's the actual difference between `instance::method` and `Class::instanceMethod`?**
`instance::method` (kind 2) targets an object that already exists — the functional interface's parameters map directly to the method's arguments. `Class::instanceMethod` (kind 3) has no object yet — the functional interface's *first* parameter becomes the receiver the method is called on, and any remaining parameters become the method's actual arguments. This is why `String::compareToIgnoreCase` can serve as a two-argument `Comparator<String>` despite being declared as a one-argument instance method.

**Q: When would you prefer a lambda over a method reference, even though method references are considered more "modern"?**
When the lambda needs to do anything beyond forwarding its arguments directly to one existing method — extra logic, reordered arguments, an additional fixed parameter not in the right position, or when a method reference produces an ambiguous-overload compile error a lambda would avoid by making parameter types explicit.
