# Part 2 — Garbage Collection

> How the JVM decides an object is garbage, minor vs major GC, the four core GC algorithms and why generational collection combines them, why the survivor space exists (and is split into two), the modern collector lineup (Serial/Parallel/G1/ZGC/Shenandoah), Stop-The-World and safepoints, and the four Java reference types with real code. Interview Q&A at the end.

## How the JVM Decides an Object Is Garbage

**What it is:** Garbage Collection automatically identifies and reclaims heap memory occupied by objects no longer reachable, so new objects can be allocated without the program ever calling `free()`.

**Reachability, precisely:** an object becomes eligible for GC the moment there's no reference path from any **GC Root** to it. GC Roots include: active thread stack variables, static variables, JNI references, and objects still referenced from an active method's call frame.

```java
public void test() {
    User u = new User(); // object created, reachable via local variable 'u' -> GC Root (stack)
    u = null;             // no reference path remains -> now eligible for GC
}
```
> ⚠️ **Pitfall:** "eligible for GC" does not mean "collected immediately." The GC runs on its own schedule (or when triggered by memory pressure) — an unreachable object can sit in the heap for an arbitrary amount of time before an actual collection cycle reclaims it. `System.gc()` only *suggests* a collection; the JVM is free to ignore it entirely.

## Minor GC vs Major GC

| | Minor (Young) GC | Major (Old) GC |
|---|---|---|
| Region | Young Generation (Eden + Survivor) | Old Generation |
| Triggered by | Eden fills up | Old Gen fills up, or promotion failure from Young Gen |
| Collects | Short-lived, newly created objects | Long-lived objects that survived multiple minor GCs |
| Frequency | Frequent | Infrequent |
| Pause time | Short | Longer, more expensive |
| Mechanism | Copying/scavenging live objects out of Eden | Mark → Sweep → Compact (collector-dependent) |

## The Four Core Algorithms

**1. Mark and Sweep** — walk from GC Roots, **mark** every reachable object; **sweep** removes everything unmarked.
```
Root -> A -> B -> C   (reachable, marked)
D -> E                (no path from root -> unmarked -> swept)
```
**Problem:** leaves fragmented free memory scattered between surviving objects — slow for large heaps, and large-object allocation can fail even with plenty of *total* free space if no single contiguous block is big enough.

**2. Mark-Compact** — same marking phase, but instead of just removing dead objects, it **slides all live objects to one end** of the region, then reclaims everything past that boundary as one contiguous block.
```
Before:  [A*][B][C*][D][E][F*]     (* = live)
After:   [A][C][F][ Free space  ]
```
Fixes fragmentation entirely, at the cost of extra work: every reference pointing at a moved object must be updated to its new address.

**3. Copying** — divide memory into two equal semi-spaces (**From** and **To**); allocate only in From; on GC, copy *only the live objects* into To, then swap roles.
```
From: [A][B][C][D][E][Free]   (only A, C, E are still live)
                |  copy live objects only
                v
To:   [A][C][E]
                |  swap roles
                v
Active space is now: [A][C][E][Free][Free][Free]
```
Very fast (touches only live objects, not dead ones) and produces zero fragmentation, at the cost of permanently sacrificing half the available memory to the inactive space.

**4. Generational Collection** — the practical combination: since the Young Generation reliably has a *high death rate* (most objects created are short-lived and die fast), the fast **Copying** algorithm suits it well — copying the few survivors is cheap. Since the Old Generation has a *high survival rate* (few new dead objects per cycle, no benefit from wasting half the space), **Mark-Sweep** or **Mark-Compact** suits it better.

> ⚠️ **Pitfall — the single most-tested GC design insight:** generational GC isn't "one algorithm made generational," it's a deliberate *combination* of two different algorithms, each matched to a different generation's actual object-lifetime statistics. Explaining *why* each algorithm fits its generation (not just naming which one goes where) is what separates a memorized answer from an understood one.

## Why the Survivor Space Exists — and Why It's Split in Two

