# Strategy Pattern — LLD Notes (SDE 1 / FAANG)

> Behavioral design pattern. The single most common pattern in LLD interviews.
> If you internalize one pattern first, make it this one.

---

## 1. The one-line idea

**Define a family of interchangeable algorithms, put each behind a common interface, and let the client swap them at runtime.**

The object that *uses* the algorithm doesn't care which one it has — it just calls the interface. The concrete choice is injected from outside.

---

## 2. The problem it solves (intuition)

You start with one class that does a job. Then the job needs to be done *differently* depending on the situation:

- A payment can be by UPI, card, or wallet.
- A route can be optimized for *time*, *distance*, or *fewest tolls*.
- A list can be sorted *ascending*, *descending*, or *by custom comparator*.

The naive fix is a giant conditional:

```java
// SMELL: this is what Strategy replaces
if (type == UPI)        { ...upi logic... }
else if (type == CARD)  { ...card logic... }
else if (type == WALLET){ ...wallet logic... }
```

Why this is bad:
- **Open/Closed violation** — every new payment type forces you to edit this method.
- **One class does many jobs** — Single Responsibility violation.
- **Hard to test** — you can't test card logic in isolation.
- **Duplication** — the same `if/else` ladder reappears wherever the behavior is used.

Strategy turns each branch into its own class, all sharing one interface.

---

## 3. Structure (the four roles)

```
            ┌─────────────────────┐
            │      Context         │      uses
            │  (e.g. PaymentService│──────────────┐
            │   has-a Strategy)    │              │
            └─────────┬───────────┘               ▼
                      │ holds a reference   ┌──────────────────┐
                      └────────────────────▶│  <<interface>>   │
                                            │  Strategy        │
                                            │  + execute()     │
                                            └────────┬─────────┘
                          implements                 │
              ┌───────────────────┬──────────────────┤
              ▼                   ▼                  ▼
       ConcreteStrategyA   ConcreteStrategyB   ConcreteStrategyC
       (UpiPayment)        (CardPayment)       (WalletPayment)
```

| Role | What it is | Example |
|------|-----------|---------|
| **Strategy** | Interface declaring the algorithm | `PaymentStrategy` |
| **ConcreteStrategy** | Each implementation | `UpiPayment`, `CardPayment` |
| **Context** | Holds a Strategy, delegates to it | `PaymentService` |
| **Client** | Picks the concrete strategy and hands it to the Context | `main()` / caller |

**Key relationship:** Context **has-a** Strategy (composition), Context does **not** inherit it. "Favor composition over inheritance" — this pattern is the textbook example.

---

## 4. Canonical code example (Java)

```java
// 1. Strategy interface
interface PaymentStrategy {
    void pay(int amountInPaise);
}

// 2. Concrete strategies — each fully self-contained & testable
class UpiPayment implements PaymentStrategy {
    private final String vpa;
    UpiPayment(String vpa) { this.vpa = vpa; }
    public void pay(int amount) {
        System.out.println("Paid " + amount + " via UPI: " + vpa);
    }
}

class CardPayment implements PaymentStrategy {
    private final String cardNumber;
    CardPayment(String cardNumber) { this.cardNumber = cardNumber; }
    public void pay(int amount) {
        System.out.println("Paid " + amount + " via Card ending "
            + cardNumber.substring(cardNumber.length() - 4));
    }
}

// 3. Context — depends on the INTERFACE, never on concrete classes
class PaymentService {
    private PaymentStrategy strategy;          // has-a

    // inject at construction OR allow swapping at runtime
    void setStrategy(PaymentStrategy strategy) {
        this.strategy = strategy;
    }
    void checkout(int amount) {
        if (strategy == null) throw new IllegalStateException("No payment method");
        strategy.pay(amount);                  // delegate
    }
}

// 4. Client picks the strategy
public class Main {
    public static void main(String[] args) {
        PaymentService service = new PaymentService();

        service.setStrategy(new UpiPayment("parthiv@upi"));
        service.checkout(50000);

        service.setStrategy(new CardPayment("1234567812349999")); // swapped at runtime
        service.checkout(120000);
    }
}
```

Notice: adding `WalletPayment` later means writing **one new class** and touching **zero existing code**. That's Open/Closed in action — and exactly the line interviewers want to hear.

---

## 5. When to use it

