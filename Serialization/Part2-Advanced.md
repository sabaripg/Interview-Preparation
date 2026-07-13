# Part 2 — Serialization Advanced

> Cyclic references and the object handle table, how serialization breaks Singleton (and the `readResolve()` fix), `readObject` vs `readResolve`, the `Externalizable`/`Serializable` mixed-hierarchy trick questions, and deep cloning via serialization. Interview Q&A at the end.

## Cyclic References — How Java Avoids Infinite Recursion

**The problem, unhandled:** two objects referencing each other would naively serialize forever.
```java
class A implements Serializable { B b; }
class B implements Serializable { A a; }

A a = new A(); B b = new B();
a.b = b;
b.a = a; // cyclic: A -> B -> A -> B -> ...
```
Without special handling: `A → serialize B → serialize A → serialize B → ...` → infinite recursion → `StackOverflowError`.

**How Java actually handles it — the Object Handle Table:** during serialization, the JVM maintains an in-progress map of every object already written, each assigned a unique **handle** (an integer ID) the first time it's encountered.
```
Handle Table
------------
A -> #1
B -> #2
```
When serializing `A`'s field `b` (a `B`), `B` gets handle `#2`. When serializing *that* `B`'s field `a`, the JVM recognizes `A` is **already in the handle table** — instead of serializing it again (which would recurse forever), it writes a **reference to handle #1**. On deserialization, the JVM reconstructs both objects, then wires `A.b -> B` and `B.a -> A` using the handle references, restoring the exact original cyclic graph — including preserving **object identity** (the reconstructed `A` and `B` genuinely reference each other, not two separate copies).

> ⚠️ **Pitfall:** this handle-table mechanism is *also* why deep-cloning via serialization (see below) correctly preserves shared references within an object graph — if two fields point at the *same* nested object, serialization writes it once and both fields resolve back to the *same* deserialized instance, not two independent copies. Naively hand-rolling a deep clone (recursively `new`-ing everything) loses this property unless you explicitly track visited objects yourself.

## How Serialization Breaks Singleton — and the Fix

**The break:** deserialization does **not** call any constructor for a `Serializable` class — a brand new object is materialized directly from the byte stream, bypassing every constructor, including a `private` singleton constructor.
```java
public class Singleton implements Serializable {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() { return INSTANCE; }
}

Singleton s1 = Singleton.getInstance();
Singleton s2 = deserialize(serialize(s1));
s1 == s2; // false -- Singleton contract violated, a second instance now exists
```
**The fix — `readResolve()`:**
```java
public class Singleton implements Serializable {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() { return INSTANCE; }
    private Object readResolve() { return INSTANCE; } // replaces the deserialized object with the real singleton
}
// with readResolve(): s1 == s2 -> true
```
`readResolve()` runs *after* the object is fully deserialized, and whatever it returns **replaces** the deserialized instance as far as the caller is concerned — the freshly deserialized (violating) object is discarded, and the caller receives the true singleton instead.

> ⚠️ **Pitfall:** this is precisely why `enum`-based singletons (see `core-java/Enums.md`) are recommended over this pattern in modern Java — Java's serialization mechanism special-cases enums to serialize only the constant's name and resolve it back to the same JVM constant automatically, with no `readResolve()` needed at all. The classic pattern above works, but only if you remember to write the defensive `readResolve()` yourself; the enum pattern makes the mistake impossible to make.

## `readObject()` vs `readResolve()` — Different Jobs, Easy to Conflate

**`readObject()`** — customizes **how fields are populated** during deserialization of a specific class.
```java
class Employee implements Serializable {
    int id;
    transient String password;
    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();     // restores id normally
        password = "default123";     // then custom logic for the transient field
    }
}
```
**`readResolve()`** — replaces the **entire deserialized object** with a different one (the Singleton fix above is the canonical use case).

**Execution order during deserialization:** (1) JVM creates the object and populates fields — running `readObject()` if present; (2) `readResolve()` runs afterward, if present, and can swap out the whole object; (3) the final (possibly substituted) object is returned to the caller.

## The Mixed-Hierarchy Trick Questions

**Trick 1 — parent `Serializable`, child `Externalizable`:**
```java
class Parent implements Serializable { int parentId = 10; }
class Child extends Parent implements Externalizable {
    int childId = 20;
    public Child() {} // mandatory
    public void writeExternal(ObjectOutput out) throws IOException { out.writeInt(parentId); out.writeInt(childId); }
    public void readExternal(ObjectInput in) throws IOException { parentId = in.readInt(); childId = in.readInt(); }
}
```
**The key rule:** once *any* class in the hierarchy implements `Externalizable`, the JVM stops using default `Serializable` mechanics for that object entirely — **parent fields are NOT automatically included**, even though the parent is `Serializable`, because the child's `Externalizable` status takes over the whole serialization process. The developer must manually write the parent's fields inside the child's `writeExternal()` (as shown above) — forgetting `out.writeInt(parentId)` silently loses that data, with the parent field falling back to whatever the no-arg constructor set it to.