**Without a survivor space:** every object surviving Eden's minor GC would go straight to the Old Generation. Many objects that survive *one* minor GC still die within the next two or three — sending them to the Old Gen immediately would fill it prematurely, triggering expensive major GCs far more often than necessary. The Survivor space acts as a probation buffer — only objects that survive several minor GC cycles (commonly tuned via `-XX:MaxTenuringThreshold`) get promoted to Old Gen.

**Why two Survivor spaces (From/To), not one:** with only one Survivor space, clearing dead objects *within* it would require Mark-Sweep on that region too — reintroducing the exact fragmentation problem generational collection exists to avoid, specifically inside the young-object-heavy region where it would hurt most. With two: each minor GC copies Eden's + the current Survivor's live objects into the *other* (empty) Survivor space, then the roles swap — the same Copying-algorithm benefit (no fragmentation) applied consistently every cycle. Splitting into three-plus spaces was considered and rejected — each partition would be small enough to fill up too fast, so two was settled on as the practical trade-off.

## Types of Garbage Collectors

```
| Collector    | Young Gen Algorithm      | Old Gen Algorithm            | Best for                              |
|--------------|--------------------------|-------------------------------|----------------------------------------|
| Serial GC    | Copying                 | Mark-Compact                  | Small apps, single-threaded, low memory|
| Parallel GC  | Copying                 | Mark-Compact                  | Throughput-focused, multi-core, default pre-Java 9 |
| CMS          | Copying                 | Mark-Sweep (concurrent)       | Low-latency (deprecated, removed Java 14)|
| G1 GC        | Copying (region-based)  | Mark-Compact (region-based)   | Balanced latency+throughput, default since Java 9 |
| ZGC          | Concurrent Mark         | Relocation (copying-like)     | Ultra-low pause, very large heaps      |
| Shenandoah   | Concurrent Mark         | Concurrent Compaction         | Ultra-low pause, alternative to ZGC    |
```
```
java -XX:+UseSerialGC   -jar app.jar
java -XX:+UseParallelGC -jar app.jar
java -XX:+UseG1GC       -jar app.jar   # default since Java 9
java -XX:+UseZGC        -jar app.jar
```
> ⚠️ **Pitfall:** CMS (`-XX:+UseConcMarkSweepGC`) is **deprecated as of Java 9 and fully removed in Java 14** — naming it as a current production option in a 2026-era interview is a dated answer. G1 is the safe modern default; ZGC/Shenandoah are the answer specifically when sub-millisecond pause times matter more than raw throughput (very large heaps, latency-sensitive trading/real-time systems).

## Stop-The-World (STW) and Safepoints

**What it is:** a JVM mechanism where **every application thread is paused** so the collector can operate on a heap guaranteed not to be changing mid-operation.

```
Application Running
        |
        v
------ STOP THE WORLD ------
| Mark objects              |
| Copy / Compact memory     |
| Update references         |
-----------------------------
        |
        v
Application Resumes
```
**Why it's required:** if application threads kept mutating references while the collector was marking or relocating objects, the heap could become inconsistent mid-collection — a moved object's old references not yet updated, or a newly-created reference to an object already swept. To guarantee correctness, the JVM brings every thread to a **safepoint** (a point where thread state is consistent and safely inspectable), freezes them, performs the GC phase, then resumes.

```java
public class StopTheWorldExample {
    public static void main(String[] args) {
        Runtime rt = Runtime.getRuntime();
        System.out.println("Free before GC: " + rt.freeMemory());
        System.gc(); // requests a GC cycle -- may trigger a real STW pause
        System.out.println("Free after GC: " + rt.freeMemory());
    }
}
```
> ⚠️ **Pitfall:** "modern collectors don't have STW pauses" is false — even fully concurrent collectors like G1 and ZGC still require **short** STW phases for specific operations (initial marking, reference processing) — they minimize and bound STW duration, they don't eliminate it. The real tuning goal is reducing STW *duration and frequency*, via heap sizing, region configuration, and collector choice — not chasing an STW-free system that doesn't exist.

## The Four Reference Types

**Strong (default)** — any normal object reference. Never eligible for GC while a strong reference exists.
```java
MyClass obj = new MyClass(); // strong reference — not collectible
obj = null;                   // now unreachable — eligible for GC
```

