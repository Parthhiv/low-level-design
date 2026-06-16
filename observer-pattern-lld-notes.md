# Observer Pattern — LLD Notes (SDE 1 / FAANG)

> Behavioral design pattern. The go-to answer for "when X changes, notify everyone who cares."
> Shows up the instant a question mentions *notifications*, *events*, *subscriptions*, or *real-time updates*.

---

## 1. The one-line idea

**Define a one-to-many dependency: when one object (the Subject) changes state, all its registered dependents (Observers) are notified and updated automatically.**

The Subject doesn't know *who* is listening or *what* they do — it just broadcasts. Observers subscribe and unsubscribe at runtime.

---

## 2. The problem it solves (intuition)

Some object holds state others need to react to:

- A YouTube channel uploads a video → all subscribers get notified.
- A stock price changes → every dashboard/alert watching it updates.
- An order's status changes → email, SMS, and analytics all need to know.

The naive fix hard-wires the subject to its consumers:

```java
// SMELL: subject knows about every consumer concretely
class Order {
    void ship() {
        this.status = SHIPPED;
        emailService.sendShippedEmail(this);   // tightly coupled
        smsService.sendShippedSms(this);        // must edit this method
        analytics.track(this);                  // ...for every new consumer
    }
}
```

Why it's bad:
- **Tight coupling** — `Order` depends on email, SMS, analytics concretely.
- **Open/Closed violation** — every new consumer edits `ship()`.
- **Subject doing too much** — it's managing notifications, not just orders (SRP).
- **No runtime flexibility** — can't add/remove listeners dynamically.

Observer inverts this: consumers *register themselves*; the subject just calls a common `update()` on whoever's subscribed.

---

## 3. Structure

```
        ┌───────────────────────────┐
        │     <<interface>>         │
        │     Subject               │
        │  + subscribe(Observer)    │
        │  + unsubscribe(Observer)  │
        │  + notifyObservers()      │
        └────────────┬──────────────┘
                     │ implements
                     ▼
        ┌───────────────────────────┐         holds list of
        │   ConcreteSubject         │────────────────────────┐
        │   (e.g. Channel)          │                        │
        │   - observers: List<Obs>  │                        ▼
        │   - state                 │            ┌──────────────────────┐
        │   + setState() ──notifies │            │   <<interface>>      │
        └───────────────────────────┘            │   Observer           │
                                                 │   + update(data)     │
                                                 └──────────┬───────────┘
                                       implements           │
                              ┌──────────────────┬──────────┴────────┐
                              ▼                  ▼                   ▼
                       EmailSubscriber   SmsSubscriber       AnalyticsLogger
```

| Role | What it is | Example |
|------|-----------|---------|
| **Subject (Observable)** | Maintains observer list, notifies them | `Channel`, `Stock` |
| **Observer** | Interface with `update()` | `Subscriber` |
| **ConcreteObserver** | Reacts to the notification | `EmailSubscriber`, `SmsSubscriber` |
| **Client** | Wires subjects and observers together | `main()` |

**Key relationship:** Subject holds a list of the **Observer interface** — never concrete observers. That's what keeps it decoupled.

---

## 4. Canonical code example (Java)

```java
// 1. Observer interface
interface Subscriber {
    void update(String videoTitle);
}

// 2. Subject interface
interface Channel {
    void subscribe(Subscriber s);
    void unsubscribe(Subscriber s);
    void notifySubscribers(String videoTitle);
}

// 3. Concrete subject — owns the observer list + the state
class YoutubeChannel implements Channel {
    private final List<Subscriber> subscribers = new ArrayList<>();
    private final String name;
    YoutubeChannel(String name) { this.name = name; }

    public void subscribe(Subscriber s)   { subscribers.add(s); }
    public void unsubscribe(Subscriber s) { subscribers.remove(s); }

    public void notifySubscribers(String videoTitle) {
        // iterate over a copy is safer if observers can unsubscribe during update
        for (Subscriber s : new ArrayList<>(subscribers)) {
            s.update(videoTitle);
        }
    }

    // state change triggers the broadcast
    public void uploadVideo(String videoTitle) {
        System.out.println(name + " uploaded: " + videoTitle);
        notifySubscribers(videoTitle);
    }
}

// 4. Concrete observers — each reacts its own way, independently testable
class EmailSubscriber implements Subscriber {
    private final String email;
    EmailSubscriber(String email) { this.email = email; }
    public void update(String videoTitle) {
        System.out.println("EMAIL → " + email + ": New video '" + videoTitle + "'");
    }
}
class SmsSubscriber implements Subscriber {
    private final String phone;
    SmsSubscriber(String phone) { this.phone = phone; }
    public void update(String videoTitle) {
        System.out.println("SMS → " + phone + ": New video '" + videoTitle + "'");
    }
}

// 5. Client wires it up
public class Main {
    public static void main(String[] args) {
        YoutubeChannel channel = new YoutubeChannel("LLD Academy");

        Subscriber a = new EmailSubscriber("parthiv@x.com");
        Subscriber b = new SmsSubscriber("+91999...");

        channel.subscribe(a);
        channel.subscribe(b);

        channel.uploadVideo("Observer Pattern in 10 min");  // both notified

        channel.unsubscribe(b);
        channel.uploadVideo("Factory Pattern in 10 min");   // only email notified
    }
}
```

