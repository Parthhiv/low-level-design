# Facade Pattern — LLD Notes (SDE 1 / FAANG)

> Structural design pattern. "One simple front door to a complicated subsystem."
> The pattern that makes a tangle of classes usable through a single, clean entry point.

---

## 1. The one-line idea

**Provide a single, simplified interface to a complex subsystem — the client talks to the facade, which orchestrates all the messy internal classes.**

The subsystem still exists; the facade just gives callers an easy way in for the common cases.

---

## 2. The problem it solves (intuition)

A task requires coordinating **many** classes in the right order, and every caller has to know all of them:

- "Place an order" = check inventory, charge payment, reserve stock, create shipment, send email.
- "Start the home theater" = power on amp, set input, dim lights, lower screen, start projector, play movie.
- "Compress a video" = read file, pick codec, encode, mux audio, write output.

The naive situation: every client wires up the whole subsystem itself:

```java
// SMELL: every caller must know the entire subsystem + the correct order
inventory.check(itemId);
payment.charge(userId, amount);
inventory.reserve(itemId);
Shipment s = shipping.schedule(itemId, address);
notifier.sendConfirmation(userId, s);
// ...this exact sequence copy-pasted across the app; change the flow → edit everywhere
```

Why it's bad:
- Clients are **coupled to every subsystem class** and to the **correct call order**.
- The orchestration is **duplicated** wherever the task is needed.
- Changing the workflow means editing **many** call sites.

Facade puts that orchestration behind one method: `orderService.placeOrder(...)`.

---

## 3. Structure

```
                    ┌──────────────────────────┐
   Client ─────────▶│        Facade            │
   (one simple      │   (OrderFacade)          │
    call)           │   + placeOrder(...)      │
                    └───────────┬──────────────┘
                                │ orchestrates (delegates in the right order)
            ┌───────────────┬───┴────────┬─────────────────┐
            ▼               ▼            ▼                 ▼
       Inventory       Payment       Shipping        Notification
       Service         Gateway       Service         Service
       (subsystem)     (subsystem)   (subsystem)     (subsystem)
```

| Role | What it is | Example |
|------|-----------|---------|
| **Facade** | The single simplified entry point | `OrderFacade` |
| **Subsystem classes** | The complex internals doing real work | `Inventory`, `Payment`, … |
| **Client** | Uses the facade, ignores the subsystem | caller |

**Key:** the facade doesn't *hide* the subsystem (advanced clients can still use it directly); it just offers an easy path for common use.

---

## 4. Canonical code example (Java)

```java
// --- Subsystem classes (complex internals) ---
class InventoryService {
    boolean isAvailable(String item) { System.out.println("Check stock: " + item); return true; }
    void reserve(String item)        { System.out.println("Reserved: " + item); }
}
class PaymentGateway {
    boolean charge(String user, double amt) { System.out.println("Charged " + amt); return true; }
}
class ShippingService {
    String schedule(String item, String addr) { System.out.println("Shipping to " + addr); return "TRK123"; }
}
class NotificationService {
    void confirm(String user, String tracking) { System.out.println("Email: tracking " + tracking); }
}

// --- Facade: one simple method orchestrating the subsystem ---
class OrderFacade {
    private final InventoryService inventory   = new InventoryService();
    private final PaymentGateway payment        = new PaymentGateway();
    private final ShippingService shipping      = new ShippingService();
    private final NotificationService notifier  = new NotificationService();

    public boolean placeOrder(String user, String item, double amount, String address) {
        if (!inventory.isAvailable(item))      return false;
        if (!payment.charge(user, amount))     return false;
        inventory.reserve(item);
        String tracking = shipping.schedule(item, address);
        notifier.confirm(user, tracking);
        return true;   // the whole dance behind ONE call
    }
}

// --- Client: blissfully unaware of the subsystem ---
public class Main {
    public static void main(String[] args) {
        OrderFacade orders = new OrderFacade();
        orders.placeOrder("parthiv", "Pizza Oven", 4999.0, "Bengaluru");
    }
}
```

The client makes one call; the facade knows the steps and the order. Change the workflow → edit the facade only.

---

## 5. When to use it

