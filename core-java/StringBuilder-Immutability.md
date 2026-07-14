# StringBuilder, StringBuffer & String Immutability

> Why `String` is immutable by design (not just "because it's a Java rule"), `StringBuilder` vs `StringBuffer`, and the real performance cost of string concatenation in loops. Interview Q&A at the end.

## Why `String` Is Immutable — the Design Reasons, Not Just "It Is"

**The shallow answer** ("Strings are immutable") is table stakes. **The reasons that actually demonstrate understanding:**

1. **String pool safety.** The JVM interns string literals to save memory — `String a = "hello"; String b = "hello";` share the *same* underlying object. If `String` were mutable, mutating `a` would silently corrupt `b` and every other reference to that pooled literal anywhere else in the JVM — a catastrophic, action-at-a-distance bug.
2. **`hashCode()` caching.** `String` caches its computed hash code the first time it's needed (`private int hash;` field, computed lazily, cached forever after). This is only safe because the string can never change — a mutable string used as a `HashMap` key could silently move to the wrong bucket after being mutated, becoming permanently unfindable via `get()` even though `containsKey()` on the original reference might still (confusingly) work depending on timing.
3. **Security.** Strings are used pervasively for security-sensitive values — file paths, class names in `Class.forName()`, database connection URLs, network hostnames. If `String` were mutable, code could pass a `String` to a security check, then mutate it *after* the check but *before* it's actually used — a classic time-of-check-to-time-of-use (TOCTOU) vulnerability. Immutability closes this class of attack structurally.
4. **Thread-safety by default.** An immutable object can be freely shared across threads with zero synchronization — there's no mutable state that could ever be concurrently modified, so there's nothing to race on.

```java
String s1 = "hello";
String s2 = "hello";
System.out.println(s1 == s2); // true -- same pooled object

String s3 = s1.concat(" world"); // does NOT mutate s1 -- returns a brand-new String
System.out.println(s1);          // still "hello"
System.out.println(s3);          // "hello world"
```
> ⚠️ **Pitfall:** every `String` method that looks like it "modifies" a string (`concat()`, `replace()`, `toUpperCase()`, `trim()`, `substring()`) actually **returns a new `String` object**, leaving the original completely untouched. Calling `s.trim();` on its own line without reassigning (`s = s.trim();`) is a genuinely common bug — the trimmed result is silently discarded, and `s` is unchanged.

## `StringBuilder` vs `StringBuffer` — Mutability With a Synchronization Choice

Both are **mutable** sequences of characters, existing specifically to make repeated string modification efficient without generating a new object on every single change.

```
| | StringBuilder | StringBuffer |
|---|---|---|
| Mutable | Yes | Yes |
| Thread-safe | No | Yes -- methods are synchronized |
| Performance | Faster (no locking overhead) | Slower (locking overhead, even single-threaded) |
| Introduced | Java 5 | Java 1.0 |
| When to use | Almost always -- single-threaded or externally-synchronized use | Only when multiple threads genuinely mutate the SAME instance concurrently |
```
```java
StringBuilder sb = new StringBuilder();
sb.append("Hello").append(", ").append("World").append("!");
System.out.println(sb.toString()); // "Hello, World!"
sb.insert(0, ">> ");
sb.reverse();
```
> ⚠️ **Pitfall — `StringBuffer` is rarely the right answer today, and knowing why matters more than picking it correctly:** `StringBuffer` predates `java.util.concurrent` entirely (Java 1.0) — its per-method `synchronized` locking protects against corruption from concurrent access to *the same instance*, but that's a narrow, unusual scenario (most `StringBuilder`/`StringBuffer` usage is local to one method, one thread, building up a result no other thread ever touches). Reaching for `StringBuffer` "to be safe" when the instance is never actually shared across threads pays real, unnecessary synchronization overhead for a guarantee nothing needs. Default to `StringBuilder`; only use `StringBuffer` if you can articulate the specific concurrent-shared-instance scenario that requires it — and even then, consider whether building the string differently (each thread builds its own, results merged once) avoids needing shared mutable state at all.

## String Concatenation Performance — the Loop Trap

```java
// BAD -- creates a NEW String object on every iteration
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i + ","; // desugars to: result = new StringBuilder().append(result).append(i).append(",").toString();
}
```
```java
// GOOD -- one mutable buffer, no intermediate String garbage
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i).append(",");
}
String result = sb.toString();
```
**Why the first version is genuinely expensive, precisely:** the compiler actually *does* rewrite `result += ...` into a `StringBuilder` internally per statement — but that's exactly the problem: it creates a **brand-new** `StringBuilder` on **every single iteration**, appends into it, then immediately converts it to a `String` and discards the builder. Across 10,000 iterations, that's 10,000 short-lived `StringBuilder` objects and 10,000 intermediate `String` objects, each holding a progressively longer copy of the accumulating text — quadratic-ish memory churn, not the linear cost the "just append" mental model suggests.

> ⚠️ **Pitfall — this specific compiler behavior is easy to over-generalize:** the JVM/compiler's implicit `StringBuilder` rewrite for a **single** `+=`/`+` expression (not in a loop) is genuinely fine and idiomatic — `String msg = "User " + userId + " logged in";` is not a performance problem, it's one `StringBuilder`, used once. The problem is specifically **repeated concatenation across loop iterations**, where each iteration pays the new-builder-plus-new-string cost again. Know the distinction: one-shot concatenation, fine; loop-accumulated concatenation, use an explicit `StringBuilder` declared *outside* the loop.

---

## Interview Q&A

**Q: Why is `String` immutable — give the real design reasons, not just "it is."**
Four reasons: string-pool safety (a mutable pooled literal would corrupt every reference sharing it), safe `hashCode()` caching (a mutable string used as a `HashMap` key could silently become unfindable), security (prevents time-of-check-to-time-of-use bugs on security-sensitive string values like file paths or class names), and free thread-safety (an immutable object needs no synchronization to share across threads).

**Q: `StringBuilder` vs `StringBuffer` — what's the actual difference, and which should you default to?**
`StringBuffer`'s methods are `synchronized`, `StringBuilder`'s are not — otherwise functionally identical. Default to `StringBuilder`; only use `StringBuffer` when multiple threads genuinely mutate the *same* instance concurrently, which is uncommon, since most builder usage is local to one method on one thread.

**Q: Why is `result += x` inside a loop a performance problem, if the compiler already turns `+=` into a `StringBuilder` internally?**
Because the compiler creates a **new** `StringBuilder` on every iteration of the loop, appends once, converts to `String`, and discards it — not one builder reused across the whole loop. A single `+=`/`+` expression outside a loop is fine (one builder, used once); the cost is specifically from repeating that pattern across iterations, where an explicitly declared `StringBuilder` outside the loop avoids the repeated builder/String churn entirely.

**Q: Does calling `s.trim()` or `s.replace(...)` modify the original `String`?**
No — every apparent "mutating" method on `String` (`trim`, `replace`, `concat`, `toUpperCase`, `substring`, etc.) returns a brand-new `String` and leaves the original completely unchanged. Calling one of these without reassigning the result (`s = s.trim();`) is a common bug where the transformed value is silently discarded.
