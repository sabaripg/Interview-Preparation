# Part 2 — Lambdas & Functional Interfaces

> What a lambda expression actually compiles to (not a hidden anonymous class), `@FunctionalInterface`, the core `java.util.function` interfaces you're expected to know cold, and the effectively-final variable-capture rule with the reason behind it. Interview Q&A at the end.

## What a Lambda Actually Is

**The shallow answer:** "syntactic sugar for an anonymous inner class implementing a single-method interface." **The precise answer, and the one that shows real understanding:** a lambda is compiled using the `invokedynamic` bytecode instruction and `LambdaMetafactory`, **not** desugared into a real anonymous class at compile time the way pre-Java-8 code required. The actual implementation class is generated **at runtime**, lazily, the first time that particular lambda site executes — not one-per-lambda-expression at compile time.

```java
Runnable r = () -> System.out.println("Hello from lambda");
```
```java
// Pre-Java-8 equivalent IN BEHAVIOR, but NOT in how it's actually compiled:
Runnable r = new Runnable() {
    @Override public void run() { System.out.println("Hello from lambda"); }
};
```
> ⚠️ **Pitfall — why this distinction actually matters, not just trivia:** anonymous classes each generate a separate `.class` file at compile time (`Outer$1.class`, `Outer$2.class`, ...), bloating the JAR and requiring a real classloading + object-instantiation cost for each one, every time. Lambdas avoid this — no extra `.class` file per lambda expression, and the JVM can reuse the generated implementation across invocations. This is a genuine performance and startup-time difference at scale, not just a stylistic one, and interviewers ask this specifically to separate "used lambdas" from "understands what a lambda is."

## `@FunctionalInterface` — What It Actually Enforces

**What it does:** marks an interface as having **exactly one abstract method** (a "SAM" — Single Abstract Method interface) — the shape a lambda expression can implement. The annotation is optional (any interface with exactly one abstract method already qualifies), but applying it makes the compiler **enforce** the constraint, turning an accidental second abstract method into a compile error instead of a confusing runtime surprise.

```java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);
    // default and static methods DON'T count against the "single abstract method" rule
    default int calculateTriple(int a, int b, int c) { return calculate(calculate(a, b), c); }
}

Calculator add = (a, b) -> a + b;
Calculator multiply = (a, b) -> a * b;
```
> ⚠️ **Pitfall:** `default` and `static` methods on an interface do **not** count toward the "single abstract method" requirement — only genuinely abstract methods do. This is precisely why `Comparator<T>` (which has several `default` methods like `reversed()`, `thenComparing()`) is still a valid functional interface usable with a lambda — it has exactly one abstract method (`compare`), and everything else is a default method providing extra behavior on top.

## The Core `java.util.function` Interfaces

```
| Interface           | Abstract Method       | Takes    | Returns | Typical Use                          |
|----------------------|------------------------|----------|---------|----------------------------------------|
| Supplier<T>          | T get()                | nothing  | T       | lazy value production, factories       |
| Consumer<T>          | void accept(T t)       | T        | nothing | side effects (forEach, logging)        |
| Function<T, R>       | R apply(T t)            | T        | R       | transformation (map)                   |
| Predicate<T>         | boolean test(T t)       | T        | boolean | filtering conditions                   |
| UnaryOperator<T>     | T apply(T t)             | T        | T       | Function<T,T> specialization           |
| BinaryOperator<T>    | T apply(T t1, T t2)      | T, T     | T       | reduce operations (sum, max)           |
| BiFunction<T, U, R>  | R apply(T t, U u)        | T, U     | R       | two-argument transformation            |
```
```java
Supplier<LocalDateTime> now = LocalDateTime::now;               // called lazily, each get() re-evaluates
Consumer<String> logger = msg -> System.out.println("[LOG] " + msg);
Function<String, Integer> length = String::length;
Predicate<String> isEmpty = String::isEmpty;
UnaryOperator<Integer> square = n -> n * n;
BinaryOperator<Integer> max = Integer::max;
BiFunction<String, String, String> concat = (a, b) -> a + b;
```
**Composing functional interfaces** — several ship with default methods specifically for chaining:
```java
Function<Integer, Integer> addOne = n -> n + 1;
Function<Integer, Integer> square = n -> n * n;
Function<Integer, Integer> addThenSquare = addOne.andThen(square); // apply addOne, THEN square the result
Function<Integer, Integer> squareThenAdd = addOne.compose(square); // apply square FIRST, then addOne
System.out.println(addThenSquare.apply(3)); // (3+1)^2 = 16
System.out.println(squareThenAdd.apply(3)); // (3^2)+1 = 10

Predicate<String> notEmpty = ((Predicate<String>) String::isEmpty).negate();
Predicate<Integer> isPositive = n -> n > 0;
Predicate<Integer> isEven = n -> n % 2 == 0;
Predicate<Integer> positiveAndEven = isPositive.and(isEven);
```
> ⚠️ **Pitfall — `andThen()` vs `compose()` direction, a genuinely easy mix-up:** `f.andThen(g)` runs `f` **first**, then feeds its result into `g`. `f.compose(g)` runs `g` **first**, then feeds its result into `f` — the naming can feel backwards relative to reading order until you internalize that `compose` mirrors mathematical function composition notation (`f(g(x))`), while `andThen` mirrors left-to-right pipeline reading (`f`, and then, `g`).

