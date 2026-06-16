# Factory Pattern — LLD Notes (SDE 1 / FAANG)

> Creational design pattern. The standard answer to "how do I create objects without `new`-ing concrete classes everywhere?"
> Comes right after Strategy because it's the pattern that *picks* the strategy.

---

## 1. The one-line idea

**Move object creation behind a dedicated method/class, so the client asks for an object by type/intent and gets back an interface — never knowing which concrete class was instantiated.**

The client says *what* it wants, the factory decides *which* concrete class to build and how.

> Heads up: "Factory" is an umbrella for three related things people lump together. Know all three and which one you're using:
> - **Simple Factory** — one class with a `create(type)` method. *Not* a GoF pattern, but by far the most common in interviews.
> - **Factory Method** — a creation method declared in a base class/interface and overridden by subclasses to decide the concrete product. (GoF)
> - **Abstract Factory** — a factory of factories: produces *families* of related objects. (GoF)

---

## 2. The problem it solves (intuition)

Object creation gets sprinkled across your code:

```java
// SMELL: scattered, duplicated, tightly coupled to concretions
Notification n;
if (channel.equals("EMAIL"))      n = new EmailNotification(cfg);
else if (channel.equals("SMS"))   n = new SmsNotification(cfg);
else if (channel.equals("PUSH"))  n = new PushNotification(cfg);
```

This same ladder reappears in every place that needs a notification. Why it's bad:

- **Tight coupling** — every caller depends on every concrete class (`new EmailNotification`).
- **Open/Closed violation** — adding `WhatsAppNotification` means editing every such block.
- **Duplication** — the construction logic (and any config wiring) is copy-pasted.
- **Mixed responsibilities** — business code is also doing object-assembly work.

Factory centralizes creation in one place. Callers depend only on the `Notification` interface and the factory.

---

## 3. Structure

### Simple Factory
```
   Client ──asks──▶ NotificationFactory.create("EMAIL")
                              │ decides
                              ▼
                    ┌──────────────────┐
                    │  <<interface>>   │
                    │  Notification    │
                    │  + send()        │
                    └────────┬─────────┘
            implements       │
        ┌───────────┬────────┴────────┐
        ▼           ▼                 ▼
   EmailNotif   SmsNotif         PushNotif
```

### Factory Method (creation deferred to subclasses)
```
        ┌────────────────────────┐
        │   <<abstract>>         │
        │   Dialog               │
        │   + render()           │   ← calls createButton()
        │   + createButton()     │   ← abstract "factory method"
        └──────────┬─────────────┘
                   │ overridden by
        ┌──────────┴───────────┐
        ▼                      ▼
   WindowsDialog          WebDialog
   createButton()→Win     createButton()→Html
```

| Role | What it is | Example |
|------|-----------|---------|
| **Product** | Interface of what's created | `Notification`, `Button` |
| **ConcreteProduct** | Each implementation | `EmailNotification`, `HtmlButton` |
| **Factory / Creator** | Decides which product to build | `NotificationFactory`, `Dialog` |
| **Client** | Asks the factory, uses the Product interface | caller / `main()` |

---

## 4. Canonical code example (Java)

### Simple Factory (use this in most interviews)
```java
// 1. Product interface
interface Notification {
    void send(String to, String body);
}

// 2. Concrete products — self-contained & testable
class EmailNotification implements Notification {
    public void send(String to, String body) {
        System.out.println("EMAIL → " + to + ": " + body);
    }
}
class SmsNotification implements Notification {
    public void send(String to, String body) {
        System.out.println("SMS → " + to + ": " + body);
    }
}
class PushNotification implements Notification {
    public void send(String to, String body) {
        System.out.println("PUSH → " + to + ": " + body);
    }
}

// 3. The factory — the ONLY place that knows the concrete classes
enum Channel { EMAIL, SMS, PUSH }

class NotificationFactory {
    Notification create(Channel channel) {
        switch (channel) {
            case EMAIL: return new EmailNotification();
            case SMS:   return new SmsNotification();
            case PUSH:  return new PushNotification();
            default:    throw new IllegalArgumentException("Unknown channel: " + channel);
        }
    }
}

// 4. Client depends only on the interface + factory
public class Main {
    public static void main(String[] args) {
        NotificationFactory factory = new NotificationFactory();
        Notification n = factory.create(Channel.SMS);
        n.send("+91999...", "Your OTP is 4321");
    }
}
```

