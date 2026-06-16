# State Pattern — LLD Notes (SDE 1 / FAANG)

> Behavioral design pattern. The answer to "this object behaves differently depending on what mode it's in."
> Structurally the **twin of Strategy** — same shape, different intent. Know the difference cold; interviewers love this trap.

---

## 1. The one-line idea

**Let an object change its behavior when its internal state changes — by delegating to a separate state object — so it looks like the object changed its class.**

Each state is its own class. The object (Context) holds a reference to "current state" and forwards work to it. States can hand control to the next state.

---

## 2. The problem it solves (intuition)

An object acts differently depending on a mode, and the modes have **transitions** between them:

- A vending machine behaves differently when *idle*, *coin inserted*, *dispensing*, *sold out*.
- A document is *draft*, *moderation*, *published*.
- A traffic light cycles *green → yellow → red → green*.

The naive fix is conditionals on a state field, *everywhere*:

```java
// SMELL: every method is a switch over the same state field
class VendingMachine {
    String state = "IDLE";
    void insertCoin() {
        if (state.equals("IDLE"))            { state = "HAS_COIN"; }
        else if (state.equals("HAS_COIN"))   { System.out.println("Coin already in"); }
        else if (state.equals("SOLD_OUT"))   { System.out.println("Refunding"); }
        // ...repeated in selectProduct(), dispense(), refund()...
    }
}
```

Why it's bad:
- **The same `switch` is duplicated** across every method.
- **Transitions are scattered** and hard to reason about — easy to reach an illegal state.
- **Open/Closed violation** — adding a `MAINTENANCE` state edits every method.
- It's a giant **finite state machine** smeared across one class.

State puts each mode in its own class that knows its own behavior *and* its valid transitions.

---

## 3. Structure

```
        ┌────────────────────────────┐
        │        Context             │       delegates every request
        │     (VendingMachine)       │────────────────────────────┐
        │   - state: State           │                             │
        │   + setState(State)        │                             ▼
        │   + insertCoin() ──────────┼──▶ state.insertCoin(this)   ┌──────────────────┐
        └────────────────────────────┘                            │  <<interface>>   │
                                                                  │  State           │
                                                                  │  + insertCoin(ctx)│
                                                                  │  + selectItem(ctx)│
                                                                  │  + dispense(ctx) │
                                                                  └────────┬─────────┘
                                          implements                       │
                       ┌──────────────────┬──────────────────┬────────────┘
                       ▼                  ▼                  ▼
                  IdleState         HasCoinState       SoldOutState
              (transitions to    (transitions to     (no transitions)
               HasCoinState)      Dispensing/Idle)
```

| Role | What it is | Example |
|------|-----------|---------|
| **Context** | Holds current State, delegates to it | `VendingMachine` |
| **State** | Interface for state-specific behavior | `State` |
| **ConcreteState** | One mode; implements behavior + triggers transitions | `IdleState`, `HasCoinState` |

**Key:** ConcreteStates usually call `context.setState(next)` to drive transitions. The Context exposes a `setState` so states can move it along.

---

## 4. Canonical code example (Java)

```java
// 1. State interface — every action the machine supports
interface State {
    void insertCoin(VendingMachine m);
    void selectItem(VendingMachine m);
    void dispense(VendingMachine m);
}

// 2. Context
class VendingMachine {
    private State state;
    private int itemCount;
    VendingMachine(int itemCount) {
        this.itemCount = itemCount;
        this.state = itemCount > 0 ? new IdleState() : new SoldOutState();
    }
    void setState(State s) { this.state = s; }
    int  getItemCount()    { return itemCount; }
    void reduceItem()      { itemCount--; }

    // public API just delegates — NO if/else anywhere
    void insertCoin() { state.insertCoin(this); }
    void selectItem() { state.selectItem(this); }
    void dispense()   { state.dispense(this); }
}

// 3. Concrete states — each owns its behavior AND its transitions
class IdleState implements State {
    public void insertCoin(VendingMachine m) {
        System.out.println("Coin accepted.");
        m.setState(new HasCoinState());          // transition
    }
    public void selectItem(VendingMachine m) { System.out.println("Insert a coin first."); }
    public void dispense(VendingMachine m)   { System.out.println("Pay first."); }
}

class HasCoinState implements State {
    public void insertCoin(VendingMachine m) { System.out.println("Coin already inserted."); }
    public void selectItem(VendingMachine m) {
        System.out.println("Item selected.");
        m.dispense();                            // proceed to dispense
    }
    public void dispense(VendingMachine m) {
        m.reduceItem();
        System.out.println("Item dispensed.");
        m.setState(m.getItemCount() > 0 ? new IdleState() : new SoldOutState());
    }
}

class SoldOutState implements State {
    public void insertCoin(VendingMachine m) { System.out.println("Sold out. Refunding coin."); }
    public void selectItem(VendingMachine m) { System.out.println("Sold out."); }
    public void dispense(VendingMachine m)   { System.out.println("Sold out."); }
}

// 4. Client
public class Main {
    public static void main(String[] args) {
        VendingMachine m = new VendingMachine(1);
        m.insertCoin();   // Coin accepted → HasCoin
        m.selectItem();   // Item selected → dispense → SoldOut (count was 1)
        m.insertCoin();   // Sold out. Refunding coin.
    }
}
```