- A subsystem is **complex** and clients only need a few common operations.
- You want to **decouple** clients from subsystem internals (so internals can change freely).
- You want a clear **entry point / API layer** over a tangle of classes.
- You're **wrapping a complex library** to expose only what your app needs.

## When NOT to use it

- The subsystem is already **simple** — a facade adds a pointless layer.
- Clients genuinely need **fine-grained control** over every subsystem class (don't force everything through the facade).
- It risks becoming a **god object** that accumulates unrelated responsibilities — keep facades focused.

---

## 6. Real systems that use Facade (good to name-drop)

- **SLF4J `LoggerFactory`** — simple front over complex logging backends.
- **`javax.faces` / `java.net.URL.openConnection()`** — simple call hiding protocol machinery.
- **Spring `JdbcTemplate`** — one clean API over the verbose JDBC subsystem (connection, statement, resultset, cleanup). Great example.
- **`java.util.concurrent.Executors`** — facade-ish factory hiding `ThreadPoolExecutor` complexity.
- **SDK clients** (AWS SDK service clients, Stripe client) — facades over HTTP/auth/retry internals.

"Spring's `JdbcTemplate` is a facade over raw JDBC" is a strong, recognizable example.

---

## 7. Facade vs. lookalike patterns (interviewers probe this)

| Pattern | Core difference |
|---------|-----------------|
| **Adapter** | Adapter makes **one** existing class fit an **expected** interface (compatibility). Facade invents a **new simplified** interface over a **whole subsystem** (convenience). Adapter = translate; Facade = simplify. |
| **Mediator** | Mediator coordinates **two-way** interaction *between* subsystem objects (they talk through it). Facade is a **one-way** front door (client → subsystem); subsystem objects don't know the facade. |
| **Proxy** | Proxy has the **same interface** as its subject and controls access. Facade has a **different, simpler** interface and doesn't restrict access. |
| **Abstract Factory** | Could be hidden behind a facade, but its job is creating families; facade's job is simplifying usage. |

The crisp line: **Adapter converts one interface, Facade simplifies many.**

---

## 8. How it shows up in FAANG LLD rounds

Facade usually appears as the **service/orchestration layer** of a bigger design:

- **E-commerce / order system** → `OrderService` orchestrating inventory + payment + shipping + notification (the textbook case).
- **Booking systems** (movie/flight/hotel) → `BookingService` coordinating seats + payment + tickets.
- **Home theater / smart home** → one `watchMovie()` driving many devices (the GoF example).
- **Compiler** → a `Compiler` facade over lexer + parser + codegen.
- **Video/file processing** → one `convert()` over a multi-step pipeline.
- **Any "place order / submit / process X" use case** that touches several services.

**Where to introduce it:** when your design has several collaborating services and a client just wants the end result. Say: *"I'll put a facade/service layer in front so callers invoke one `placeOrder()` and stay decoupled from the subsystem and its call order."*

---

## 9. How to talk about it (scoring phrases)

- "The facade gives clients **one simple entry point** and decouples them from subsystem internals."
- "Subsystem classes can change without touching clients — **loose coupling**."
- "It owns the **orchestration and ordering**, so the workflow lives in one place."
- "It **simplifies** rather than translates — that's how it differs from Adapter."
- "I'll keep it focused so it doesn't become a god object."

---

## 10. Common mistakes (avoid these in the room)

1. **Confusing it with Adapter** — Facade simplifies a subsystem; Adapter converts one class's interface.
2. **God-object facade** — piling unrelated operations into one class; split by concern.
3. **Forcing all access through the facade** — facade should *offer* a simple path, not *forbid* direct subsystem use when needed.
4. **Putting heavy business logic in the facade** — it should mostly *orchestrate*; deep logic belongs in the subsystem.
5. **Leaking subsystem types** in the facade's signatures — defeats the decoupling.

---

## 11. Worked problem — a "place order" service for an e-commerce app

**Prompt:** *Placing an order must check inventory, process payment, reserve stock, schedule shipping, and send a confirmation — each handled by a separate service. Callers (web, mobile, admin) should trigger the whole flow with one call and not depend on the individual services or their order.*

Facade is the highlighted pattern. (Internally the subsystem may itself use **Strategy** for payment, **Observer** for notifications, etc. — but the thing taming the complexity for callers is the Facade.)

### Step 1 — the subsystem services (each does one job, independently)
```java
class InventoryService {
    boolean inStock(String sku, int qty) { /* query DB */ return true; }
    void reserve(String sku, int qty)    { /* decrement stock */ }
}
class PaymentService {
    boolean pay(String userId, double amount) { /* charge via gateway */ return true; }
}
class ShippingService {
    String createShipment(String sku, String address) { /* book courier */ return "TRACK-77"; }
}
class NotificationService {
    void orderConfirmed(String userId, String tracking) { /* email/SMS */ }
}
```

### Step 2 — the Facade orchestrates them behind one method
```java
class OrderService {                                   // the Facade
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;
    private final NotificationService notifier;

    OrderService(InventoryService i, PaymentService p, ShippingService s, NotificationService n) {
        this.inventory = i; this.payment = p; this.shipping = s; this.notifier = n;
    }

    public OrderResult placeOrder(String userId, String sku, int qty, double amount, String address) {
        if (!inventory.inStock(sku, qty))      return OrderResult.fail("Out of stock");
        if (!payment.pay(userId, amount))      return OrderResult.fail("Payment declined");
        inventory.reserve(sku, qty);                          // step order matters — owned here
        String tracking = shipping.createShipment(sku, address);
        notifier.orderConfirmed(userId, tracking);
        return OrderResult.ok(tracking);
    }
}
record OrderResult(boolean success, String message) {
    static OrderResult ok(String tracking) { return new OrderResult(true, tracking); }
    static OrderResult fail(String reason)  { return new OrderResult(false, reason); }
}
```

### Step 3 — clients make ONE call
```java
OrderService orders = new OrderService(
        new InventoryService(), new PaymentService(),
        new ShippingService(),  new NotificationService());

OrderResult r = orders.placeOrder("u_42", "PIZZA-OVEN", 1, 4999.0, "Bengaluru");
System.out.println(r.success() ? "Order placed: " + r.message() : "Failed: " + r.message());
```

### 🔍 Where Facade is doing the work (the highlight)
- `OrderService.placeOrder(...)` is the **single front door**. The web/mobile/admin clients call it and stay completely unaware that five separate services — and a specific *order* of operations (stock → pay → reserve → ship → notify) — sit behind it. Hiding that orchestration behind one simple method *is* the Facade pattern.
- The **subsystem still exists and stays independent** — `PaymentService`, `ShippingService`, etc. don't know the facade exists, and an advanced caller could still use one directly. The facade *offers* simplicity, it doesn't lock the subsystem away.
- The workflow and its ordering live in **one place**. Change the flow (e.g. add fraud-check before payment) → edit only `OrderService`; the three client apps don't change → **loose coupling**, no duplicated orchestration.
- Contrast with Adapter: nothing here is being *converted* to match an expected interface — the facade is inventing a brand-new, *simpler* API over many classes. Adapter translates one interface; Facade simplifies many.

---

## 12. 60-second recall summary

- **Problem:** a complex subsystem of many classes; every client must know them all and the right call order.
- **Fix:** a Facade exposes one simplified method that orchestrates the subsystem internally.
- **Wins:** loose coupling, single entry point, orchestration in one place, freedom to change internals.
- **Distinguish:** Facade *simplifies* a whole subsystem; Adapter *converts* one class's interface; Mediator coordinates two-way between objects.
- **Watch out:** don't let it become a god object; don't forbid direct subsystem access.
- **Real example to cite:** Spring `JdbcTemplate` over raw JDBC, SLF4J `LoggerFactory`.

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅ · 2. Factory ✅ · 3. Observer ✅ · 4. State ✅ · 5. Decorator ✅ · 6. Builder ✅ · 7. Singleton ✅ · 8. Adapter ✅ · 9. Command ✅ · 10. Chain of Responsibility ✅
11. Facade ✅ (this one)

**You've now covered the 11 patterns that handle virtually every LLD interview.** Remaining GoF patterns (Proxy, Composite, Template Method, Bridge, Flyweight, Prototype, Iterator, Mediator, Memento, Visitor, Interpreter) are completeness-only — learn them if you want full coverage.
