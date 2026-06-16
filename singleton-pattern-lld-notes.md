# Singleton Pattern — LLD Notes (SDE 1 / FAANG)

> Creational design pattern. "Exactly one instance, with a global access point."
> Easy to state, but interviewers grill the **thread-safety** details — that's where the points are.

---

## 1. The one-line idea

**Ensure a class has only one instance and provide a single global access point to it.**

The class controls its own instantiation (private constructor) and hands everyone the same object.

---

## 2. The problem it solves (intuition)

Some things should exist **once** in the whole app:

- A configuration registry.
- A logger.
- A database connection pool / thread pool.
- An in-memory cache.

If everyone `new`s their own, you get inconsistent config, multiple pools fighting for connections, wasted memory, and no single source of truth. Singleton guarantees one shared instance.

```java
// SMELL: every caller makes its own → duplicated state, wasted resources
ConfigManager c1 = new ConfigManager(); // reads config file
ConfigManager c2 = new ConfigManager(); // reads it AGAIN, may differ
```

> ⚠️ Caveat to say out loud: Singleton is the **most controversial** pattern. It's effectively global state, which hurts testability and hides dependencies. Use it for genuinely single resources; otherwise prefer **dependency injection** of a single instance. Saying this shows maturity.

---

## 3. Structure

```
   ┌─────────────────────────────┐
   │        Singleton            │
   │  - static instance          │◀─── the one and only
   │  - private Singleton()      │     (constructor is private)
   │  + static getInstance() ────┼──▶ returns the same instance every time
   └─────────────────────────────┘
```

| Role | What it is |
|------|-----------|
| **private constructor** | Stops anyone else from `new`-ing it |
| **static instance field** | Holds the single instance |
| **static `getInstance()`** | Global access point, creates-or-returns the instance |

---

## 4. Canonical code examples (Java) — know the trade-offs

### (a) Eager initialization — simplest, thread-safe by classloading
```java
public class Config {
    private static final Config INSTANCE = new Config();  // created at class load
    private Config() { /* load config */ }
    public static Config getInstance() { return INSTANCE; }
}
```
✅ Thread-safe for free. ❌ Created even if never used (fine for cheap objects).

### (b) Lazy + double-checked locking — lazy & thread-safe
```java
public class Config {
    private static volatile Config instance;   // volatile is REQUIRED here
    private Config() { }
    public static Config getInstance() {
        if (instance == null) {                 // 1st check (no lock, fast path)
            synchronized (Config.class) {
                if (instance == null) {         // 2nd check (locked)
                    instance = new Config();
                }
            }
        }
        return instance;
    }
}
```
✅ Lazy + thread-safe. ⚠️ `volatile` is essential — without it, instruction reordering can hand out a half-constructed object. This is the #1 detail interviewers want.

### (c) Bill Pugh / static holder — the clean idiom (recommended)
```java
public class Config {
    private Config() { }
    private static class Holder {               // loaded only on first access
        private static final Config INSTANCE = new Config();
    }
    public static Config getInstance() { return Holder.INSTANCE; }
}
```
✅ Lazy **and** thread-safe **without** synchronization, using the JVM's class-init guarantees. Cleanest of the lot.

### (d) Enum singleton — most robust (Effective Java's pick)
```java
public enum Config {
    INSTANCE;
    public void load() { /* ... */ }
}
// usage: Config.INSTANCE.load();
```
✅ Thread-safe, and the **only** form immune to reflection and serialization attacks. ❌ Less flexible (can't lazy-init with args, can't extend).

---

## 5. When to use it

- Exactly **one** instance must exist (shared resource: config, logger, pool, cache).
- That instance needs a well-known **global access point**.
- The object is **stateless or safely shared** across the app.

## When NOT to use it