Each state is self-contained, and illegal actions are handled *locally* ("Insert a coin first") instead of by a far-away conditional.

---

## 5. When to use it

- An object's behavior depends on its **state**, and there are **many states with transitions** between them.
- You have large `switch`/`if-else` blocks on a state field repeated across multiple methods.
- You're modeling a **finite state machine** (workflow, lifecycle, protocol).
- Transitions are part of the logic and you want them explicit and localized.

## When NOT to use it

- Only **two or three** states with trivial behavior → a simple enum + `switch` may be clearer.
- The "states" don't change behavior, only data → just a field.
- There are no transitions, only interchangeable algorithms → that's **Strategy**, not State.

---

## 6. Real systems that use State (good to name-drop)

- **TCP connection** lifecycle (`LISTEN`, `ESTABLISHED`, `CLOSED`) — the classic example.
- **Thread state** in the JVM (`NEW`, `RUNNABLE`, `BLOCKED`, `TERMINATED`).
- **Order / payment lifecycle** (`CREATED → PAID → SHIPPED → DELIVERED`).
- **Media players** (`Stopped`, `Playing`, `Paused`).
- **Workflow / approval engines**, **UI wizards** (step state), **game character states** (idle/running/jumping).

---

## 7. State vs. lookalike patterns (interviewers probe this)

| Pattern | Core difference |
|---------|-----------------|
| **Strategy** | **Structurally identical.** Difference is **intent**: Strategy's variants are *independent* and chosen by the *client*; State's variants *know about each other* and the *states themselves drive transitions*. Strategy = "how I do this," State = "what I am right now." |
| **Finite State Machine (table)** | State pattern is one OO way to implement an FSM. For huge FSMs a transition **table/map** can be cleaner than a class per state. |
| **Observer** | Unrelated — Observer broadcasts events; State changes one object's behavior by mode. |

The single most important sentence to say: *"Same structure as Strategy, but here the states transition between each other, so it's State, not Strategy."*

---

## 8. How it shows up in FAANG LLD rounds

State is often the **core** pattern of a whole question:

- **Vending machine** → the textbook State question.
- **Elevator system** → moving up/down/idle/maintenance.
- **ATM** → card inserted / pin entered / dispensing / out-of-cash.
- **Traffic light controller** → cyclic states.
- **Order management / Tic-Tac-Toe game / Jira-style workflow** → lifecycle states.
- **Media player, gesture/UI state machines**.

**Where to introduce it:** when the prompt describes an object with distinct modes and rules about moving between them. Say: *"This is a finite state machine; I'll model each state as a class so behavior and transitions live together and adding a state doesn't touch the others."*

---

## 9. How to talk about it (scoring phrases)

- "Each state is a class owning its behavior **and** its valid transitions — no scattered `switch`."
- "Adding a new state doesn't modify existing states — **Open/Closed**."
- "Illegal actions are handled **locally** per state, so we can't drift into an invalid state."
- "Same shape as Strategy, but the **states transition between themselves** — that's the distinguishing intent."

---

## 10. Common mistakes (avoid these in the room)