### Registry variant (no `switch` at all — the clean, extensible version)
```java
class NotificationFactory {
    private final Map<Channel, Supplier<Notification>> registry = new EnumMap<>(Channel.class);

    NotificationFactory() {
        register(Channel.EMAIL, EmailNotification::new);
        register(Channel.SMS,   SmsNotification::new);
        register(Channel.PUSH,  PushNotification::new);
    }
    void register(Channel c, Supplier<Notification> creator) { registry.put(c, creator); }

    Notification create(Channel c) {
        Supplier<Notification> creator = registry.get(c);
        if (creator == null) throw new IllegalArgumentException("Unknown channel: " + c);
        return creator.get();
    }
}
```
With the registry, adding a new channel is `register(...)` + one new class — **zero edits** to `create()`. That's Open/Closed, and a strong thing to show in the room.

---

## 5. When to use it

- Object creation involves a **choice between several concrete types** based on input/config.
- You see `new ConcreteX()` scattered across the codebase and want to centralize it.
- The client should depend on an **interface**, not on concrete constructors (Dependency Inversion).
- Construction is **non-trivial** (wiring config, choosing implementation by environment) and you don't want callers doing it.
- **Abstract Factory** specifically: you must create **families** of related products that must be used together (e.g. a `WindowsButton` + `WindowsCheckbox`, never mixed with web ones).

## When NOT to use it

- There's only **one** concrete type and likely always will be — just call the constructor.
- Construction is trivial and the concrete type is genuinely fixed; a factory adds ceremony for nothing.
- You actually need to vary *behavior at runtime on an existing object* → that's **Strategy**, not Factory.

---

## 6. Real systems that use Factory (good to name-drop)

- **`java.util.Calendar.getInstance()`** — returns a locale-appropriate concrete `Calendar`; you never `new` it.
- **`Collection` factories** — `List.of(...)`, `Set.of(...)`, `Collections.unmodifiableList(...)`.
- **JDBC `DriverManager.getConnection(url)`** — returns a driver-specific `Connection`.
- **`java.util.concurrent.Executors`** — `newFixedThreadPool()`, `newCachedThreadPool()` (static factory methods).
- **Spring `BeanFactory` / `ApplicationContext`** — the whole container is an industrial-strength factory.
- **`Integer.valueOf(int)`** — a static factory that *caches* small values (factory ≠ always `new`, a key insight).

Citing `valueOf` caching shows you understand a factory can do more than a constructor: pooling, caching, returning subtypes, returning nulls/cached singletons.

---

## 7. Factory vs. lookalike patterns (interviewers probe this)

| Pattern | Core difference |
|---------|-----------------|
| **Strategy** | Factory decides **which** object to create; Strategy defines **how** an object behaves. Complementary: a Factory often *produces* a Strategy. Factory = creational, Strategy = behavioral. |
| **Builder** | Builder constructs **one complex object step-by-step** (many optional params); Factory picks **which type** to create in one call. Use Builder when the *construction* is complex, Factory when the *choice of type* is the problem. |
| **Abstract Factory** | A factory that creates **families** of related products. Simple/Method Factory creates **one** product. |
| **Factory Method vs Simple Factory** | Factory Method uses **inheritance** (subclass overrides the creating method); Simple Factory is **one class with a conditional**. Factory Method is "real" GoF; Simple Factory is the pragmatic everyday one. |
| **Singleton** | Often combined — the factory itself may be a singleton. Different intent: Singleton controls *how many* instances, Factory controls *which class*. |

---

## 8. How it shows up in FAANG LLD rounds

Factory is a supporting pattern — it removes `new`/`switch` noise from a bigger design:

- **Notification system** → `NotificationFactory` picks channel implementation.
- **Payment system / Splitwise** → factory builds the right `PaymentStrategy` (Factory + Strategy combo).
- **Logging framework** → factory creates appenders/formatters by config.
- **Parking lot** → factory creates the right `Vehicle`/`Spot` by type.
- **Pizza / coffee shops (classic)** → `PizzaFactory` / Abstract Factory for ingredient families.
- **Ride-hailing** → factory for vehicle types (Sedan/SUV/Auto), each with its own fare strategy.
- **UI toolkits / cross-platform** → Abstract Factory (Windows vs Mac vs Web widget families).

**Where to introduce it:** the moment your client code is about to `new` a concrete class chosen by a flag. Say: *"I'll hide this construction behind a factory so callers depend on the interface and adding a new type doesn't touch them."*

---

## 9. How to talk about it (scoring phrases)

