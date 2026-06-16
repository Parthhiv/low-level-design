# Command Pattern — LLD Notes (SDE 1 / FAANG)

> Behavioral design pattern. "Turn a request into a standalone object."
> The enabler for **undo/redo**, queuing, scheduling, and logging of operations.

---

## 1. The one-line idea

**Encapsulate a request as an object — bundling the action, its receiver, and its parameters — so you can pass it around, queue it, log it, and undo it.**

The caller (Invoker) holds a `Command` and just calls `execute()`; it has no idea what the command actually does.

---

## 2. The problem it solves (intuition)

You want to trigger operations, but also to **store, queue, schedule, log, or undo** them — and to decouple the thing that *triggers* an action from the thing that *performs* it:

- A remote control button should run *some* action, configurable per button.
- A text editor needs **undo/redo** of every edit.
- A job/task queue must hold operations to run later, possibly retry them.

The naive fix hard-codes the action into the trigger:

```java
// SMELL: the button KNOWS the receiver and the exact method → rigid, no undo, no queue
class Button {
    void onClick() {
        light.turnOn();   // hard-wired; can't reassign, can't undo, can't log generically
    }
}
```

Why it's bad:
- The invoker is **coupled** to a specific receiver + method.
- You **can't reassign** what a button does at runtime.
- **No uniform way** to queue/log/undo — every action is special-cased.

Command makes every operation a uniform object with `execute()` (and often `undo()`).

---

## 3. Structure

```
   Client ──configures──▶ ConcreteCommand(receiver, args)
                                  │ implements
   ┌───────────────┐    holds     ▼
   │   Invoker     │──────▶ ┌──────────────────┐        calls
   │  (Button,     │        │  <<interface>>   │     ┌────────────────┐
   │   Remote,     │        │  Command         │────▶│   Receiver     │
   │   Queue)      │        │  + execute()     │     │ (Light, Editor)│
   │  + press()    │        │  + undo()        │     │  + actualWork()│
   └───────────────┘        └──────────────────┘     └────────────────┘
```

| Role | What it is | Example |
|------|-----------|---------|
| **Command** | Interface with `execute()` (and `undo()`) | `Command` |
| **ConcreteCommand** | Binds a Receiver + params to an action | `TurnOnCommand` |
| **Receiver** | Knows how to do the actual work | `Light`, `Document` |
| **Invoker** | Triggers the command, doesn't know its internals | `RemoteControl`, queue |
| **Client** | Creates commands and wires them to invokers | `main()` |

---

## 4. Canonical code example (Java)

```java
// 1. Command interface
interface Command {
    void execute();
    void undo();
}

// 2. Receiver — knows the real work
class Light {
    void on()  { System.out.println("Light ON"); }
    void off() { System.out.println("Light OFF"); }
}

// 3. Concrete commands — bind receiver + action, and know how to reverse it
class TurnOnCommand implements Command {
    private final Light light;
    TurnOnCommand(Light light) { this.light = light; }
    public void execute() { light.on(); }
    public void undo()    { light.off(); }
}
class TurnOffCommand implements Command {
    private final Light light;
    TurnOffCommand(Light light) { this.light = light; }
    public void execute() { light.off(); }
    public void undo()    { light.on(); }
}

// 4. Invoker — holds commands, supports undo via a history stack
class RemoteControl {
    private final Deque<Command> history = new ArrayDeque<>();
    void submit(Command c) { c.execute(); history.push(c); }
    void undoLast() { if (!history.isEmpty()) history.pop().undo(); }
}

// 5. Client wires it up
public class Main {
    public static void main(String[] args) {
        Light light = new Light();
        RemoteControl remote = new RemoteControl();

        remote.submit(new TurnOnCommand(light));   // Light ON
        remote.submit(new TurnOffCommand(light));  // Light OFF
        remote.undoLast();                          // Light ON  (undo of OFF)
    }
}
```

