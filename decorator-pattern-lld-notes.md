# Decorator Pattern — LLD Notes (SDE 1 / FAANG)

> Structural design pattern. The answer to "add responsibilities to an object dynamically, without subclassing for every combination."
> The classic coffee-shop / pizza-topping question is *always* Decorator.

---

## 1. The one-line idea

**Wrap an object in another object that shares its interface and adds behavior — stack wrappers to compose features at runtime, instead of creating a subclass for every combination.**

A decorator *is-a* component (same interface) and *has-a* component (wraps one). It delegates to the inner object and adds something before/after.

---

## 2. The problem it solves (intuition)

You have a base thing and many optional add-ons that can combine freely:

- A coffee can have milk, soy, whip, caramel… in any combination.
- A pizza can have cheese, olives, jalapeños…
- A text stream can be buffered, then compressed, then encrypted.

The naive fix is a subclass per combination — **class explosion**:

```java
// SMELL: one class per combination → exponential blow-up
class Espresso {}
class EspressoWithMilk {}
class EspressoWithMilkAndWhip {}
class EspressoWithSoyAndCaramelAndWhip {}   // ...this never ends
```

Why it's bad:
- **Combinatorial explosion** — N add-ons → up to 2^N classes.
- **Open/Closed violation** — a new add-on multiplies the class count.
- **Rigid** — combinations are fixed at compile time; you can't build one from user choices at runtime.

Decorator turns each add-on into **one wrapper class** you can stack in any order.

---

## 3. Structure

```
        ┌────────────────────┐
        │   <<interface>>    │
        │   Beverage         │◀───────────────┐
        │   + cost()         │                │ wraps (has-a)
        │   + description()  │                │
        └─────────┬──────────┘                │
       implements │                           │
        ┌─────────┴─────────┐        ┌─────────┴──────────────┐
        ▼                   ▼        │   <<abstract>>          │
   Espresso            HouseBlend    │  BeverageDecorator      │
   (ConcreteComponent)               │  - Beverage inner       │
                                     │  + cost() = inner.cost()│
                                     └─────────┬───────────────┘
                                       extends │
                              ┌────────────────┼───────────────┐
                              ▼                ▼               ▼
                         MilkDecorator   WhipDecorator   CaramelDecorator
                         (+10, +" Milk") (+15)           (+20)
```

| Role | What it is | Example |
|------|-----------|---------|
| **Component** | Common interface | `Beverage` |
| **ConcreteComponent** | The base object being decorated | `Espresso` |
| **Decorator** | Abstract wrapper holding a Component | `BeverageDecorator` |
| **ConcreteDecorator** | Adds one responsibility | `MilkDecorator`, `WhipDecorator` |

**Key:** decorator and component share the interface, so a wrapped object is still usable anywhere the component is — and wrappers can wrap wrappers.

---

## 4. Canonical code example (Java)

```java
// 1. Component
interface Beverage {
    String description();
    double cost();
}

// 2. Concrete component (the base)
class Espresso implements Beverage {
    public String description() { return "Espresso"; }
    public double cost()        { return 80; }
}

// 3. Abstract decorator — IS-A Beverage, HAS-A Beverage
abstract class BeverageDecorator implements Beverage {
    protected final Beverage inner;
    BeverageDecorator(Beverage inner) { this.inner = inner; }
}

// 4. Concrete decorators — each adds one thing, delegating to inner
class Milk extends BeverageDecorator {
    Milk(Beverage b) { super(b); }
    public String description() { return inner.description() + " + Milk"; }
    public double cost()        { return inner.cost() + 10; }
}
class Whip extends BeverageDecorator {
    Whip(Beverage b) { super(b); }
    public String description() { return inner.description() + " + Whip"; }
    public double cost()        { return inner.cost() + 15; }
}
class Caramel extends BeverageDecorator {
    Caramel(Beverage b) { super(b); }
    public String description() { return inner.description() + " + Caramel"; }
    public double cost()        { return inner.cost() + 20; }
}

// 5. Client stacks wrappers at runtime
public class Main {
    public static void main(String[] args) {
        Beverage order = new Caramel(new Whip(new Milk(new Espresso())));
        System.out.println(order.description()); // Espresso + Milk + Whip + Caramel
        System.out.println(order.cost());        // 80 + 10 + 15 + 20 = 125.0
    }
}
```

