# Chain of Responsibility Pattern — LLD Notes (SDE 1 / FAANG)

> Behavioral design pattern. "Pass a request along a chain of handlers until one handles it."
> The pattern behind middleware pipelines, approval workflows, and tiered processing.

---

## 1. The one-line idea

**Give a request a chain of potential handlers; each handler either handles it or passes it to the next — so the sender doesn't know or care which handler ends up dealing with it.**

Handlers are linked in a line. Each decides: handle, pass on, or both.

---

## 2. The problem it solves (intuition)

A request must go through several checks/handlers, and which one acts depends on the request — and you want to add/reorder/remove handlers freely:

- An expense approval: team lead approves ≤ ₹10k, manager ≤ ₹1L, director above.
- An ATM dispenses using ₹2000 → ₹500 → ₹100 notes in order.
- A web request passes through auth → rate-limit → logging → handler (middleware).
- Logging: DEBUG, INFO, ERROR handlers each decide whether to act.

The naive fix is one fat method with a conditional ladder:

```java
// SMELL: one method knows every level + every threshold → rigid, unreorderable
void approve(double amount) {
    if (amount <= 10_000)      teamLead.sign();
    else if (amount <= 100_000) manager.sign();
    else                        director.sign();
}
```

Why it's bad:
- All approval knowledge is **centralized and coupled**.
- **Reordering or inserting** a level (e.g. a new "VP" tier) edits this method.
- The sender is **coupled** to every handler and every threshold.

Chain of Responsibility makes each handler an independent link you can compose.

---

## 3. Structure

```
   Client ──request──▶ Handler1 ──next──▶ Handler2 ──next──▶ Handler3 ──next──▶ null
                          │                  │                  │
                       handle?            handle?            handle?
                      (else pass)        (else pass)        (else pass)
```

```
        ┌──────────────────────────┐
        │   <<abstract>> Handler   │
        │   - next: Handler        │◀─┐ holds the next link
        │   + setNext(Handler)     │  │
        │   + handle(request)      │──┘ default: delegate to next
        └────────────┬─────────────┘
           extends    │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
   TeamLead      Manager       Director
   (≤10k)        (≤1L)         (else)
```

| Role | What it is | Example |
|------|-----------|---------|
| **Handler** | Interface/abstract with `handle()` + a `next` link | `Approver` |
| **ConcreteHandler** | Handles what it can, else passes on | `TeamLead`, `Manager` |
| **Client** | Builds the chain and sends the request to the head | `main()` |

---

## 4. Canonical code example (Java)

```java
// 1. Handler — base class holding the link + default pass-through
abstract class Approver {
    protected Approver next;
    public Approver setNext(Approver next) { this.next = next; return next; } // fluent linking

    public void handle(double amount) {
        if (canApprove(amount)) {
            approve(amount);
        } else if (next != null) {
            next.handle(amount);           // pass along
        } else {
            System.out.println("No one could approve " + amount);
        }
    }
    protected abstract boolean canApprove(double amount);
    protected abstract void approve(double amount);
}

// 2. Concrete handlers — each owns ONLY its own rule
class TeamLead extends Approver {
    protected boolean canApprove(double a) { return a <= 10_000; }
    protected void approve(double a) { System.out.println("TeamLead approved " + a); }
}
class Manager extends Approver {
    protected boolean canApprove(double a) { return a <= 100_000; }
    protected void approve(double a) { System.out.println("Manager approved " + a); }
}
class Director extends Approver {
    protected boolean canApprove(double a) { return true; }   // final catch-all
    protected void approve(double a) { System.out.println("Director approved " + a); }
}

// 3. Client builds the chain and fires requests at the head
public class Main {
    public static void main(String[] args) {
        Approver head = new TeamLead();
        head.setNext(new Manager()).setNext(new Director());  // TeamLead → Manager → Director

        head.handle(5_000);     // TeamLead approved 5000
        head.handle(50_000);    // Manager approved 50000
        head.handle(500_000);   // Director approved 500000
    }
}
```

