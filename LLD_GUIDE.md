# Low Level Design (LLD) Master Learning Guide — C# / .NET Edition

> **Target:** FAANG / MAANG SDE-1 LLD round readiness  
> **Language:** C# (.NET 10 LTS, C# 14)  
> **Source:** Merged from Head First Design Patterns, Refactoring.Guru, GoF catalog, and real .NET interview patterns  
> **Existing notes:** 11 Java-pattern files in this folder — this guide supersedes them with C# idioms and the correct learning order

---

## 📦 What's in This Repo Already

| File | Pattern | Category |
|------|---------|----------|
| `singleton-pattern-lld-notes.md` | Singleton | Creational |
| `factory-pattern-lld-notes.md` | Factory Method | Creational |
| `builder-pattern-lld-notes.md` | Builder | Creational |
| `adapter-pattern-lld-notes.md` | Adapter | Structural |
| `decorator-pattern-lld-notes.md` | Decorator | Structural |
| `facade-pattern-lld-notes.md` | Facade | Structural |
| `observer-pattern-lld-notes.md` | Observer | Behavioral |
| `state-pattern-lld-notes.md` | State | Behavioral |
| `strategy-pattern-lld-notes.md` | Strategy | Behavioral |
| `command-pattern-lld-notes.md` | Command | Behavioral |
| `chain-of-responsibility-pattern-lld-notes.md` | Chain of Responsibility | Behavioral |

> **All 11 are Java.** Use this guide as the C# replacement + the missing 14 patterns.

---

## 🛠 Toolchain (Pin This)

- **Runtime:** .NET 10 (LTS, Nov 2025 → supported to **Nov 14, 2028**)  
  *Do not start new repos on .NET 8 or 9 — both EOL **Nov 10, 2026**.*
- **Language:** C# 14 (collection expressions, `field` keyword, extension members)
- **IDE:** Visual Studio 2026, or VS Code + C# Dev Kit, or JetBrains Rider
- **Scaffold per pattern:**
  ```bash
  dotnet new console -n Patterns.Strategy -o src/Patterns.Strategy
  dotnet new xunit   -n Patterns.Strategy.Tests -o tests/Patterns.Strategy.Tests
  dotnet new sln && dotnet sln add **/*.csproj
  ```
- **Write `async`/`await` from Day 1.** .NET LLD follow-ups are about `Task`, `Channel<T>`, `SemaphoreSlim`, not `synchronized`.

---

## 🔄 Java → C# Translation Cheat Sheet

| Java | C# | Gotcha |
|------|-----|--------|
| `synchronized (x)` | `lock (privateReadonlyObject)` | Never lock `this`, `string`, or boxed value |
| `wait()` / `notify()` | `Monitor.Wait` / `Monitor.Pulse` | |
| `static final` | `const` (compile-time) vs `static readonly` (runtime) | Interview favorite |
| `String.equals(a,b)` | `a == b` for `string`, `string.Equals(a,b)` | C# `==` on `string` **is value equality** |
| `equals`/`hashCode` | `Equals`/`GetHashCode` override, or use `record` | |
| `volatile` | `volatile` (not allowed on `long`/`double`) | Prefer `Lazy<T>` |
| `ConcurrentHashMap` | `ConcurrentDictionary` | |
| `ArrayBlockingQueue` | **`Channel<T>.CreateBounded(...)`** | The .NET producer-consumer primitive |
| `Semaphore` | `SemaphoreSlim` (+ `WaitAsync`) | |
| getters/setters | Properties `{ get; set; }`, `init`, `required` | |
| — | `record` + `with` expressions | Replaces Builder + Prototype for value types |
| — | Pattern matching, `switch` expressions | Weakens Visitor and State |
| — | LINQ | Strategy + Iterator in the BCL |
| — | Nullable reference types `string?` | Weakens Null Object |
| — | `Span<T>`, `Memory<T>`, `ArrayPool<T>` | Essential for Flyweight / alloc-pressure talks |
| Annotations `@Foo` | Attributes `[Foo]` | ASP.NET filters = Command/Proxy territory |

---

## 🧭 The C#-Specific Learning Order (Why It Differs)