Adding "SoyMilk" later = **one new decorator class** and zero changes to existing code. Any combination is just nesting.

---

## 5. When to use it

- You need to add responsibilities to **individual objects**, not a whole class, and do it **at runtime**.
- Subclassing would cause a **combinatorial explosion** of feature combinations.
- Features should be **stackable** and **order-flexible**.
- You want to keep each add-on as a **single-responsibility** unit.

## When NOT to use it

- The add-ons are **fixed and few** — plain fields/flags may be simpler.
- Behavior depends on **deep interaction** between decorators (they should be largely independent).
- Code relies on the **concrete type** of the object — decorators hide it (you only get the interface).

---

## 6. Real systems that use Decorator (good to name-drop)

- **Java I/O streams** — `new BufferedReader(new InputStreamReader(new FileInputStream(f)))`. The *cleanest* real example: each wraps the next, adding behavior. Cite this and the interviewer knows you get it.
- **`java.util.Collections.synchronizedList` / `unmodifiableList`** — wrap a list to add behavior.
- **Servlet `HttpServletRequestWrapper`**, Spring's `BeanWrapper`.
- **`javax.swing` borders**, UI component decoration.
- **Web middleware** (each layer wraps the handler) — conceptually the same.

---

## 7. Decorator vs. lookalike patterns (interviewers probe this)

| Pattern | Core difference |
|---------|-----------------|
| **Inheritance / subclassing** | Compile-time, one combination per subclass. Decorator is runtime + composable. "Favor composition over inheritance" — this is the example. |
| **Proxy** | Same shape (wraps an object, same interface) but **intent** differs: Proxy *controls access* (lazy load, security, remoting) and usually wraps **one fixed** subject; Decorator *adds behavior* and is meant to **stack**. |
| **Adapter** | Adapter **changes** the interface to a different one; Decorator **keeps the same** interface and adds behavior. |
| **Composite** | Composite composes objects into trees (part-whole); Decorator wraps a single object to enhance it. Similar recursion, different goal. |
| **Strategy** | Strategy changes the *guts* (the algorithm) of an object; Decorator wraps the object from *outside* to add layers. |

---

## 8. How it shows up in FAANG LLD rounds

- **Coffee shop / Starbucks** → beverages + condiments (the textbook prompt).
- **Pizza ordering** → base + toppings.
- **Notification with add-ons** → base message decorated with formatting/encryption/logging.
- **Text editor** → text with bold/italic/underline as stackable decorators.
- **Pricing with discounts/taxes/fees** → wrap a base price (though Strategy/Chain also fit).
- **I/O or data pipelines** → buffering/compression/encryption layers.

**Where to introduce it:** when you see "base item + optional add-ons in any combination, priced/described cumulatively." Say: *"Subclassing every combination explodes; I'll use Decorator so each add-on is one class and orders are composed at runtime."*

---

## 9. How to talk about it (scoring phrases)

- "Each add-on is one class — **Single Responsibility** — and adding one doesn't touch existing code — **Open/Closed**."
- "Decorators share the component interface, so a wrapped object is a drop-in — **Liskov**."
- "This is **composition over inheritance**; it kills the 2^N subclass explosion."
- "Same idea as Java's `BufferedInputStream` wrapping a `FileInputStream`."

---

## 10. Common mistakes (avoid these in the room)

