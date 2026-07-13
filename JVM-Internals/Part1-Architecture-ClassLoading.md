# Part 1 — JVM Architecture & Class Loading

> JDK vs JRE vs JVM, bytecode and real platform independence, the four JVM architecture components, runtime data areas, class loading phases, the classloader delegation model and custom classloaders, JIT compilation, and JVM tuning/monitoring. Interview Q&A at the end.

## JDK vs JRE vs JVM

- **JDK** — compiler (`javac`), debugger, and the full set of development tools + libraries.
- **JRE** — a subset of the JDK: only what's needed to *run* compiled Java, no compiler.
- **JVM** — the actual runtime engine inside the JRE that executes bytecode, manages memory, and runs garbage collection.

> ⚠️ **Pitfall:** "JVM is platform-independent" is imprecise and a common interview trap. The **JVM itself is platform-specific** — there's a distinct Windows JVM, Linux JVM, macOS JVM binary. What's actually platform-independent is the **bytecode** (`.class` files) — the same bytecode runs unmodified on any OS that has a compatible JVM installed. Platform independence is achieved *through* the JVM acting as a translation layer, not by the JVM somehow transcending platforms itself.

## Bytecode and Execution

`javac` compiles source into platform-independent bytecode. The JVM loads and verifies it, then executes it two ways simultaneously: an **interpreter** runs it line-by-line immediately (fast startup, slower steady-state), while the **JIT compiler** watches for "hot" methods executed repeatedly and compiles *those specific methods* into native machine code, caching the result so future calls skip interpretation entirely.

## The Four JVM Architecture Components

```
┌─────────────────────────────────────────────┐
│              Class Loader Subsystem          │  -- loads .class files into memory
├─────────────────────────────────────────────┤
│              Runtime Data Areas              │  -- Heap, Stack, Method Area, PC Register, Native Stack
├─────────────────────────────────────────────┤
│              Execution Engine                │  -- Interpreter + JIT Compiler
├─────────────────────────────────────────────┤
│              Garbage Collector                │  -- reclaims unreachable heap memory (Part 2)
└─────────────────────────────────────────────┘
```

## Runtime Data Areas — Heap vs Stack, Precisely

| | Heap | Stack |
|---|---|---|
| Stores | Objects, class instances | Method call frames, local variables, references |
| Shared across threads? | **Yes** — one heap for the whole JVM | **No** — one stack **per thread** |
| Managed by | Garbage Collector | LIFO push/pop on method call/return |
| Speed | Slower (GC overhead, fragmentation) | Faster (simple push/pop) |
| Size | Large, configurable (`-Xms`/`-Xmx`) | Smaller, per-thread (`-Xss`) |

**Other runtime areas**, less commonly asked but worth naming precisely: the **Method Area** (Metaspace since Java 8, replacing PermGen) stores class metadata, method bytecode, and the runtime constant pool. The **PC (Program Counter) Register** is per-thread, tracking the address of the currently executing instruction — necessary because each thread can be at a different point in execution. The **Native Method Stack** supports native (JNI, C/C++) method calls, separate from the Java stack.

**What happens when you write `new Employee()`:**
1. `new` allocates memory for the object **on the heap**.
2. The JVM initializes fields (default values, then constructor logic) and assigns a memory address.
3. The **reference** to that heap object is stored on the **stack** (a local variable or field slot).
4. Once no reference path from a GC root reaches the object, it becomes eligible for garbage collection (Part 2).

> ⚠️ **Pitfall:** the object itself is never "on the stack" — only the reference to it is, and only for local variables. This is the precise mechanical answer behind "why do stack overflows and heap OOMs have completely different causes" — a `StackOverflowError` is unbounded recursion filling per-thread stack frames; a heap OOM is too many live *objects*, an entirely separate memory region.

## Class Loading — Phases and Loaders

**Three phases, in order:**
1. **Loading** — the class file is located and read into memory.
2. **Linking** — verification (bytecode safety checks), preparation (default values for static fields), and (optionally) resolution of symbolic references.
3. **Initialization** — static variables get their actual assigned values, and static initializer blocks run, in declaration order.

**Three standard classloaders, forming a hierarchy:**
```
Bootstrap ClassLoader   (loads java.lang.*, core JDK classes — written in native code, has no Java parent)
        │
Extension/Platform ClassLoader   (loads extension libraries)
        │
Application ClassLoader   (loads YOUR classes, from the classpath)
```

**The delegation model:** when a class is requested, the request is delegated **up** to the parent first — a child ClassLoader only actually searches itself if every ancestor above it fails to find the class. This prevents duplicate class definitions and stops application code from accidentally shadowing core JDK classes (you cannot define your own `java.lang.String` and have it silently replace the real one, because the request for `java.lang.String` always resolves via the Bootstrap loader first, before your Application loader ever gets a chance).

**A minimal custom ClassLoader**, overriding `findClass()` — the pattern behind plugin systems and hot-reload frameworks:
```java
public class FileSystemClassLoader extends ClassLoader {
    private final String rootDir;
    public FileSystemClassLoader(String rootDir) { this.rootDir = rootDir; }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] bytes = Files.readAllBytes(Path.of(rootDir, name.replace('.', '/') + ".class"));
            return defineClass(name, bytes, 0, bytes.length); // hands raw bytecode bytes to the JVM
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}
```
> ⚠️ **Pitfall:** overriding `loadClass()` directly (rather than `findClass()`) is how you'd *break* the delegation model on purpose — some frameworks (Tomcat's per-webapp classloaders, OSGi) deliberately do this for isolation, but it's an unusual, deliberate override, not the default extension point.