**Key insight:** C# absorbs several patterns into the language. A strong candidate *knows the language feature first, then the pattern*, and can argue when the language feature is better.

| Pattern | vs. Java Order | Reason |
|---------|----------------|--------|
| **Iterator** | #20 → **Week 1, Stage 0** | `yield return`, `IEnumerable<T>`, `IAsyncEnumerable<T>` + `await foreach` are core C# skills |
| **Strategy** | → **#1** | Usually a delegate: `Func<Order, decimal>`; also `IComparer<T>` |
| **Observer** | → **#2** | `IObservable<T>/IObserver<T>` in BCL; `event` is a first-class language feature |
| **Chain of Responsibility** | #18 → **#14** | ASP.NET Core middleware pipeline is a literal CoR |
| **Proxy** | #13 → **#13 (stays high)** | EF Core lazy-loading proxies, gRPC stubs, Moq/Castle, `DispatchProxy` |
| **Builder** | #7 → **#9** | `record` + `init` + `required` + `with` covers most cases |
| **Prototype** | #10 → **#11, understand-only** | `record` + `with` + `[.. items]` spread replaced it |
| **Null Object** | #21 → **#19** | NRT, `Array.Empty<T>()`, `??`/`??=`/`?.`, `TryGetValue` solve most cases |
| **Template Method** | #19 → **#21** | Modern C#: compose `Func<>` pipelines, not inheritance |
| **Visitor** | #24 → **#24** | Pattern matching + `switch` expressions largely replace double dispatch |
| **State** | → **#3** | Must teach counterpoint: `enum` + `switch` expression often better |
| **Singleton** | → **#10** | Must teach: **don't hand-roll** — use DI `AddSingleton`, `Lazy<T>`, `IHttpClientFactory` |

---

## 📅 Final C# Order (6 Weeks @ ~1.5h/day)

### Stage 0 — C# Idioms That Replace Patterns (Week 1, Days 1–2)
SOLID/GRASP · LINQ-as-Strategy · `yield return` · Events & delegates · Pattern matching · `record`/`with` · Nullable reference types · DI lifetimes · `async`/`await` · `IDisposable`/`IAsyncDisposable`

### Stage A — Varying Behavior, Broadcast, Lifecycle (Week 1, Days 3–5)
1. **Strategy**  
2. **Observer**  
3. **State**  
4. **Factory Method**  
5. **Simple Factory** (non-GoF, but ubiquitous)

### Stage B — Wrapping & Construction (Week 2)
6. **Decorator**  
7. **Adapter**  
8. **Facade**  
9. **Builder**  
10. **Singleton**  
11. **Abstract Factory**  
12. **Prototype**

### Stage C — Structural & Rarer Behavioral (Week 3)
13. **Proxy**  
14. **Chain of Responsibility**  
15. **Command**  
16. **Iterator** (pattern: custom traversal orders)  
17. **Flyweight**  
18. **Composite**  
19. **Bridge**  
20. **Null Object**

### Stage D — Rarely Asked (Week 4)
21. **Template Method**  
22. **Memento**  
23. **Mediator**  
24. **Visitor**  
25. **Interpreter**

### Stage E — Non-GoF, Higher ROI Than Most GoF (Week 4–5)
- **Dependency Injection / IoC** ★★★★★
- **Repository + Unit of Work** (EF Core) ★★★★★
- **Publisher–Subscriber & Message Bus** (MassTransit, NServiceBus, MediatR) ★★★★★
- **TPL Dataflow Pipelines** ★★★★☆
- **BackgroundService + Channel<T>** ★★★★☆
- **Service Layer** ★★★★☆
- **Object Pool** ★★★☆☆
- **Front Controller / MVC** ★★☆☆☆

---

## 💎 The 8 C# Forms That Are Non-Obvious from Java

### 1. STRATEGY — Usually a Delegate, Not a Class
```csharp
public sealed record Order(decimal Base, decimal Distance, TimeSpan Time);

public sealed class UberFare {
    private readonly Func<Order, decimal> _mode;
    public UberFare(Func<Order, decimal> mode) => _mode = mode;
    public decimal Quote(Order o) => _mode(o);

    public static UberFare Surge(decimal x) => new(o => o.Base * x);
    public static UberFare Night()       => new(o => o.Base + 150);
}

// BCL equivalents: IComparer<T>, Comparison<T>, Func<T,bool> in LINQ, IFormatProvider
```