Inserting a "VP" tier between Manager and Director is a new class + one `setNext` rewiring — the handlers themselves don't change.

---

## 5. When to use it

- A request may be handled by **one of several** handlers, decided at runtime.
- You want to **decouple sender from receiver** and let the set/order of handlers vary.
- You're building a **pipeline**: validation, middleware, filters, interceptors.
- Tiered/escalating processing (approval levels, support tiers, log levels).

## When NOT to use it

- There's exactly **one** handler, always → just call it.
- **Every** handler must always run in a fixed way → a simple loop/pipeline may be clearer than the pattern's "stop when handled" semantics.
- A request that goes **unhandled** would be a silent bug → ensure a catch-all or explicit "unhandled" path.

---

## 6. Real systems that use Chain of Responsibility (good to name-drop)

- **Servlet `Filter` chain** / **Spring `HandlerInterceptor`** — the canonical example (auth, logging, compression filters in a line).
- **`java.util.logging`** handlers / Log4j appender hierarchy — levels passed along.
- **Express.js / ASP.NET middleware**, **Netty `ChannelPipeline`**.
- **Exception handling** itself (a try/catch hierarchy is conceptually a chain).
- **AWS API Gateway / interceptor stacks**, **OkHttp `Interceptor` chain**.

"It's the same idea as a servlet filter chain or OkHttp interceptors" lands well.

---

## 7. CoR vs. lookalike patterns (interviewers probe this)

| Pattern | Core difference |
|---------|-----------------|
| **Decorator** | Both build a linked structure of wrappers. Decorator **always** passes through and **every** layer adds behavior (all run). CoR **stops** when a handler handles it (often only one acts). Intent: Decorator enhances; CoR finds a handler. |
| **Command** | Command wraps the **request** as an object; CoR is about **who handles** a request. They combine — a Command can travel a chain. |
| **Observer** | Observer notifies **all** observers (broadcast); CoR passes to **one at a time** until handled (and may stop). |
| **Mediator** | Mediator centralizes many-to-many comms through a hub; CoR is a linear pass-along. |

Key line: **CoR passes until *one* handles (and can stop); Observer broadcasts to *all*; Decorator runs *every* layer.**

---

## 8. How it shows up in FAANG LLD rounds

- **Logging framework** → log-level handlers (DEBUG/INFO/ERROR) — classic.
- **ATM cash dispenser** → ₹2000 → ₹500 → ₹100 note handlers (very common).
- **Expense/leave approval workflow** → escalating approver tiers.
- **Web server / API gateway** → middleware: auth → rate-limit → validation → handler.
- **Support ticket escalation** → L1 → L2 → L3.
- **Vending machine coin handling**, **event/exception handling pipelines**.

**Where to introduce it:** when the prompt describes **escalation, tiers, filters/middleware, or "try handlers in order until one works."** Say: *"I'll use a Chain of Responsibility so each handler owns one rule and passes the rest along — adding or reordering a tier doesn't touch the others."*

---

## 9. How to talk about it (scoring phrases)

- "Each handler owns **one** rule and passes the rest along — **Single Responsibility**."
- "The sender is **decoupled** from which handler acts."
- "Adding or reordering a handler doesn't modify the others — **Open/Closed**."
- "Same shape as a servlet **filter chain** or OkHttp interceptors."
- "I'll include a **catch-all** so no request goes silently unhandled."

---

## 10. Common mistakes (avoid these in the room)

1. **No terminal/catch-all handler** → a request falls off the end unhandled (silent bug).
2. **Confusing it with Decorator** — CoR stops when handled; Decorator runs every layer.
3. **Handlers knowing about each other's rules** — each should know only its own; linking is the client's job.
4. **Broken chain links** (forgot `setNext`) → requests silently dropped.
5. **Order-dependent bugs** — if correctness depends on order, make the wiring explicit and documented.
6. **Overusing it** when a simple `if/else` with two cases is clearer.