Adding a `PushSubscriber` later = **one new class**, `channel.subscribe(...)`, and **zero edits** to `YoutubeChannel`. Open/Closed.

---

## 5. Push vs. Pull model (interviewers love this distinction)

When the subject notifies, how much data does it send?

- **Push model** — subject sends the data in `update(data)` (as above). Simple, but the subject must guess what observers need; over-sends or forces a fat interface.
- **Pull model** — subject sends only "something changed" (often a reference to itself), and each observer *pulls* what it needs: `update(subject)` → `subject.getState()`. More flexible; observers take only what they care about. Slightly more coupling (observer needs the subject's API).

Say in the room: *"I'll use the pull model so observers fetch only the fields they need and I don't bloat the update signature."*

---

## 6. When to use it

- A state change in one object must trigger updates in an **unknown/variable number** of others.
- You want **loose coupling** between the source of an event and its handlers.
- Subscribers should be able to **join/leave at runtime**.
- You're building anything described as *notifications, events, subscriptions, listeners, pub/sub, real-time feeds*.

## When NOT to use it

- The set of reactions is **fixed and tiny** and will never change — a direct call may be clearer.
- You need **ordering guarantees, delivery guarantees, or cross-process** delivery → that's a **message queue / event bus** (Kafka, etc.), not the in-memory Observer pattern. Mention this; it's a maturity signal.
- Update chains risk **cascading/cyclic notifications** (A updates B updates A...) — Observer makes these easy to create accidentally.

---

## 7. Real systems that use Observer (good to name-drop)

- **Java `java.util.Observer`/`Observable`** — the original (now **deprecated** since Java 9 — know this; cite `PropertyChangeListener` or listeners instead as the modern form).
- **Swing/AWT event listeners** — `addActionListener`, `addMouseListener` are Observer.
- **`PropertyChangeListener` / JavaBeans** — the canonical modern JDK example.
- **RxJava / Reactive Streams** — `Observable`/`Observer`, the pattern scaled up.
- **Frontend frameworks** — React state subscriptions, Vue reactivity, DOM `addEventListener`.
- **Spring `ApplicationEvent` / `@EventListener`**, the pub/sub of the framework.
- **Kafka / RabbitMQ** — the *distributed* descendants of this idea (good to contrast).

Saying "Java's `Observable` is actually deprecated, so I'd model it with a listener interface or use `PropertyChangeListener`" is a strong signal.

---

## 8. Observer vs. lookalike patterns (interviewers probe this)

| Pattern | Core difference |
|---------|-----------------|
| **Pub/Sub (message broker)** | Observer is **in-process, synchronous, direct** — subject holds references to observers. Pub/Sub adds a **broker/channel** in the middle; publishers and subscribers don't know each other at all, and it's usually async/cross-process. Observer is the design pattern; Pub/Sub is the architecture. |
| **Mediator** | Mediator centralizes **complex many-to-many** communication through one hub. Observer is **one-to-many broadcast** in one direction. |
| **Chain of Responsibility** | CoR passes a request along until *one* handler handles it. Observer notifies *all* observers. |
| **Strategy** | Strategy swaps *how one thing behaves*; Observer *broadcasts events to many*. Unrelated intents, sometimes combined. |

---

## 9. How it shows up in FAANG LLD rounds

Observer is frequently a *core* part of the question when notifications are involved:

- **Notification system** → the headline use case (often Observer + Factory + Strategy together).
- **Stock price / trading dashboard** → price ticker is the subject, dashboards/alerts are observers.
- **Splitwise** → notify members when an expense is added.
- **Logging framework** → log events broadcast to multiple appenders.
- **Chat / messaging app** → new message notifies all participants in a room.
- **Cricket/sports scoreboard** (classic textbook) → score updates pushed to displays.
- **Auction system** → new bid notifies all bidders.
- **Cache invalidation** → data change notifies dependent caches.

**Where to introduce it:** as soon as the prompt says "notify", "when X happens then Y, Z should...", or "real-time updates". Say: *"This is a one-to-many event broadcast, so I'll use Observer — the subject keeps a list of listeners and stays decoupled from them."*

---

## 10. How to talk about it (scoring phrases)

- "Subject depends only on the **Observer interface**, not concrete listeners — **loose coupling / DIP**."
- "Adding a new listener is a new class + `subscribe()` — **Open/Closed**, no edits to the subject."
- "Subscribers can register/unregister at **runtime**."
- "I'll use the **pull model** so the update signature stays lean."
- "In-process this is Observer; if I needed durability or cross-service delivery I'd move to a **message queue / pub-sub broker**."

---

## 11. Common mistakes (avoid these in the room)

1. **Subject holding concrete observers** instead of the interface → defeats the decoupling.
2. **`ConcurrentModificationException`** — if an observer unsubscribes during `notifyObservers()`, iterating the live list crashes. Iterate over a **copy**, or use `CopyOnWriteArrayList`.
3. **Memory leaks (lapsed listener)** — observers that never unsubscribe stay referenced by the subject forever. Always provide `unsubscribe`; consider weak references for long-lived subjects.
4. **Notification order assumptions** — don't assume observers fire in any particular order; if order matters, Observer may be the wrong tool.
5. **Cascading/cyclic updates** — an observer that mutates the subject can trigger infinite notification loops. Guard against re-entrancy.
6. **Doing heavy/blocking work in `update()`** — a slow observer blocks the whole broadcast (it's synchronous). Offload to a queue/async if needed.
7. **Confusing it with full Pub/Sub** — using "broker", "topic", "async delivery" vocabulary for what is actually an in-memory synchronous pattern.

---

## 12. Worked problem — a stock-price alert system

**Prompt:** *Design a system where users watch a stock. When its price changes, every interested party must be updated automatically — a mobile app view, an email alerter, and an analytics logger. Watchers can come and go at runtime.*

This is the canonical **Observer** problem (the highlighted pattern). It pairs naturally with a tiny bit of **Strategy** (how each observer reacts), but Observer is what makes the broadcast work.

### Step 1 — the two interfaces
```java
interface PriceObserver {                 // Observer
    void update(String symbol, double price);
}
interface StockTicker {                    // Subject
    void subscribe(PriceObserver o);
    void unsubscribe(PriceObserver o);
    void notifyObservers();
}
```

### Step 2 — the concrete Subject owns the list + the state
```java
class Stock implements StockTicker {
    private final List<PriceObserver> observers = new CopyOnWriteArrayList<>(); // safe under concurrent (un)subscribe
    private final String symbol;
    private double price;
    Stock(String symbol, double price) { this.symbol = symbol; this.price = price; }

    public void subscribe(PriceObserver o)   { observers.add(o); }
    public void unsubscribe(PriceObserver o) { observers.remove(o); }
    public void notifyObservers() {
        for (PriceObserver o : observers) o.update(symbol, price);
    }

    public void setPrice(double newPrice) {   // state change → broadcast
        if (newPrice == price) return;         // guard: no spurious notifications
        this.price = newPrice;
        notifyObservers();
    }
}
```

### Step 3 — concrete Observers, each reacting independently
```java
class MobileAppView implements PriceObserver {
    public void update(String symbol, double price) {
        System.out.println("[App] " + symbol + " = " + price);
    }
}
class EmailAlerter implements PriceObserver {
    private final double threshold;
    EmailAlerter(double threshold) { this.threshold = threshold; }
    public void update(String symbol, double price) {
        if (price >= threshold) System.out.println("[Email] " + symbol + " hit " + price);
    }
}
class AnalyticsLogger implements PriceObserver {
    public void update(String symbol, double price) { /* persist tick */ }
}
```

### Step 4 — wire it up
```java
Stock infy = new Stock("INFY", 1500);
PriceObserver app = new MobileAppView();
infy.subscribe(app);
infy.subscribe(new EmailAlerter(1600));
infy.subscribe(new AnalyticsLogger());

infy.setPrice(1620);   // all three notified; email fires (>=1600)
infy.unsubscribe(app); // app leaves at runtime
infy.setPrice(1550);   // only email + analytics notified now
```

### 🔍 Where Observer is doing the work (the highlight)
- `Stock` holds a list of the **`PriceObserver` interface**, never concrete observers — so it has no idea an email or an app exists. That decoupling *is* the Observer pattern → **loose coupling / DIP**.
- A new reactor (e.g. `SlackAlerter`) = **one new class** + `subscribe(...)`, with **zero edits** to `Stock` → **Open/Closed**.
- Observers **join and leave at runtime** via subscribe/unsubscribe.
- Note the two interview-grade safety touches: `CopyOnWriteArrayList` (no `ConcurrentModificationException` if an observer unsubscribes mid-broadcast) and the equality **guard** in `setPrice` (avoids redundant/cyclic notifications).
- The `EmailAlerter`'s threshold check is a whiff of **Strategy** (how it reacts), but the *delivery* — one change fanning out to many — is pure Observer.

---

## 13. 60-second recall summary

- **Problem:** when one object changes, a variable set of others must react, without tight coupling.
- **Fix:** Subject keeps a list of Observers (by interface), broadcasts `update()` on state change.
- **Models:** push (send data) vs pull (send "changed", observer fetches).
- **Wins:** loose coupling, Open/Closed, runtime subscribe/unsubscribe, SRP.
- **Watch out:** concurrent modification, memory leaks, cyclic updates, blocking observers.
- **Bigger cousin:** Pub/Sub via a message broker (async, cross-process).
- **Real example to cite:** Swing listeners, `PropertyChangeListener` (and note `java.util.Observable` is deprecated).

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅
2. Factory / Factory Method ✅
3. Observer ✅ (this one)
4. State (elevator, vending machine) ← next
5. Decorator (pizza/coffee builder questions)
6. Builder & Singleton (used everywhere, easy wins)