### 2. OBSERVER — BCL Contract + Events
```csharp
// .NET's built-in Observer contract
public sealed class Stock : IObservable<decimal> {
    private readonly List<IObserver<decimal>> _obs = [];
    public IDisposable Subscribe(IObserver<decimal> observer) {
        _obs.Add(observer);
        return new Unsubscriber(_obs, observer); // clean unsubscribe token
    }
    public void SetPrice(decimal p) {
        foreach (var o in _obs.ToArray()) o.OnNext(p);
    }
}

// The other .NET Observer: events (compiler hides add/remove)
public sealed class Order {
    public event EventHandler<OrderShipped>? Shipped; // lapsed-listener leak risk
    public void Ship() => Shipped?.Invoke(this, EventArgs.Empty);
}
```

### 3. DECORATOR — `System.IO.Stream` IS the Textbook
```csharp
using var stream = new CryptoStream(
    new GZipStream(
        new BufferedStream(
            new FileStream("data.bin", FileMode.Open, FileAccess.Read)),
        CompressionMode.Compress),
    key, CryptoStreamMode.Write);

stream.Write(bytes);
stream.FlushFinalBlock(); // CRITICAL: every layer must forward Dispose() or you leak the inner FileStream
```
Other .NET decorators: `DelegatingHandler` (retry/caching/logging/auth — **Polly** is literally this), `ILogger` wrappers, `TextWriter`.

### 4. SINGLETON — .NET Way: `Lazy<T>` + DI
```csharp
public sealed class Config { // sealed blocks inheritance
    private Config() { }
    public static Lazy<Config> Instance { get; } =
        new(() => new Config(), LazyThreadSafetyMode.ExecutionAndPublication);
}

// Preferred in production:
services.AddSingleton<ICache, RedisCache>();

// Preferred for HttpClient:
services.AddHttpClient<IFoo, Foo>(); // IHttpClientFactory

// ASP.NET trap to raise unprompted:
// Injecting a Scoped service into a Singleton = "captive dependency"
// → resolve via IServiceScopeFactory.CreateScope()
```

### 5. STATE — Plus the Counterpoint You Must Defend
```csharp
// Pattern version
public sealed class VendingMachine {
    private State _state = new IdleState();
    public void InsertCoin() => _state.InsertCoin(this);
    public void SetState(State s) => _state = s;
}

// C# counterpoint for SMALL machines — often better:
public enum OrderState { Pending, Paid, Shipped, Cancelled }

public string Next(OrderState s, OrderEvent e) => (s, e) switch {
    (OrderState.Pending,  OrderEvent.Pay)     => nameof(OrderState.Paid),
    (OrderState.Paid,     OrderEvent.Ship)    => nameof(OrderState.Shipped),
    (OrderState.Shipped,  OrderEvent.Cancel)  => "not allowed",
    _ => throw new ArgumentOutOfRangeException(nameof(e))
};

// BCL State examples: DbConnection.State, EF Core EntityState, TaskStatus, HttpConnectionState
```

### 6. ITERATOR — Language Feature First
```csharp
// Language feature (teach first)
public static IEnumerable<int> Fib(int limit = 20) {
    long a = 0, b = 1;
    while (a < limit) { yield return (int)a; (a, b) = (b, a + b); }
}

public static async IAsyncEnumerable<int> PagesAsync(string url) { // C# 8+
    await foreach (var item in client.StreamPagesAsync(url)) yield return item;
}

// Iterator PATTERN — needed when traversal ORDER varies over an object graph:
foreach (var node in tree.DepthFirst()) { }
foreach (var node in tree.BreadthFirst()) { }
```