The invoker treats every operation uniformly — which is exactly what makes undo, queuing, and logging possible.

---

## 5. When to use it

- You need **undo/redo**.
- You want to **queue, schedule, or log** operations, or run them later/remotely.
- You want to **decouple** the trigger from the action (configurable buttons, menu items, hotkeys).
- You need **macro commands** (composite operations) or retryable jobs.

## When NOT to use it

- The action is a **simple, fixed** call with no need for undo/queue/log → just call it.
- Adding a Command class per trivial action is **ceremony** with no payoff.
- A lambda/`Runnable` already suffices (Java treats Command-ish things as functional interfaces).

---

## 6. Real systems that use Command (good to name-drop)

- **`java.lang.Runnable` / `Callable`** — a task as an object; what thread pools execute. The everyday Command.
- **`java.util.concurrent` `ExecutorService.submit(task)`** — queued commands.
- **Swing `Action`** — encapsulates menu/button operations.
- **Undo/redo** in editors, IDEs, Photoshop.
- **Job queues** (Sidekiq/Celery-style), **transactional outbox**, **CQRS commands**.

Saying "a `Runnable` submitted to an executor is the Command pattern" is a clean real-world anchor.

---

## 7. Command vs. lookalike patterns (interviewers probe this)

| Pattern | Core difference |
|---------|-----------------|
| **Strategy** | Strategy encapsulates **how** to do one thing (an algorithm) chosen by the context; Command encapsulates **a request to do something** (action + receiver + args), to be stored/queued/undone. Strategy is about interchangeability; Command is about decoupling + bookkeeping. |
| **Observer** | Observer broadcasts events to many; Command is a single encapsulated request handed to an invoker. |
| **Chain of Responsibility** | CoR passes one request along handlers until handled; Command wraps the request itself. They combine: a Command can travel a chain. |
| **Memento** | Often paired for undo: Command triggers the change; Memento stores the prior state to restore. |

---

## 8. How it shows up in FAANG LLD rounds

- **Text editor / IDE** → undo/redo stack of edit commands (very common).
- **Remote control / smart home** → configurable buttons, macros.
- **Task scheduler / job queue** → commands enqueued and executed by workers.
- **Vending machine / ATM** → operations as commands (sometimes).
- **Game move system** → moves as commands (enables replay/undo).
- **Transactional systems** → commands logged for replay/audit.

**Where to introduce it:** when the prompt mentions **undo/redo, queuing, scheduling, logging of actions, or "configurable" triggers**. Say: *"I'll model each operation as a Command with execute/undo, so the invoker can keep a history stack and we get undo and queuing for free."*

---

## 9. How to talk about it (scoring phrases)

- "Each operation is an object with `execute()`/`undo()`, so the invoker handles them **uniformly**."
- "The invoker is **decoupled** from the receiver — it doesn't know what the command does."
- "A history stack of commands gives **undo/redo** naturally."
- "Commands can be **queued, logged, scheduled, or replayed** — same as a `Runnable` on an executor."

---

## 10. Common mistakes (avoid these in the room)

1. **Confusing it with Strategy** — Strategy = interchangeable algorithm; Command = a stored, undoable request.
2. **Putting the real logic in the Command** instead of the **Receiver** — the command should *delegate*; the receiver does the work.
3. **Undo that doesn't capture enough state** — to reverse, the command (or a Memento) must remember what it needs.
4. **Stateful commands reused** across executions causing incorrect undo — create per-invocation or snapshot state.
5. **Over-applying it** for one-off trivial actions where a lambda would do.

---

## 11. Worked problem — a text editor with undo/redo

**Prompt:** *Design the edit operations of a text editor supporting `type`, `delete`, and unlimited **undo/redo**. Each action must be reversible.*

Command is the highlighted pattern. (**Memento** could store snapshots for undo; here we keep the reverse logic inside each command, which is the Command-centric approach.)