**`ClassNotFoundException` vs `NoClassDefFoundError`:** covered in full in `core-java/Exception-Handling.md` — briefly, `ClassNotFoundException` is checked, thrown by explicit dynamic-load APIs (`Class.forName()`); `NoClassDefFoundError` is a JVM-thrown `Error` when a class available at compile time is missing at runtime during implicit resolution — and a failed static initializer permanently converts every future access of that class into a `NoClassDefFoundError`.

## The Execution Engine — Interpreter and JIT

**Interpreter** — executes bytecode instruction-by-instruction. Fast to start (no compilation delay), slow for code that runs millions of times (re-interprets every time).

**JIT (Just-In-Time) Compiler** — detects "hotspot" methods (hence "HotSpot JVM") via execution counters, then compiles *those specific methods* to native machine code, applying real optimizations: **method inlining** (replacing a call with the callee's body to remove call overhead), **loop unrolling**, **dead code elimination**, **constant folding**.

```
C1 (Client Compiler)  — optimizes fast, less aggressively; good for quick startup, short-lived processes
C2 (Server Compiler)  — optimizes slower, far more aggressively; good for long-running, throughput-sensitive services
-XX:+TieredCompilation — uses BOTH: C1 first for fast warm-up, then re-compiles genuinely hot methods with C2
```
**JIT vs AOT (Ahead-of-Time):** JIT compiles during execution using real runtime profiling data, giving excellent peak performance for long-running processes at the cost of a warm-up period. AOT compiles to native code *before* execution (GraalVM native-image is the modern example) — faster startup and lower memory, but no runtime-profile-driven optimization, which is why AOT suits serverless/containerized short-lived workloads and JIT suits long-running application servers.

> ⚠️ **Pitfall:** "JIT makes Java as fast as C++" oversells it — JIT closes a large part of the interpreted-vs-compiled gap for hot paths specifically, after a warm-up period; it does nothing for code paths that only run once or twice, and the warm-up itself is a real cost for short-lived processes (a classic pain point AOT/GraalVM specifically targets).

## Tuning and Monitoring

**Common flags:**
```
-Xms512m               initial heap size
-Xmx2g                 maximum heap size
-XX:NewRatio=2         ratio between young and old generation
-XX:+UseG1GC           select the G1 garbage collector (Part 2)
-XX:+PrintGCDetails    verbose GC logging
-XX:+HeapDumpOnOutOfMemoryError   auto-capture a heap dump on OOM (see Exception-Handling.md)
```
**Monitoring tools:** `jconsole`/`VisualVM` for real-time heap/CPU/GC visibility, **Java Flight Recorder (JFR)** + **Java Mission Control (JMC)** for low-overhead production profiling, `jstack`/`jcmd Thread.print` for thread dumps, `jmap -dump` for heap dumps.

> ⚠️ **Pitfall — the most common tuning mistake:** treating `-Xmx` in isolation without considering the deployment environment. In containers/Kubernetes specifically, `-Xmx` must sit comfortably **below** the container's memory limit (roughly 75% is a common rule of thumb) — the JVM needs headroom beyond the heap for Metaspace, thread stacks, and native buffers, or the container gets OOMKilled by the OS/cgroup regardless of what the JVM's own heap usage looks like. Full detail in `core-java/Exception-Handling.md`.

---

## Interview Q&A

**Q: Is the JVM platform-independent?**
No — the JVM binary itself is platform-specific (a Windows JVM, a Linux JVM, etc.). What's platform-independent is the *bytecode* it executes; the JVM is the translation layer that makes the same bytecode runnable everywhere.

**Q: What's the difference between Heap and Stack memory, precisely?**
Heap holds objects and is shared JVM-wide, managed by the garbage collector; Stack holds method call frames/local variables/references, is per-thread, and follows simple LIFO push/pop — no GC involvement at all.

**Q: Explain the classloader delegation model, and why it matters.**
A load request is delegated to the parent classloader first; a child only searches itself if every ancestor fails. This prevents application code from silently shadowing core JDK classes and avoids duplicate class definitions across the hierarchy.

**Q: JIT vs AOT compilation — when would you prefer each?**
JIT compiles hot methods at runtime using real execution profiling, giving excellent peak throughput for long-running processes after a warm-up period. AOT compiles before execution for fast startup and low memory, at the cost of missing runtime-profile-driven optimizations — better suited to short-lived, serverless, or fast-cold-start workloads.

**Q: A container running your Spring Boot app keeps getting OOMKilled even though your JVM never logs a heap OOM — why?**
`-Xmx` was likely set too close to (or above) the container's memory limit. The JVM's heap isn't the only memory it uses — Metaspace, thread stacks, and native buffers all sit outside `-Xmx` — so the OS/cgroup kills the container for total memory usage, a distinct failure mode from a Java-level `OutOfMemoryError`. Fix: lower `-Xmx` to leave real headroom, and confirm `-XX:+UseContainerSupport` is active.