### 7. FLYWEIGHT — Modern .NET Form Is Pooling
```csharp
var buffer = ArrayPool<byte>.Shared.Rent(4096); // thousands of callers, ONE pool
try { /* use buffer[..n] */ }
finally { ArrayPool<byte>.Shared.Return(buffer); }

// Also: string.Intern, ImmutableArray<T>/ImmutableDictionary (lock-free sharing),
//       MemoryCache<TKey,TValue>, RegexOptions.Compiled, MemoryPool<T>

// Intrinsic (shared) vs extrinsic (per-call) = Roslyn's interned symbols,
// 1M characters in an editor, a forest of 1M identical trees.
```

### 8. CHAIN OF RESPONSIBILITY — ASP.NET Core, Verbatim
```csharp
var app = builder.Build();
app.UseAuthentication();      // handler1
app.UseAuthorization();       // handler 2
app.UseRateLimiter();         // handler 3
app.MapControllers();         // terminal handler

// Interview nuance: middleware pipeline and DelegatingHandler chain are SAME SHAPE
// but DIFFERENT INTENT — CoR (pass along, any subset may handle) vs Decorator
// (every layer MUST add behavior). Arguing that boundary = strong point.
```

---

## 📚 BCL Name-Drop Table (All 25 Patterns)

| Pattern | Real .NET / C# Examples to Cite |
|---------|----------------------------------|
| **Strategy** | `IComparer<T>`, `Comparison<T>`, LINQ `Func<>` operators, `IFormatProvider`/`CultureInfo`, `JsonConverter<T>`, Polly policies |
| **Observer** | `IObservable<T>`/`IObserver<T>`, `event`/`EventHandler<T>`, `INotifyPropertyChanged` (WPF binding, EF change tracking), `ObservableCollection<T>.CollectionChanged`, `Timer.Elapsed`, `PeriodicTimer`, `CancellationToken.Register`, Rx.NET, SignalR hubs |
| **State** | `DbConnection.State` (`Open/Closed/Broken`), EF Core `EntityState`, `TaskStatus`, workflow engines, `System.Activities` |
| **Factory Method** | `DbProviderFactory` (`SqlClientFactory.Instance`), `IServiceProvider.GetRequiredService<T>`, `Activator.CreateInstance<T>`, `new T()` + `new()` constraint |
| **Simple Factory** | Static `Create(kind)` with `switch`/dictionary — DI registration lists, `JsonSerializerOptions.Converters` |
| **Abstract Factory** | `DbProviderFactory` (family: connection + command + parameter per provider), DI `IServiceProvider` as configured family, WPF `Theme` |
| **Builder** | `StringBuilder`, `ConnectionStringBuilder`, `DbContextOptionsBuilder<T>`, `HttpRequestMessage` setup, `ImmutableArray<T>.Builder` — **modern replacement**: `record` + `init` + `required` + `with` |
| **Singleton** | `Lazy<T>`, `AddSingleton<T>`, `IOptions<T>` (singleton config), `IHttpClientFactory`, `JsonSerializer` caches. Hand-rolled = anti-pattern; `Mutex` for cross-process |
| **Prototype** | `record ... with`, collection spread `[.. items]`, `ICloneable` (legacy/ambiguous — say so), JSON round-trip deep copy |
| **Decorator** | **`Stream` (`BufferedStream`/`GZipStream`/`CryptoStream`/`NetworkStream`)**, **`HttpMessageHandler`/`DelegatingHandler`**, Polly resilience pipeline, `ILogger` wrappers, `TextWriter` |
| **Adapter** | `Serilog.Extensions.Logging`, `NLog.Extensions.Logging`, `Microsoft.Extensions.Logging` shims, `DbConnection`/`DbDataReader` over vendor drivers, third-party SDK wrappers, `JsonConverter` for foreign payloads |
| **Facade** | `HttpClient`, `DbContext`, `JsonSerializer`, LINQ, `ILogger`, `ArgumentNullException.ThrowIfNull`, `Regex`, place-order orchestration service |
| **Proxy** | **EF Core lazy-loading proxies**, **gRPC generated client stubs**, `DispatchProxy.Create<T,TProxy>()`, Castle DynamicProxy / Moq, `UseLazyLoadingProxies`, authz proxy |
| **Chain of Responsibility** | **ASP.NET Core middleware**, `DelegatingHandler` chain, `IExceptionHandler` (ASP.NET 8+), `IValidatableObject` + custom validators, Serilog enrichers, `System.Text.Json` converter chain |
| **Command** | `System.Windows.Input.ICommand` (WPF/MVVM), `DbCommand`, `IMenuItem`, undo/redo stacks, macro recording, retryable task queue, saga compensating actions, CLI verbs |
| **Iterator (pattern)** | Custom tree/graph traversal orders, paginated API cursors, `foreach` over a domain graph — vs language feature `yield return`/`IAsyncEnumerable<T>` |
| **Flyweight** | **`ArrayPool<T>`/`MemoryPool<T>`**, `Microsoft.Extensions.ObjectPool.ObjectPool<T>`, `string.Intern`, `ImmutableArray<T>`/`ImmutableDictionary`, `MemoryCache`, `Span<T>`/`stackalloc`, Roslyn symbol interning |
| **Composite** | **LINQ expression trees** (`System.Linq.Expressions`), **ASP.NET Core endpoint-routing tree**, `JsonNode`/`JsonObject`/`JsonArray`, WPF `VisualTreeHelper`, `FileSystemInfo`/`DirectoryInfo`, `CompositeDisposable`, ASTs |
| **Bridge** | `ILogger` × `ILoggerProvider`, `DbConnection`/`DbCommand` × ADO.NET provider, `TextReader`/`TextWriter` × backing store, "message type × transport" (MQTT lib) |
| **Memento** | Immutable-record snapshots, `System.Text.Json` state serialization into a stack, `ImmutableArray<T>` checkpoints, **Azure Durable Functions** history replay |
| **Mediator** | **MediatR** (standard .NET request bus — note: commercial licensing, free below revenue threshold), DI container as implicit mediator, chat room, workflow/saga orchestrator, air-traffic control |
| **Null Object** | **`Array.Empty<T>()`/`Enumerable.Empty<T>()`**, `string.Empty`, `Microsoft.Extensions.Logging.Abstractions.NullLogger`, `NoOp` cache/metrics/gateway, `Nullable<T>`, `??`/`??=`/`?.`, `TryParse`/`TryGetValue` |
| **Template Method** | `Stream.ReadByte` → `Read`, `DbCommand.ExecuteReader`, NUnit/xUnit `[OneTimeSetUp]`/`[SetUp]` hooks, `BackgroundService.ExecuteAsync`, ASP.NET `IStartupFilter`. **Modern alternative**: `Func<Ctx, ValueTask>` composition, TPL Dataflow blocks |
| **Visitor** | Roslyn `CSharpSyntaxVisitor<T>`/`SyntaxVisitor`, reflection-based `Type.GetMethods`, ANTLR — **but say first that pattern matching + `switch` expressions usually beat it in C#** |
| **Interpreter** | `DataTable.Compute`, PowerShell engine, your own recursive-descent calculator / boolean expression parser, `Regex` (pattern-language interpreter), Roslyn as compiler host |
| **Simple Factory** (non-GoF) | Static `Create(kind)` — very idiomatic in C# |