**Weak** — doesn't prevent collection; the referent can be collected the moment no *strong* reference remains, regardless of memory pressure.
```java
Gfg g = new Gfg();
WeakReference<Gfg> weakRef = new WeakReference<>(g);
g = null;
// g may be collected on the VERY NEXT GC cycle -- no memory pressure required
Gfg retrieved = weakRef.get(); // may return null if already collected
```
Real use: `WeakHashMap`, canonicalizing caches where entries should disappear the instant nothing else references the key.

**Soft** — like weak, but the JVM specifically tries to **keep** softly-reachable objects alive as long as possible, only clearing them **right before** it would otherwise throw `OutOfMemoryError`.
```java
Gfg g = new Gfg();
SoftReference<Gfg> softRef = new SoftReference<>(g);
g = null;
// g is very likely to survive many GC cycles -- only cleared under genuine memory pressure
```
Real use: memory-sensitive caches (image caches, computed-result caches) that should hold onto data as long as there's spare memory, but yield gracefully instead of causing an OOM.

**Phantom** — the referent is *already* effectively gone; `get()` **always** returns `null` (unlike weak/soft). Used purely for cleanup notification via a `ReferenceQueue`, after `finalize()` (or, in modern code, a `Cleaner`) has run.
```java
ReferenceQueue<Gfg> queue = new ReferenceQueue<>();
PhantomReference<Gfg> phantomRef = new PhantomReference<>(g, queue);
g = null;
Gfg retrieved = phantomRef.get(); // ALWAYS null, even before collection
```
> ⚠️ **Pitfall — the ordering that trips people up:** Strong → Soft → Weak → Phantom is the order of "how hard the GC tries to keep it alive," from "never collect while referenced" down to "already gone, this is just a collection notification." Confusing Soft and Weak specifically (both survive until *some* trigger) is the most common mix-up — the precise distinction is Weak dies on the *next* GC regardless of memory state, Soft survives until memory pressure specifically forces the JVM's hand.

---

## Interview Q&A

**Q: How does the JVM decide an object is garbage?**
No reference path exists from any GC Root (active thread stacks, static variables, JNI references) to the object. Reachability, not reference *counting*, is the mechanism — this is exactly why Java handles cyclic references correctly (two objects referencing each other, but unreachable from any root, are both still collected).

**Q: Explain why generational GC uses different algorithms for Young and Old generations.**
Young Gen has a high death rate (most objects are short-lived) — the Copying algorithm is cheap here because so few objects actually need copying. Old Gen has a high survival rate — Copying would waste half its (large) space for little benefit, so Mark-Sweep/Mark-Compact suits it better instead.

**Q: Why does the Survivor space exist, and why is it split into two?**
Without it, every minor-GC survivor would go straight to Old Gen, filling it prematurely even though many would die within the next couple of cycles anyway — the Survivor space is a probation buffer. It's split into two (From/To) so the Copying algorithm's no-fragmentation benefit can be applied every cycle by alternating roles, rather than needing Mark-Sweep (and its fragmentation) within a single Survivor region.

**Q: Is CMS still a reasonable answer for "which GC would you pick for low latency" in a modern interview?**
No — CMS is deprecated since Java 9 and removed entirely in Java 14. G1 is the modern balanced default; ZGC/Shenandoah are the current answer when sub-millisecond pauses on very large heaps genuinely matter.

**Q: Do modern low-latency collectors like ZGC eliminate Stop-The-World pauses entirely?**
No — they minimize and bound STW duration for specific short phases (initial marking, reference processing), but some STW pause still exists. "Fully STW-free" is a common misconception worth correcting directly if asked.

**Q: Soft vs Weak reference — what's the precise difference, not just "both are weaker than strong"?**
A `WeakReference`'s referent can be collected on the very next GC cycle once no strong reference remains, independent of memory pressure. A `SoftReference`'s referent is deliberately kept alive by the JVM as long as practical, only being cleared when memory pressure genuinely requires it (right before an OOM would otherwise be thrown) — making Soft the right choice for memory-sensitive caches, and Weak the right choice for canonicalizing/identity-based structures like `WeakHashMap`.
