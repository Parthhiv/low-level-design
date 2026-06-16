# Builder Pattern — LLD Notes (SDE 1 / FAANG)

> Creational design pattern. The answer to "this object has too many constructor parameters, half of them optional."
> One of the easiest interview wins — clean, recognizable, and useful in almost every design.

---

## 1. The one-line idea

**Construct a complex object step-by-step through a separate Builder, then produce the finished, immutable object in one `build()` call — instead of a giant constructor or a maze of setters.**

You set only the fields you care about, in any order, with readable names, and get back a fully-validated object.

---

## 2. The problem it solves (intuition)

An object has many fields, several optional:

- A `User` has name, email, phone, age, address, isVerified, …
- An HTTP request has url, method, headers, body, timeout, retries…
- A `Pizza` has size, crust, cheese, and a dozen optional toppings.

Two naive fixes, both bad:

**(a) Telescoping constructors** — one constructor per combination:
```java
// SMELL: which boolean is which? what's null for?
User(String name, String email) { ... }
User(String name, String email, String phone) { ... }
User(String name, String email, String phone, int age, String addr, boolean verified) { ... }
new User("A", "a@x.com", null, 0, null, true); // unreadable, error-prone
```

**(b) No-arg constructor + setters** — readable but leaves the object **mutable** and allows **half-built/invalid** objects to exist:
```java
User u = new User();
u.setName("A");
// forgot setEmail() — object is now in an invalid state, and mutable forever
```

Builder fixes both: readable named steps, validation in one place, and an **immutable** result.

---

## 3. Structure

```
   Client
     │  chains setters
     ▼
  ┌──────────────────────────┐        builds
  │   User.Builder           │──────────────────────┐
  │   - name, email, phone…  │                       ▼
  │   + name(..)  : Builder  │              ┌──────────────────┐
  │   + phone(..) : Builder  │              │   User           │
  │   + build()   : User ────┼─────────────▶│  (immutable;     │
  └──────────────────────────┘              │   final fields)  │
                                            └──────────────────┘
```

| Role | What it is | Example |
|------|-----------|---------|
| **Product** | The complex object being built | `User` |
| **Builder** | Collects fields, validates, builds | `User.Builder` |
| **(Director)** | *Optional* — encapsulates a fixed build recipe | `UserDirector` |

> The **Director** is part of GoF but usually omitted in interviews; the fluent inner-class Builder (Effective Java style) is what people expect. Mention the Director exists.

---

## 4. Canonical code example (Java — Effective Java style)

```java
public class User {
    // all final → immutable
    private final String name;       // required
    private final String email;      // required
    private final String phone;      // optional
    private final int    age;        // optional
    private final boolean verified;  // optional

    // private: the ONLY way to build is via Builder
    private User(Builder b) {
        this.name = b.name;
        this.email = b.email;
        this.phone = b.phone;
        this.age = b.age;
        this.verified = b.verified;
    }

    public static class Builder {
        private final String name;   // required → in Builder's constructor
        private final String email;
        private String phone = "";    // optional → sensible defaults
        private int age = 0;
        private boolean verified = false;

        public Builder(String name, String email) {   // required args up front
            this.name = name;
            this.email = email;
        }
        public Builder phone(String phone)    { this.phone = phone; return this; }     // return this → chaining
        public Builder age(int age)           { this.age = age; return this; }
        public Builder verified(boolean v)    { this.verified = v; return this; }

        public User build() {
            // validation lives in ONE place
            if (name == null || name.isBlank()) throw new IllegalStateException("name required");
            if (age < 0) throw new IllegalStateException("age must be >= 0");
            return new User(this);
        }
    }
}

// Client — readable, order-free, only the fields you want
User u = new User.Builder("Parthiv", "p@x.com")
        .phone("+91999")
        .verified(true)
        .build();
```

No telescoping constructors, no mutable half-built object, validation centralized, and `User` is immutable (thread-safe to share).

---

## 5. When to use it

- An object has **many parameters**, several **optional** — especially 4+.
- You want the result to be **immutable** but still convenient to construct.
- You want construction **validation in one place**, rejecting invalid objects before they exist.
- The same construction process should build different representations (with a Director).

## When NOT to use it

- The object has **few fields** (2–3, all required) → a plain constructor is simpler.
- All fields are genuinely required and order is obvious → constructor or a record.
- The hard part is choosing **which class** to create, not assembling fields → that's **Factory**, not Builder.

---

## 6. Real systems that use Builder (good to name-drop)

- **`StringBuilder`** — the everyday example (append step-by-step, then `toString()`).
- **`StringBuilder`/`Stream.Builder`**, `Calendar.Builder`, `Locale.Builder` in the JDK.
- **OkHttp `Request.Builder`**, **Retrofit**, **`HttpRequest.newBuilder()`** (Java 11 HTTP client) — cite this, it's modern and clean.
- **Lombok `@Builder`** — generates exactly this pattern.
- **Protocol Buffers** generated builders, **Guava** `ImmutableList.Builder`.

---

## 7. Builder vs. lookalike patterns (interviewers probe this)

| Pattern | Core difference |
|---------|-----------------|
| **Factory** | Factory decides **which class** to instantiate (one call, returns a type by intent). Builder assembles **one complex object step-by-step** (many optional fields). Hard part = *choosing type* → Factory; hard part = *construction* → Builder. |
| **Abstract Factory** | Creates **families** of products. Builder builds **one** product over multiple steps. |
| **Prototype** | Builds by **cloning** an existing instance; Builder builds fresh from parts. |
| **Fluent setters on a mutable object** | Looks similar but leaves the object mutable and possibly half-built. Builder yields an **immutable, validated** product. |