1. **Calling it Strategy** (or vice-versa) — wrong vocabulary loses the point. State = transitions, Strategy = interchangeable algorithms.
2. **Putting transition logic back in the Context** as a `switch` — defeats the purpose; let states transition.
3. **Stateful state objects with shared mutable fields** — prefer stateless state classes (then a single shared instance per state, even singletons, is fine and saves allocations).
4. **God Context** that still contains all the behavior — push behavior into the state classes.
5. **Forgetting illegal transitions** — every state must answer every action, even if just to reject it.
6. **Over-modeling** trivial 2-state toggles as full State classes.

> Optimization to mention: since concrete states are usually stateless, cache them (`static final IDLE = new IdleState()`) and reuse instead of `new`-ing on every transition.

---

## 11. Worked problem — document publishing workflow

**Prompt:** *Model a CMS document lifecycle. A document is `Draft`, then goes to `Moderation`, then `Published`. Authors can edit a draft; only published docs are public; a rejected doc returns to draft. Behavior of `publish()`, `edit()`, and `reject()` differs per state.*

This is a pure **State** problem (highlighted). A **Factory** could create the initial state, but the heart is the state machine.

### Step 1 — state interface for the document's actions
```java
interface DocState {
    void edit(Document d, String content);
    void publish(Document d);
    void reject(Document d);
}
```

### Step 2 — Context holds current state + delegates
```java
class Document {
    private DocState state = new DraftState();   // initial state
    private String content = "";
    void setState(DocState s) { this.state = s; }
    void setContent(String c) { this.content = c; }
    String getContent()       { return content; }

    void edit(String c)  { state.edit(this, c); }
    void publish()       { state.publish(this); }
    void reject()        { state.reject(this); }
}
```

### Step 3 — concrete states own behavior + transitions
```java
class DraftState implements DocState {
    public void edit(Document d, String c) { d.setContent(c); System.out.println("Draft updated."); }
    public void publish(Document d)        { System.out.println("Sent to moderation."); d.setState(new ModerationState()); }
    public void reject(Document d)         { System.out.println("Already a draft."); }
}
class ModerationState implements DocState {
    public void edit(Document d, String c) { System.out.println("Can't edit while in moderation."); }
    public void publish(Document d)        { System.out.println("Published!"); d.setState(new PublishedState()); }
    public void reject(Document d)         { System.out.println("Rejected → back to draft."); d.setState(new DraftState()); }
}
class PublishedState implements DocState {
    public void edit(Document d, String c) { System.out.println("Unpublish before editing."); }
    public void publish(Document d)        { System.out.println("Already published."); }
    public void reject(Document d)         { System.out.println("Taken down → draft."); d.setState(new DraftState()); }
}
```

### Step 4 — drive it
```java
Document doc = new Document();
doc.edit("v1 content");  // Draft updated.
doc.publish();           // Sent to moderation.  (Draft → Moderation)
doc.edit("sneaky");      // Can't edit while in moderation.
doc.publish();           // Published!            (Moderation → Published)
doc.reject();            // Taken down → draft.    (Published → Draft)
```

### 🔍 Where State is doing the work (the highlight)
- `Document` has **no `if/else`** — it blindly delegates `edit/publish/reject` to whatever `DocState` it currently holds. The object "changes its class" as the state swaps.
- Each transition (`Draft → Moderation → Published`, plus the reject paths) lives **inside the state that owns it**, not in a central switch — so the FSM is explicit and local.
- Adding a `ScheduledState` = **one new class**; existing states stay untouched → **Open/Closed**.
- This is exactly what distinguishes it from Strategy: the states **know the next state** and transition the Context themselves. A Strategy never says "now become this other strategy."

---

## 12. 60-second recall summary

- **Problem:** an object behaves differently per mode, with transitions, and `switch`-on-state is duplicated across methods.
- **Fix:** one class per state implementing a common interface; Context delegates and states drive transitions via `setState`.
- **Wins:** localized behavior + transitions, Open/Closed, no illegal-state drift.
- **Twin to know:** Strategy — identical structure, but Strategy variants are independent and client-chosen; State variants transition between themselves.
- **Optimize:** stateless state classes → cache/share single instances.
- **Real example to cite:** TCP connection states, JVM thread states, order lifecycle.

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅
2. Factory / Factory Method ✅
3. Observer ✅
4. State ✅ (this one)
5. Decorator (pizza/coffee builder questions) ← next
6. Builder & Singleton (used everywhere, easy wins)