**Constructor execution here:** the `Child` no-arg constructor **does** run (required by `Externalizable`); the `Parent` constructor does **not** run — because the JVM never falls back to `Serializable`'s "call the non-serializable ancestor's constructor" rule from Part 1, since `Externalizable` is handling everything.

**Trick 2 — parent `Externalizable`, child `Serializable` (the more dangerous direction):**
```java
class Parent implements Externalizable {
    int parentId = 10;
    public Parent() { System.out.println("Parent constructor"); }
    public void writeExternal(ObjectOutput out) throws IOException { out.writeInt(parentId); }
    public void readExternal(ObjectInput in) throws IOException { parentId = in.readInt(); }
}
class Child extends Parent implements Serializable {
    int childId = 20;
    public Child() { System.out.println("Child constructor"); }
}
```
**Externalizable takes priority over Serializable across the whole hierarchy** — since `Parent` is `Externalizable`, the JVM uses `Externalizable` semantics for the entire object, calling `Parent.writeExternal()`. **Child fields are silently NOT serialized** — nothing in `writeExternal()` writes `childId`, and there's no automatic child-field handling the way there would be if `Parent` were plain `Serializable`.

**Deserialization order:** Parent's no-arg constructor runs → Child's no-arg constructor runs → `Parent.readExternal()` populates `parentId`. `childId` is left at its field-initializer default (`20` here, from the field initializer running during construction — not from any restored data).
```
| Field    | Value after deserialization           |
|----------|----------------------------------------|
| parentId | restored from the stream               |
| childId  | whatever the constructor/initializer set it to — NOT from serialized data |
```
> ⚠️ **Pitfall — the trap in this exact scenario:** it's tempting to assume `childId` shows `0` (a "not restored" default), but it actually shows the **field initializer's value** (`20`), because the `Child` constructor still runs normally (unlike in pure `Serializable` deserialization) and its field initializers execute as part of that normal construction. The "constructor doesn't run" rule from Part 1 is specifically a `Serializable`-only rule — it does not apply here, because `Externalizable` requires full object construction *before* `readExternal()` populates state.

**The bad-but-sometimes-seen workaround** — a parent reaching into a child's fields via `instanceof`:
```java
public void writeExternal(ObjectOutput out) throws IOException {
    out.writeInt(parentId);
    if (this instanceof Child) { out.writeInt(((Child) this).childId); } // parent depending on child -- inverted, fragile design
}
```
Functionally works, but inverts the normal dependency direction (parent classes shouldn't know about specific subclasses) — a legitimate answer to "how would you fix this," paired with an explicit acknowledgment that it's a design smell, not a recommended pattern.

## Deep Cloning via Serialization

**Yes, it's a valid technique:** serialize an object graph to a byte stream, then immediately deserialize it back — producing a completely independent copy where every nested object (not just the top-level one) is newly reconstructed, because the entire graph passes through the byte stream.
```java
@SuppressWarnings("unchecked")
static <T extends Serializable> T deepClone(T original) throws IOException, ClassNotFoundException {
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    try (ObjectOutputStream oos = new ObjectOutputStream(baos)) { oos.writeObject(original); }
    try (ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(baos.toByteArray()))) {
        return (T) ois.readObject();
    }
}
```
> ⚠️ **Pitfall:** every class in the object graph must be `Serializable` (or the clone throws `NotSerializableException`), and this approach is measurably slower than hand-written field-by-field cloning for hot paths, precisely because of the reflection-heavy, general-purpose nature of the default serialization mechanism (see the Reflection notes in `core-java/Reflection.md` for why reflection-based mechanisms carry real overhead). It's a legitimate quick technique for correctness-over-performance situations, not a default cloning strategy for performance-sensitive code.

---

## Interview Q&A

**Q: How does Java's serialization avoid infinite recursion with cyclic references?**
An object handle table tracks every object already written during the current serialization pass; encountering an already-serialized object a second time writes a reference to its existing handle instead of serializing it again — this also preserves object identity across the deserialized graph.

**Q: Why does deserialization break the Singleton pattern, and how does `readResolve()` fix it?**
Deserialization builds a new object directly from the byte stream without calling any constructor, bypassing a private singleton constructor entirely and producing a second instance. `readResolve()` runs after deserialization and lets you substitute the real singleton instance in place of the newly-deserialized one before it reaches the caller.

**Q: If a Serializable child's parent is Externalizable, does the parent's constructor run during deserialization?**
Yes — this is the key exception to the usual "Serializable never calls constructors" rule. Because `Externalizable` requires the object to be fully constructed via its normal constructors *before* `readExternal()` populates state, both the parent's and child's constructors run normally, unlike pure `Serializable` deserialization.

**Q: Can you deep-clone an object using serialization, and what's the catch?**
Yes — serialize then immediately deserialize the same object; every object in the graph is freshly reconstructed, correctly preserving shared-reference structure via the handle table. The catch: every class in the graph must be `Serializable`, and the approach is noticeably slower than a hand-written clone for performance-sensitive code.