- You're using it just to avoid passing an object around → that's **global state**; prefer **dependency injection**.
- The object has **mutable per-request state** → a singleton shared across threads becomes a bug magnet.
- It makes **unit testing hard** (can't substitute a mock) — a big reason many teams avoid it.
- You might need **more than one** later (e.g. multiple caches) — singletons are painful to un-singleton.

---

## 6. Real systems that use Singleton (good to name-drop)

- **`java.lang.Runtime.getRuntime()`** — textbook JDK singleton.
- **`Logger`** in most logging frameworks (often effectively singletons).
- **Spring beans** — default scope is **singleton** (one per container) — note Spring manages it *for* you via DI, sidestepping the testability problem.
- **`DriverManager`**, `Desktop.getDesktop()`.
- **Connection pools / thread pools**, app-wide caches.

Saying "Spring gives you singleton-scoped beans via DI, which keeps them testable unlike a hand-rolled singleton" is a strong, modern point.

---

## 7. Singleton vs. lookalike concepts (interviewers probe this)

| Concept | Difference |
|---------|-----------|
| **Static class / static methods** | A utility class of statics has no instance, no state, can't implement interfaces or be passed around. Singleton is a real **object** (can implement interfaces, be injected). |
| **Factory** | Often combined — the factory may be a singleton. Different intent: Singleton controls *how many* instances; Factory controls *which class*. |
| **Monostate** | Alternative where all instances share static state (looks normal, acts single). Rarely asked but good to know. |
| **Dependency Injection** | The modern alternative: create one instance and inject it, getting "one instance" without global access — keeps testability. |

---

## 8. How it shows up in FAANG LLD rounds

Singleton is usually a **supporting** element, but the thread-safety question is a favorite:

- **Logger / Config manager** → the standard examples.
- **Connection pool / thread pool** in any system design.
- **Cache** in front of a service.
- **ID generator** (careful: must be thread-safe).
- **Game/board manager**, **device driver**.

Common follow-ups they'll fire at you: *"Is your `getInstance()` thread-safe? Why `volatile`? How does double-checked locking work? How would you break a singleton with reflection/serialization, and how does the enum prevent it?"* Have answers ready.

**Where to introduce it:** when a resource is logically single. Say: *"The config registry should be a single shared instance; I'll make it a singleton — Bill Pugh holder for lazy, thread-safe init — or inject one instance via DI for testability."*

---

## 9. How to talk about it (scoring phrases)

- "Private constructor + static accessor guarantees a **single instance**."
- "I'll use the **static holder idiom** — lazy and thread-safe without locking."
- "Double-checked locking needs **`volatile`** to avoid handing out a partially constructed object."
- "The **enum** singleton is immune to reflection and serialization attacks."
- "It's global state, so I'd consider **DI** of a single instance to keep it testable."

---

## 10. Common mistakes (avoid these in the room)

1. **Naive lazy init without synchronization** — two threads can create two instances. The classic bug.
2. **Double-checked locking without `volatile`** — subtly broken via reordering.
3. **Ignoring reflection** — `setAccessible(true)` can call the private constructor; guard it or use enum.
4. **Ignoring serialization** — deserializing creates a second instance; implement `readResolve()` or use enum.
5. **Singletons holding mutable state shared across threads** without synchronization → race conditions.
6. **Overusing it** as a dumping ground for globals — hurts testability and hides dependencies.
7. **Forgetting cloning** — override `clone()` to throw, or it can duplicate the instance.

---

## 11. Worked problem — an application-wide logger

**Prompt:** *Design a `Logger` used everywhere in the app. There must be exactly one instance (shared file handle / buffer), it must be safe to call from many threads, and it should be created lazily.*

Singleton is the highlighted pattern. (A logger often also uses **Strategy** for log format and **Observer** to fan out to multiple appenders — but the "exactly one" guarantee is Singleton's job.)

### Step 1 — lock down instantiation (Bill Pugh holder = lazy + thread-safe)
```java
public class Logger {
    private Logger() { /* open file handle, init buffer */ }

    private static class Holder {                   // loaded on first getInstance()
        private static final Logger INSTANCE = new Logger();
    }
    public static Logger getInstance() { return Holder.INSTANCE; }

    // ... logging methods below
}
```

### Step 2 — make the shared instance safe under concurrency
```java
    public synchronized void log(Level level, String msg) {  // guard the shared resource
        String line = "[" + level + "] " + System.currentTimeMillis() + " " + msg;
        // write to the single shared file/stream
        System.out.println(line);
    }
}
enum Level { INFO, WARN, ERROR }
```

### Step 3 — harden against the two classic attacks (mention or implement)
```java
    // reflection guard: refuse a second construction
    private Logger() {
        if (Holder.INSTANCE != null) throw new IllegalStateException("Use getInstance()");
    }
    // serialization guard: return the existing instance instead of a new one
    protected Object readResolve() { return Holder.INSTANCE; }
```
*(Or sidestep both entirely with `enum Logger { INSTANCE; ... }`.)*

### Step 4 — use it anywhere
```java
Logger.getInstance().log(Level.INFO, "Order placed");
Logger.getInstance().log(Level.ERROR, "Payment failed");   // same instance, same file
```

### 🔍 Where Singleton is doing the work (the highlight)
- The **private constructor** makes `new Logger()` impossible from outside, so the class itself controls how many instances exist — that control *is* the Singleton pattern.
- The **static holder** gives lazy initialization (the file handle opens only on first `getInstance()`) and thread-safe publication **for free** via the JVM's class-init lock — no `synchronized` in the hot path.
- Every `Logger.getInstance()` call across the codebase returns the **same object**, so all log lines flow through one shared buffer/file — the single source of truth we needed.
- The reflection/serialization guards (or the enum form) close the loopholes that would otherwise let a "singleton" quietly become two — exactly the follow-up interviewers ask about.
- If asked to add formats or multiple outputs, that's where **Strategy**/**Observer** would layer on top — but they don't change the fact that there's *one* Logger.

---

## 12. 60-second recall summary

- **Problem:** a resource must exist exactly once with global access (config, logger, pool).
- **Fix:** private constructor + static accessor returning one shared instance.
- **Thread-safe options:** eager, double-checked locking (needs `volatile`), **static holder (best)**, **enum (most robust)**.
- **Watch out:** reflection & serialization can break it; mutable shared state needs synchronization.
- **Maturity point:** it's global state — prefer **DI of a single instance** for testability.
- **Real example to cite:** `Runtime.getRuntime()`, Spring singleton-scoped beans.

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅
2. Factory / Factory Method ✅
3. Observer ✅
4. State ✅
5. Decorator ✅
6. Builder ✅
7. Singleton ✅ (this one) — then Adapter / Command / Chain of Responsibility / Facade