- "Callers depend on the **interface**, not concrete constructors — **Dependency Inversion**."
- "Creation lives in one place, so a new type is a new class + one registry line — **Open/Closed**."
- "The factory owns object-assembly; business code stays focused — **Single Responsibility**."
- "A Factory decides *which*; a Strategy decides *how* — I'll combine them here."
- "A static factory can cache or pool, unlike a raw constructor — like `Integer.valueOf`."

---

## 10. Common mistakes (avoid these in the room)

1. **Leaving the `switch`/`if-else` in `create()`** and calling that "done" — fine for small cases, but mention the **registry/map** variant to show you know how to make it Open/Closed.
2. **Returning concrete types** from the factory → leaks the abstraction. Return the **interface**.
3. **Confusing Factory with Builder** — if the hard part is *many optional fields*, that's Builder, not Factory.
4. **Confusing Factory with Strategy** — Factory creates, Strategy behaves. Saying "I'll use a Factory to swap behavior at runtime" is the wrong vocabulary.
5. **Over-engineering to Abstract Factory** when there's only one product family — pick the simplest variant that fits.
6. **Factory that also runs business logic** — keep it to construction; don't let it grow into a god class.

---

## 11. Worked problem — a report export module

**Prompt:** *A reporting feature must export a report to PDF, CSV, or HTML. The caller shouldn't know how each format is produced, and new formats (Excel, JSON) will be added frequently.*

This is a clean **Factory** problem — the highlighted pattern. (You could later add a **Strategy** for, say, compression, but the creation decision is the heart of it.)

### Step 1 — one product interface the caller depends on
```java
interface Exporter {
    byte[] export(Report report);
}
```

### Step 2 — concrete products (each self-contained)
```java
class PdfExporter  implements Exporter { public byte[] export(Report r) { /* build PDF  */ return new byte[0]; } }
class CsvExporter  implements Exporter { public byte[] export(Report r) { /* build CSV  */ return new byte[0]; } }
class HtmlExporter implements Exporter { public byte[] export(Report r) { /* build HTML */ return new byte[0]; } }
```

### Step 3 — the Factory, registry style (no switch → Open/Closed)
```java
enum Format { PDF, CSV, HTML }

class ExporterFactory {
    private final Map<Format, Supplier<Exporter>> registry = new EnumMap<>(Format.class);

    ExporterFactory() {
        register(Format.PDF,  PdfExporter::new);
        register(Format.CSV,  CsvExporter::new);
        register(Format.HTML, HtmlExporter::new);
    }
    void register(Format f, Supplier<Exporter> creator) { registry.put(f, creator); }

    Exporter create(Format f) {
        Supplier<Exporter> creator = registry.get(f);
        if (creator == null) throw new IllegalArgumentException("Unsupported format: " + f);
        return creator.get();
    }
}

// Client — depends only on the interface + the factory
Exporter exporter = factory.create(Format.PDF);
byte[] file = exporter.export(report);
```

### 🔍 Where Factory is doing the work (the highlight)
- The client says **what** it wants (`Format.PDF`) and receives an `Exporter` — it never calls `new PdfExporter()` and is decoupled from every concrete class → **Dependency Inversion**.
- The concrete-class knowledge lives in **exactly one place** (the registry). Adding `ExcelExporter` = one new class + one `register(...)` line, with **zero edits** to `create()` or to any caller → **Open/Closed**.
- Construction logic is owned by the factory, keeping business code focused → **Single Responsibility**.
- Because creation is centralized, the factory could also **cache or pool** exporters (like `Integer.valueOf`) without callers ever knowing — something a raw constructor can't do.

---

## 12. 60-second recall summary

- **Problem:** scattered `new ConcreteX()` chosen by a type/flag, coupling callers to concretions.
- **Fix:** centralize creation behind a factory that returns an interface.
- **Three flavors:** Simple Factory (1 class + conditional), Factory Method (subclass overrides creator), Abstract Factory (families of products).
- **Wins:** Dependency Inversion, Open/Closed (esp. with a registry), SRP.
- **Partner:** Strategy — Factory picks *which*, Strategy defines *how*.
- **Real example to cite:** `Calendar.getInstance()`, `Integer.valueOf()` (caching), `Executors`.

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅
2. Factory / Factory Method ✅ (this one)
3. Observer (notification systems) ← next
4. State (elevator, vending machine)
5. Decorator (pizza/coffee builder questions)
6. Builder & Singleton (used everywhere, easy wins)