---

## 🔒 Thread-Safety in C# — The Translation of Every `synchronized`/`volatile` Claim

| Java Concept | C# Answer (Use These in Interviews) |
|--------------|--------------------------------------|
| `synchronized` / `ReentrantLock` | `lock (privateReadonlyObject)`; `Monitor.Enter/Exit`; `SemaphoreSlim` (+ `WaitAsync`) for async |
| `AtomicInteger` | `Interlocked.Increment`, `Interlocked.CompareExchange` |
| `BlockingQueue` | **`System.Threading.Channels.Channel<T>`** — `CreateBounded(cap)` with `BoundedChannelFullMode.Wait`; `CreateUnbounded` + `TryWrite/TryRead` |
| `ExecutorService` | `Task.Run`, `Task.WhenAll/WhenAny`, `BackgroundService` (`IHostedService`), TPL Dataflow |
| `ConcurrentHashMap` | `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag`, `ConcurrentStack` |
| `ReadWriteLock` | `ReaderWriterLockSlim` — or `ImmutableDictionary` for lock-free reads |
| `volatile` + DCL singleton | `Lazy<T>` with `LazyThreadSafetyMode.ExecutionAndPublication` |
| Defensive copy | `ToArray()`/`.ToList()` snapshot, or `ImmutableArray<T>` field |
| — | **`CancellationToken` / `CancellationTokenSource`** — cooperative cancellation, `CreateLinkedTokenSource` |
| — | **ASP.NET DI lifetimes**: Singleton (must be thread-safe!), Scoped (per request), Transient. "Can I inject a Scoped into a Singleton?" → **no**, captive dependency |
| — | **async/await correctness**: never `.Result`/`.Wait()` (deadlock + thread-pool starvation); `ConfigureAwait(false)` in libraries; `async` all the way |