### Step 1 — the receiver that does the real work
```java
class Document {
    private final StringBuilder text = new StringBuilder();
    void insert(String s)           { text.append(s); }
    void deleteLast(int n)          { text.delete(text.length() - n, text.length()); }
    String content()                { return text.toString(); }
}
```

### Step 2 — Command interface with execute + undo
```java
interface Command {
    void execute();
    void undo();
}
```

### Step 3 — concrete commands that know how to reverse themselves
```java
class TypeCommand implements Command {
    private final Document doc;
    private final String text;
    TypeCommand(Document doc, String text) { this.doc = doc; this.text = text; }
    public void execute() { doc.insert(text); }
    public void undo()    { doc.deleteLast(text.length()); }   // reverse of insert
}
class DeleteCommand implements Command {
    private final Document doc;
    private final int n;
    private String removed;                  // remember what we deleted, to restore
    DeleteCommand(Document doc, int n) { this.doc = doc; this.n = n; }
    public void execute() {
        String c = doc.content();
        removed = c.substring(c.length() - n);
        doc.deleteLast(n);
    }
    public void undo() { doc.insert(removed); }   // restore the captured text
}
```

### Step 4 — the invoker manages undo + redo stacks
```java
class Editor {
    private final Document doc = new Document();
    private final Deque<Command> undoStack = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();

    void run(Command c) { c.execute(); undoStack.push(c); redoStack.clear(); }
    void undo() { if (!undoStack.isEmpty()) { Command c = undoStack.pop(); c.undo(); redoStack.push(c); } }
    void redo() { if (!redoStack.isEmpty()) { Command c = redoStack.pop(); c.execute(); undoStack.push(c); } }
    String content() { return doc.content(); }
    Document doc()   { return doc; }
}
```

### Step 5 — drive it
```java
Editor editor = new Editor();
Document d = editor.doc();
editor.run(new TypeCommand(d, "Hello"));   // "Hello"
editor.run(new TypeCommand(d, " World"));  // "Hello World"
editor.undo();                              // "Hello"        (undo of " World")
editor.redo();                              // "Hello World"  (redo)
editor.run(new DeleteCommand(d, 6));        // "Hello"
editor.undo();                              // "Hello World"  (delete restored)
```

### 🔍 Where Command is doing the work (the highlight)
- Every edit is a **first-class object** implementing `execute()`/`undo()`. Because they're uniform objects, the `Editor` can shove them onto stacks without knowing whether it's a type or a delete — that uniformity *is* the Command pattern, and it's exactly what makes generic **undo/redo** possible.
- The **invoker (`Editor`) is decoupled from the receiver (`Document`)**: it never calls `insert`/`deleteLast` directly — it just calls `execute()`/`undo()` on whatever command it holds.
- Each command **carries the state needed to reverse itself** (`DeleteCommand` remembers the removed text). That self-contained reversibility is why we don't need a central switch over action types.
- The two stacks turn a pile of command objects into full undo *and* redo history — add logging or a job queue and the *same* command objects would serialize/replay unchanged. (If we'd instead snapshotted the whole document per step, that'd be the **Memento** variant — same goal, different trade-off.)

---

## 12. 60-second recall summary

- **Problem:** you need to queue/log/schedule/undo operations and decouple the trigger from the action.
- **Fix:** wrap each request as a Command object (`execute()`/`undo()`); an Invoker runs them uniformly; a Receiver does the real work.
- **Wins:** undo/redo via history stacks, queuing, logging, macros, decoupling.
- **Distinguish:** Strategy = interchangeable algorithm; Command = stored, undoable request. Pairs with Memento for undo.
- **Real example to cite:** `Runnable`/`Callable` submitted to an `ExecutorService`.

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅ · 2. Factory ✅ · 3. Observer ✅ · 4. State ✅ · 5. Decorator ✅ · 6. Builder ✅ · 7. Singleton ✅ · 8. Adapter ✅
9. Command ✅ (this one) — then Chain of Responsibility / Facade