1. **Confusing it with Proxy** — both wrap; Decorator *adds behavior and stacks*, Proxy *controls access*.
2. **Decorators depending on concrete inner types** — depend on the **interface** so any wrapper works under any wrapper.
3. **Order-sensitive decorators that secretly conflict** — keep them independent; if order changes the result unexpectedly, rethink.
4. **Losing the underlying type when the client needs it** — decorators mask the concrete object; don't use it where downcasting is required.
5. **Too many tiny decorators** for things that could be simple parameters — adds indirection for little gain.

---

## 11. Worked problem — a pizza ordering system

**Prompt:** *Price a pizza: a base (Margherita / Farmhouse) plus any combination of toppings (extra cheese, olives, jalapeños). Show the full description and total. New toppings are added all the time.*

Pure **Decorator** (highlighted). (A **Factory** could create the base pizza, and a **Builder** could assemble a fixed combo, but free-form stackable toppings = Decorator.)

### Step 1 — component interface
```java
interface Pizza {
    String description();
    double cost();
}
```

### Step 2 — concrete base pizzas
```java
class Margherita implements Pizza {
    public String description() { return "Margherita"; }
    public double cost()        { return 200; }
}
class Farmhouse implements Pizza {
    public String description() { return "Farmhouse"; }
    public double cost()        { return 280; }
}
```

### Step 3 — abstract topping decorator
```java
abstract class ToppingDecorator implements Pizza {
    protected final Pizza pizza;          // wraps a Pizza, IS a Pizza
    ToppingDecorator(Pizza p) { this.pizza = p; }
}
```

### Step 4 — concrete toppings (one class each)
```java
class ExtraCheese extends ToppingDecorator {
    ExtraCheese(Pizza p) { super(p); }
    public String description() { return pizza.description() + ", Extra Cheese"; }
    public double cost()        { return pizza.cost() + 40; }
}
class Olives extends ToppingDecorator {
    Olives(Pizza p) { super(p); }
    public String description() { return pizza.description() + ", Olives"; }
    public double cost()        { return pizza.cost() + 30; }
}
class Jalapenos extends ToppingDecorator {
    Jalapenos(Pizza p) { super(p); }
    public String description() { return pizza.description() + ", Jalapenos"; }
    public double cost()        { return pizza.cost() + 25; }
}
```

### Step 5 — compose at runtime (from user choices)
```java
Pizza order = new Jalapenos(new ExtraCheese(new Margherita()));
System.out.println(order.description()); // Margherita, Extra Cheese, Jalapenos
System.out.println(order.cost());        // 200 + 40 + 25 = 265.0
```

### 🔍 Where Decorator is doing the work (the highlight)
- Each topping is **one wrapper class** that *is* a `Pizza` and *holds* a `Pizza`. It delegates `cost()`/`description()` to the inner pizza and adds its own bit — that wrap-and-delegate *is* the Decorator pattern.
- Any combination is just **nesting constructors**; there is no `MargheritaWithCheeseAndOlives` class. We dodged the 2^N **class explosion**.
- A new "Paneer" topping = **one new decorator**, zero edits to existing pizzas or toppings → **Open/Closed**, **Single Responsibility**.
- Because every wrapper exposes the `Pizza` interface, the client treats a fully-loaded pizza exactly like a plain one — the recursion is invisible to callers.

---

## 12. 60-second recall summary

- **Problem:** base object + freely combinable optional features; subclassing every combo explodes.
- **Fix:** wrappers that share the component interface, each adding one responsibility, stackable at runtime.
- **Wins:** no class explosion, Open/Closed, SRP, composition over inheritance.
- **Twin to distinguish:** Proxy (controls access, one subject) vs Decorator (adds behavior, stacks); Adapter (changes interface) vs Decorator (keeps interface).
- **Real example to cite:** `BufferedReader(new InputStreamReader(new FileInputStream(...)))`.

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅
2. Factory / Factory Method ✅
3. Observer ✅
4. State ✅
5. Decorator ✅ (this one)
6. Builder & Singleton (used everywhere, easy wins) ← next