**Also have ready:** thread-pool starvation, `Task.Run` for CPU-bound vs I/O-bound, `IAsyncEnumerable` for streaming, backpressure with bounded `Channel<T>`, retry with exponential backoff + jitter via Polly, single-flight/circuit breaker, thread-safe LRU cache with `ConcurrentDictionary` + lock striping.

---

## 🧩 Problems — .NET-Flavored (Twist the Generic Ones)

### Stage A
- **Strategy:** `IComparer<Person>` sorting orders · compression via `DeflateStream`/`BrotliStream` · pricing modes via `Func<>` · LINQ vs manual loop perf
- **Observer:** Stock tick feed via `IObservable<decimal>` · EF Core change notifications · `INotifyPropertyChanged` for WPF/Electron view · log fan-out to console+file+Serilog sinks
- **State:** Order lifecycle (with `EntityState` parallel) · TCP/`HttpClient` connection lifecycle · job pipeline with `BackgroundService`
- **Factory Method:** Log provider selection (`Serilog`/`NLog`/`Microsoft.Extensions`) · `DbProviderFactory` swap · JSON serializer options
- **Simple Factory:** Static `Create(kind)` for notification channels (Email/SMS/Push) · `JsonConverter` registry

### Stage B
- **Abstract Factory:** SQL Server vs Postgres provider bundle · Cloud client (S3/Azure) with matching storage+metrics+auth
- **Builder:** `ConnectionStringBuilder` · EF Core `DbContextOptionsBuilder` with retry + logging + interceptor · `ImmutableArray` request DTO
- **Singleton:** Connection pool · Cache · `IOptions<T>` config · `HttpClient` (and the `IHttpClientFactory` answer)
- **Prototype:** Record `with` for DTO copies · JSON round-trip deep clone · Collection spread `[.. items]`
- **Decorator:** Build compression+encryption+logging `Stream` chain · Retry/caching `DelegatingHandler` with Polly
- **Adapter:** Wrap `Serilog`/`NLog` behind `ILogger` · Unify `DbConnection` over two providers · Wrap a REST vendor SDK
- **Facade:** One-method place-order across inventory/payment/shipping services over EF Core + HttpClient

### Stage C
- **Proxy:** EF Core lazy-loading proxies · Remote proxy over `HttpClient` · Caching proxy for slow service · Moq-backed authz proxy
- **CoR:** ASP.NET middleware (auth → rate-limit → validation → audit) · Password-strength handler chain · Exception-handler chain
- **Command:** Text editor undo/redo with records · MVVM `ICommand` in WPF/Avalonia · `Channel<T>`-drained command queue with retry
- **Iterator:** `yield return` lazy fib/primes · Infinite sequence · Tree in depth-first/breadth-first/in-order · Paginated API cursor via `IAsyncEnumerable`
- **Flyweight:** 1M characters rendered with `ArrayPool<char>` · Forest of 1M trees sharing flyweights · String interning benchmark · `ImmutableDictionary` cache vs `ConcurrentDictionary`
- **Composite:** LINQ expression tree evaluator · AST for calculator · `JsonNode` tree traversal · File-system tree
- **Bridge:** `ILogger` × provider · `DbCommand` × provider · Message type × transport (MQTT)
- **Null Object:** `NullPaymentGateway`, `NullLogger`, `NoOpCache` — show how it kills every `if (x != null)` check