## Effectively-Final Variable Capture

**The rule:** a lambda (or anonymous class) can only capture a local variable from its enclosing scope if that variable is **effectively final** — assigned exactly once, never reassigned after initialization, even if not explicitly marked `final`.

```java
int counter = 0;
Runnable r = () -> System.out.println(counter); // fine -- counter never reassigned, effectively final
counter = 5; // uncomment this and the lambda above becomes a COMPILE ERROR
```
```java
List<Runnable> tasks = new ArrayList<>();
for (int i = 0; i < 3; i++) {
    int taskId = i; // a NEW effectively-final variable each iteration -- this is the fix
    tasks.add(() -> System.out.println("Task " + taskId));
}
tasks.forEach(Runnable::run); // prints Task 0, Task 1, Task 2 -- each captured its own value correctly
```
**Why this restriction exists, precisely:** a lambda can genuinely **outlive** the method call that created it (it might be stored, passed to another thread, or invoked much later). The JVM handles this by having the lambda capture a **copy** of the local variable's value at creation time, not a live reference to the stack slot (which may no longer even exist by the time the lambda runs). If reassignment were allowed, the lambda's captured copy and the enclosing method's live variable would silently diverge — which value should the lambda actually see? Making the variable effectively final removes the ambiguity by construction: there's only ever one value to capture, so there's nothing to diverge.

> ⚠️ **Pitfall — the classic "closure over a mutable loop variable" trap, and why Java doesn't have it:** in some languages, closures created inside a loop all end up sharing the *same* underlying loop variable, so all of them see its *final* value after the loop ends (a well-known bug class). Java's effectively-final rule makes this specific bug **impossible** — you're forced to introduce a fresh variable per iteration (as `taskId` does above) precisely because the loop variable `i` itself is reassigned every iteration and therefore isn't effectively final, so the compiler refuses to let a lambda capture it directly.

**Capturing `this` inside a lambda vs an anonymous class — a real, different-behavior gotcha:**
```java
class Handler {
    private String name = "outer";
    void register() {
        Runnable lambdaVersion = () -> System.out.println(this.name);       // 'this' = the ENCLOSING Handler instance
        Runnable anonymousVersion = new Runnable() {
            private String name = "inner"; // shadows nothing from Runnable, just this anonymous class's own field
            public void run() { System.out.println(this.name); }            // 'this' = the ANONYMOUS CLASS instance itself
        };
    }
}
```
A lambda does **not** introduce its own `this` — it's lexically scoped, so `this` inside a lambda always refers to the enclosing instance, exactly like any other code in that method. An anonymous class **does** introduce its own `this`, referring to the anonymous class instance itself. This is a real, observable behavioral difference, not just a style preference between the two syntaxes.

---

## Interview Q&A

**Q: Is a lambda "just syntactic sugar for an anonymous class"?**
Behaviorally similar, but not how it's actually compiled. Lambdas use `invokedynamic` and `LambdaMetafactory` to generate the implementation class lazily at runtime, avoiding a separate compile-time `.class` file per lambda expression — a real difference in JAR size and classloading cost versus anonymous classes, not just a syntax difference.

**Q: What does `@FunctionalInterface` actually enforce, and do `default` methods count against it?**
It enforces exactly one abstract method, turning an accidental second abstract method into a compile error. `default` and `static` methods do not count — which is why interfaces like `Comparator<T>` (one abstract method, several default methods) are still valid functional interfaces.

**Q: `Function.andThen()` vs `Function.compose()` — what's the actual execution order for each?**
`f.andThen(g)` runs `f` first, then passes its result to `g`. `f.compose(g)` runs `g` first, then passes its result to `f` — `compose` mirrors mathematical `f(g(x))` notation, which is the opposite order from how it reads left to right.

**Q: Why must a variable captured by a lambda be effectively final?**
A lambda can outlive the method that created it, so it captures a copy of the variable's value at creation time rather than a live reference to a stack slot that might not exist later. If the variable could be reassigned after that copy was taken, the lambda's captured value and the enclosing method's current value could silently diverge — effectively-final removes that ambiguity by guaranteeing there's only ever one value to capture.

**Q: Does `this` inside a lambda refer to the same thing as `this` inside an anonymous class implementing the same interface?**
No. A lambda is lexically scoped and doesn't introduce its own `this` — it refers to the enclosing instance, same as any other code in that method. An anonymous class does introduce its own `this`, referring to the anonymous class instance itself. This is a real behavioral difference, not just style.