---

## 8. How it shows up in FAANG LLD rounds

Builder rarely *is* the question — it's a clean construction tool you sprinkle in:

- **Any entity with many fields** → `User`, `Order`, `Pizza`, `Vehicle`, `Resume`.
- **Request/response objects** → HTTP request builders, query builders.
- **Configuration objects** → `DatabaseConfig`, `ServerConfig`.
- **SQL/query builders** → `SELECT … WHERE … ORDER BY …` chained.
- **Meal/pizza combos** → build a fixed combo via a Director.

**Where to introduce it:** the moment you're about to write a constructor with many params, or someone asks "how do you create the `Order` object?" Say: *"It has several optional fields, so I'll use a Builder for a readable, immutable, validated construction."*

---

## 9. How to talk about it (scoring phrases)

- "It removes **telescoping constructors** and keeps the product **immutable**."
- "Required fields go in the Builder's constructor; optional ones are chained — invalid objects can't be built."
- "Validation lives in `build()` — **one place**, fail fast."
- "Same idea as `StringBuilder` or `HttpRequest.newBuilder()`."
- "Builder assembles one object; a **Factory** would pick *which* object — different jobs."

---

## 10. Common mistakes (avoid these in the room)

1. **Confusing it with Factory** — Builder = complex construction; Factory = choosing a type.
2. **Leaving the product mutable** (public setters) — defeats the immutability win; make fields `final`, constructor private.
3. **No validation in `build()`** — you've just rebuilt setters with extra steps.
4. **Forgetting `return this`** in setters — breaks the fluent chain.
5. **Using a Builder for 2 required fields** — over-engineering; a constructor is clearer.
6. **A shared/reused Builder instance** producing objects that accidentally share state — build fresh, or reset.

---

## 11. Worked problem — assembling an HTTP request object

**Prompt:** *Design an `HttpRequest` object. A request always needs a URL and method; optionally headers, a body, a timeout, and a retry count. Callers should build it readably, it must be immutable once built, and invalid requests (e.g. body on a GET) should be rejected.*

Pure **Builder** (highlighted). (A **Factory** could choose between `HttpRequest`/`HttpsRequest` subtypes, but assembling this one object is the Builder's job.)

### Step 1 — immutable product with a private constructor
```java
public class HttpRequest {
    private final String method;                 // required
    private final String url;                     // required
    private final Map<String,String> headers;     // optional
    private final String body;                    // optional
    private final int timeoutMs;                  // optional, default
    private final int retries;                    // optional, default

    private HttpRequest(Builder b) {
        this.method = b.method;
        this.url = b.url;
        this.headers = Map.copyOf(b.headers);     // defensive copy → truly immutable
        this.body = b.body;
        this.timeoutMs = b.timeoutMs;
        this.retries = b.retries;
    }
    // getters only — no setters
}
```

### Step 2 — fluent Builder with required args up front
```java
public static class Builder {
    private final String method;                              // required
    private final String url;                                 // required
    private final Map<String,String> headers = new HashMap<>(); // optional
    private String body = null;
    private int timeoutMs = 30_000;                            // sensible defaults
    private int retries = 0;

    public Builder(String method, String url) {
        this.method = method;
        this.url = url;
    }
    public Builder header(String k, String v) { headers.put(k, v); return this; }
    public Builder body(String b)             { this.body = b; return this; }
    public Builder timeoutMs(int t)           { this.timeoutMs = t; return this; }
    public Builder retries(int r)             { this.retries = r; return this; }

    public HttpRequest build() {
        // centralized validation — fail before the object exists
        if (url == null || url.isBlank()) throw new IllegalStateException("url required");
        if ("GET".equals(method) && body != null) throw new IllegalStateException("GET cannot have a body");
        if (retries < 0) throw new IllegalStateException("retries must be >= 0");
        return new HttpRequest(this);
    }
}
```

### Step 3 — build it readably
```java
HttpRequest req = new HttpRequest.Builder("POST", "https://api.x.com/orders")
        .header("Authorization", "Bearer ...")
        .header("Content-Type", "application/json")
        .body("{\"item\":\"pizza\"}")
        .timeoutMs(5_000)
        .retries(3)
        .build();
```

### 🔍 Where Builder is doing the work (the highlight)
- **Required vs optional is enforced by structure:** required fields (`method`, `url`) are in the Builder's constructor; optional ones are chained — you literally cannot forget a required field, and you only specify the optionals you want. That kills **telescoping constructors**.
- **`build()` is the single validation gate** — the GET-with-body rule lives in one place and an invalid `HttpRequest` can never come into existence (fail fast).
- The product has **only `final` fields + a private constructor**, so once built it's **immutable** — safe to cache and share across threads.
- Contrast with Factory: a Factory would answer *"HttpRequest or HttpsRequest?"*; the Builder answers *"how do I assemble this one request from many optional parts?"* — that construction focus is the whole point of the pattern.

---

## 12. 60-second recall summary

- **Problem:** many constructor params (many optional) → telescoping constructors or a mutable, half-built object.
- **Fix:** a Builder collects fields via fluent named setters, validates in `build()`, returns an immutable product.
- **Wins:** readable construction, immutability, centralized validation, fail-fast.
- **Distinguish:** Factory picks *which* class; Builder assembles *one* object step-by-step.
- **Real example to cite:** `StringBuilder`, `HttpRequest.newBuilder()`, OkHttp `Request.Builder`, Lombok `@Builder`.

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅
2. Factory / Factory Method ✅
3. Observer ✅
4. State ✅
5. Decorator ✅
6. Builder ✅ (this one) — Singleton next, then Adapter / Command / Chain of Responsibility / Facade