### Stage D
- **Template Method:** Ingest pipeline (read→parse→transform→write) — implement both inheritance version and `Func<>`-pipeline version; argue which you'd ship
- **Memento:** Editor checkpoints serialized to JSON · Durable-Functions-style replay
- **Mediator:** Chat room · MediatR-style `IRequest`/`IRequestHandler` bus · Workflow orchestrator
- **Visitor:** AST walker for calculator with `switch` pattern matching vs real double-dispatch Visitor — compare both
- **Interpreter:** Recursive-descent calculator parsing `System.Text.Json`-ish grammar

### Concurrency Capstones (Do Not Skip)
1. Bounded-queue worker pool with `Channel<T>` + `SemaphoreSlim`
2. Thread-safe LRU cache with `ConcurrentDictionary` + lock striping
3. Producer/consumer with backpressure
4. Retry with exponential backoff + jitter via Polly
5. Single-flight / circuit breaker

### Multi-Pattern Capstones (One Prompt, 3+ Patterns — Real Rounds)
- **Logger** — Singleton + Strategy + Observer
- **Notification Service** — Factory + Strategy + Observer
- **Amazon Order Flow** — Facade + Command + State + Adapter
- **Parking Lot** — Factory + Strategy (allocation)
- **Vending Machine / Elevator / ATM** — State + Strategy + CoR
- **Uber/OLA Fare Engine** — Strategy + Factory + Composite (ride types)
- **Splitwise** — Observer + Factory
- **Home Theater** — Facade + Adapter + Command (HHDP classic)
- **Chat/WhatsApp** — Observer + Mediator + Command
- **BookMyShow / LinkedIn / Chess** — Strategy + State + Observer
- **Expression Evaluator / Compiler** — Interpreter + Visitor + Composite + Template Method
- **LRU Cache / File System / Text Editor with Undo** — Flyweight / Composite / Command+Memento

---

## ⏱ Cadence & Self-Grading Rubric

**Per pattern, per day (~90 min):**
1. **45 min** — Read note / write C# canonical code from memory
2. **20 min** — Memorize §9 scoring phrases + "when NOT to use" + comparison table as flashcards
3. **20 min** — Solve 2 signature problems from scratch (no peeking)
4. **10 min** — Write the "where this pattern shows up" triggers

**Every Saturday (60 min capstone + 20 min grade):**
Grade yourself on the 5 rubric items:
1. **Requirements clarity** — asked clarifying questions, stated assumptions
2. **API-first** — designed interfaces before classes, named patterns out loud with justification
3. **Runnable core** — 2–3 classes compile and run
4. **Complexity + thread-safety** — stated Big-O, identified shared state, proposed correct C# primitives
5. **Extensibility proof** — "add a new X" requires zero edits to existing code (Open/Closed)

---

## 📋 Missing Patterns (14) — To Be Created in Your Template

| Pattern | Category | Priority |
|---------|----------|----------|
| Abstract Factory | Creational | High |
| Prototype | Creational | Low (understand-only) |
| Proxy | Structural | High |
| Chain of Responsibility | Behavioral | High |
| Iterator | Behavioral | Medium (language feature first) |
| Flyweight | Structural | Medium |
| Composite | Structural | Medium |
| Bridge | Structural | Medium |
| Null Object | Behavioral | Medium |
| Template Method | Behavioral | Low |
| Memento | Behavioral | Low |
| Mediator | Behavioral | Low |
| Visitor | Behavioral | Low |
| Interpreter | Behavioral | Very Low |

---

## 🎯 Next Steps

1. **Use this guide as the curriculum.** Follow the 6-week order.
2. **For each pattern:** Read the corresponding section above → write the C# canonical code → solve 2 problems → create flashcards from the "scoring phrases" and BCL name-drops.
3. **When you want the markdown notes generated:** Say the word and I'll create all 25 pattern files in your exact 12-section template (C# canonical code, .NET name-drop tables, C# thread-safety, C#-specific "when NOT to use" caveats), plus a `README.md` master index and per-pattern problem sheets.

---

*Generated from merged Head First Design Patterns, Refactoring.Guru, GoF catalog, and real .NET interview patterns. Target: .NET 10 LTS / C# 14.*