---

## 11. Worked problem — an ATM cash dispenser

**Prompt:** *Design an ATM that dispenses a withdrawal amount using the fewest notes, in denominations ₹2000, ₹500, ₹100. Each denomination handler dispenses as many of its note as it can, then passes the remainder on. New denominations should be easy to add.*

Chain of Responsibility is the highlighted pattern. (A **Factory**/**Builder** could assemble the chain; the dispensing logic itself is the chain.)

### Step 1 — the handler base with the link + pass-through
```java
abstract class CashHandler {
    protected CashHandler next;
    private final int denomination;
    private final int notesAvailable;

    protected CashHandler(int denomination, int notesAvailable) {
        this.denomination = denomination;
        this.notesAvailable = notesAvailable;
    }
    CashHandler setNext(CashHandler next) { this.next = next; return next; }

    void dispense(int amount) {
        if (amount <= 0) return;
        int needed = amount / denomination;
        int give   = Math.min(needed, notesAvailable);   // dispense what this handler can
        int remainder = amount - give * denomination;

        if (give > 0) System.out.println("Dispensing " + give + " x ₹" + denomination);

        if (remainder > 0) {
            if (next != null) next.dispense(remainder);    // pass the rest along
            else System.out.println("Cannot dispense remaining ₹" + remainder);  // terminal guard
        }
    }
}
```

### Step 2 — concrete denomination handlers (each owns ONE note)
```java
class TwoThousandHandler extends CashHandler { TwoThousandHandler(int n) { super(2000, n); } }
class FiveHundredHandler  extends CashHandler { FiveHundredHandler(int n)  { super(500, n);  } }
class HundredHandler      extends CashHandler { HundredHandler(int n)      { super(100, n);  } }
```

### Step 3 — build the chain high→low and dispense
```java
CashHandler atm = new TwoThousandHandler(5);
atm.setNext(new FiveHundredHandler(10))
   .setNext(new HundredHandler(20));

atm.dispense(5800);
// Dispensing 2 x ₹2000   (remainder 1800)
// Dispensing 3 x ₹500    (remainder 300)
// Dispensing 3 x ₹100    (remainder 0)
```

### 🔍 Where CoR is doing the work (the highlight)
- The withdrawal request **flows down a chain** of denomination handlers. Each handler does *only* its part (dispense as many of its note as possible) and **passes the remainder to the next** — no single method knows about all denominations. That pass-the-remainder linkage *is* the Chain of Responsibility.
- Each handler owns exactly **one rule** (its denomination + stock) → **Single Responsibility**; the ATM client just builds the chain and fires the request at the head, decoupled from how the work is split.
- Adding a ₹200 note = **one new handler class** + one `setNext` in the wiring; existing handlers are untouched → **Open/Closed**. Reordering denominations is just rewiring links.
- The terminal "cannot dispense remaining" branch is the **catch-all guard** — the thing beginners forget — so a request never falls off the end silently.
- Contrast with Decorator (which would run *every* layer to add behavior): here the chain is about **splitting/handling** a request as it travels, and a handler can end it when nothing remains.

---

## 12. 60-second recall summary

- **Problem:** a request may be handled by one of several handlers; you want to decouple sender from handler and vary the set/order.
- **Fix:** link handlers; each handles what it can and passes the rest to `next`.
- **Wins:** SRP per handler, Open/Closed (add/reorder freely), sender/receiver decoupling.
- **Distinguish:** CoR stops when handled; Decorator runs every layer; Observer broadcasts to all.
- **Always add:** a terminal/catch-all so nothing goes unhandled.
- **Real example to cite:** servlet `Filter` chain, OkHttp interceptors, Log4j appenders.

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅ · 2. Factory ✅ · 3. Observer ✅ · 4. State ✅ · 5. Decorator ✅ · 6. Builder ✅ · 7. Singleton ✅ · 8. Adapter ✅ · 9. Command ✅
10. Chain of Responsibility ✅ (this one) — then Facade