Reach for Strategy when you see:
- A class has multiple variants of *the same behavior* selected by a flag/enum/type.
- You're writing (or about to write) a growing `if/else` or `switch` over a "type" field.
- Different clients need different variants of an algorithm.
- You want to swap behavior **at runtime**, not just at compile time.

## When NOT to use it

- There are only ever **two fixed** behaviors that will never grow — a simple boolean or `if` may be clearer than the ceremony of an interface + classes.
- The "algorithms" don't actually share a common interface / signature.
- Don't pattern-match for the sake of it. Interviewers penalize over-engineering as much as under-engineering. Use it when variation is real and likely to grow.

---

## 6. Real systems that use Strategy (good things to name-drop)

- **Java `Comparator`** — `Collections.sort(list, comparator)`. The comparator *is* a strategy for ordering. This is the cleanest real example you can cite.
- **`java.util.concurrent` rejection policies** on `ThreadPoolExecutor` (`AbortPolicy`, `CallerRunsPolicy`, ...).
- **Spring's `Resource` loaders / `PlatformTransactionManager`** implementations.
- Pluggable **compression** (zip/gzip/lz4), **payment gateways**, **route planners** (Google Maps "fastest vs shortest"), **pricing/discount engines**.

Saying "this is the same idea as `Comparator` in Java" instantly signals you actually understand it.

---

## 7. Strategy vs. lookalike patterns (interviewers probe this)

| Pattern | Core difference |
|---------|-----------------|
| **State** | *Structurally identical* to Strategy. Difference is **intent**: State transitions between states (often a state changes the Context's next state); Strategy's variants are independent and don't know about each other. State is about "what I am now," Strategy is about "how I do this." |
| **Template Method** | Uses **inheritance** to vary *steps* of a fixed algorithm skeleton. Strategy uses **composition** to vary the *whole* algorithm. Template = compile-time, Strategy = runtime. |
| **Factory** | Factory decides **which** strategy to create; Strategy defines **how** it behaves. They're complementary — a Factory often produces Strategies. |
| **Decorator** | Adds/wraps behavior around an object. Strategy *replaces* a behavior. |

A common, clean combo: a **Factory** returns the right **Strategy** based on input, so the client stays free of `new ConcreteStrategy()` calls.

---

## 8. How it shows up in FAANG LLD rounds

Strategy is rarely the *whole* question — it's a tool you apply inside a bigger design. Common prompts where it earns points:

- **Payment system / Splitwise** → payment methods, split strategies (equal / exact / percentage).
- **Ride-hailing (Uber)** → fare-calculation strategy, driver-matching strategy.
- **Parking lot** → pricing strategy (hourly / flat / day-based).
- **Notification system** → channel strategy (email / SMS / push).
- **File compression / logging** → format strategies.
- **Chess / board games** → move-validation per piece (often Strategy or polymorphism).
- **Vending machine / elevator** → frequently State, but people confuse it with Strategy — know the difference.

**Where to introduce it in the interview flow:** after you've drawn the core entities, when you hit a behavior that varies. Say: *"Discount calculation varies by promo type and we'll add more over time, so I'll make it a Strategy to keep this Open/Closed."* Then draw the interface + 2 concrete classes.

---

## 9. How to talk about it (scoring phrases)

- "I'll extract this into a Strategy so adding a new type doesn't modify existing code — **Open/Closed**."
- "Context depends on the **interface**, not concretions — **Dependency Inversion**."
- "Each variant is its own class — **Single Responsibility**, and independently **unit-testable**."
- "This is the same shape as Java's `Comparator`."
- "A Factory can decide *which* strategy; Strategy decides *how* — keeps the client clean."

These map the pattern to **SOLID**, which is what the interviewer is actually grading.

---

## 10. Common mistakes (avoid these in the room)

1. **Context `new`-ing the concrete strategy itself** → defeats the purpose. Inject it (constructor or setter), or use a Factory.
2. **Strategies needing wildly different parameters** → if signatures don't unify, it's not really one strategy family. Pass a context object, or rethink.
3. **Confusing it with State** and using the wrong vocabulary.
4. **Over-applying it** for a behavior that will never have a second variant.
5. **Stateful strategies shared across threads** → if a strategy holds mutable state, sharing one instance is a bug. Prefer stateless strategies (then a single shared instance, even a singleton/enum, is fine).
6. **Leaking the enum/`switch`** back in — if you still have `switch(type)` *inside* the client to pick the strategy, push that selection into a Factory/Map.

A clean selection map (no `if/else` anywhere):

```java
Map<PaymentType, Supplier<PaymentStrategy>> registry = Map.of(
    PaymentType.UPI,  () -> new UpiPayment(vpa),
    PaymentType.CARD, () -> new CardPayment(card)
);
PaymentStrategy s = registry.get(type).get();
```

---

## 11. Worked problem — surge pricing for a ride-hailing app

**Prompt:** *Design the fare calculation for an Uber-like app. A base fare comes from distance + time, but the final price depends on a pricing mode — Normal, Surge (peak, multiply), Night (flat surcharge), Airport (flat fixed). New modes will be added over time.*

This is a textbook **Strategy** problem, with a **Factory** helping select the strategy (the same combo your Factory notes describe). Strategy is the star here; Factory is the supporting actor.

### Step 1 — spot the varying behavior
"Final price depends on mode … new modes will be added" → a growing `if/else` over a type → extract a **Strategy**.

```java
// The varying algorithm: turn a base fare into a final fare
interface PricingStrategy {
    double calculate(double baseFare);
}
```

### Step 2 — one class per mode (the Strategy core)
```java
class NormalPricing implements PricingStrategy {
    public double calculate(double base) { return base; }
}
class SurgePricing implements PricingStrategy {
    private final double multiplier;
    SurgePricing(double m) { this.multiplier = m; }
    public double calculate(double base) { return base * multiplier; }
}
class NightPricing implements PricingStrategy {
    public double calculate(double base) { return base + 50; }   // flat surcharge
}
class AirportPricing implements PricingStrategy {
    public double calculate(double base) { return 499; }         // flat fixed rate
}
```

### Step 3 — the Context delegates, never branches
```java
class FareCalculator {                  // Context
    private PricingStrategy strategy;    // has-a (composition)
    FareCalculator(PricingStrategy s) { this.strategy = s; }
    void setStrategy(PricingStrategy s) { this.strategy = s; }   // swap at runtime

    double finalFare(double distanceKm, double minutes) {
        double base = distanceKm * 10 + minutes * 2;  // base formula
        return strategy.calculate(base);              // delegate — no if/else here
    }
}
```

### Step 4 — a Factory picks the strategy (keeps the client clean)
```java
enum Mode { NORMAL, SURGE, NIGHT, AIRPORT }

class PricingFactory {
    PricingStrategy create(Mode mode) {
        switch (mode) {
            case SURGE:   return new SurgePricing(1.8);
            case NIGHT:   return new NightPricing();
            case AIRPORT: return new AirportPricing();
            default:      return new NormalPricing();
        }
    }
}

// Client
FareCalculator calc = new FareCalculator(new PricingFactory().create(Mode.SURGE));
calc.finalFare(12, 25);   // base = 12*10 + 25*2 = 370 → *1.8 = 666.0
```

### 🔍 Where Strategy is doing the work (the highlight)
- `FareCalculator` **never knows which pricing it holds** — it just calls `strategy.calculate()`. That delegation through an interface *is* the Strategy pattern.
- Adding "Festival pricing" = **one new class** + one Factory line; `FareCalculator` is untouched → **Open/Closed**.
- Each pricing rule is isolated and **independently unit-testable**.
- The **Factory** only chooses *which* Strategy to build; the **Strategy** owns *how* the fare is computed. Factory = creational, Strategy = behavioral — complementary, not competing. If you removed the Factory you'd still have Strategy; remove the Strategy and the whole design collapses back into the `if/else` smell.

---

## 12. 60-second recall summary

- **Problem:** growing `if/else` over a "type" to pick behavior.
- **Fix:** one interface, one class per behavior, Context holds the interface and delegates.
- **Wins:** Open/Closed, SRP, DIP, runtime swapping, testability — favor composition over inheritance.
- **Twin to know:** State (same structure, different intent).
- **Partner:** Factory picks the strategy.
- **Real example to cite:** `Comparator`.

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅ (this one)
2. Factory / Factory Method
3. Observer (notification systems)
4. State (elevator, vending machine)
5. Decorator (pizza/coffee builder questions)
6. Builder & Singleton (used everywhere, easy wins)
