# Adapter Pattern — LLD Notes (SDE 1 / FAANG)

> Structural design pattern. "Make an incompatible interface fit the one your code expects."
> The go-to answer for integrating a third-party library or legacy class you can't change.

---

## 1. The one-line idea

**Wrap an existing class so its interface looks like the interface the client expects — a translator between two incompatible APIs.**

Your code talks to a `Target` interface; the adapter implements `Target` and internally calls the incompatible `Adaptee`.

---

## 2. The problem it solves (intuition)

You have code that expects one interface, but the class you must use exposes a different one — and you **can't change** that class (third-party, legacy, or external):

- Your app expects `PaymentProcessor.pay(amount)`, but the Stripe SDK exposes `stripeCharge(cents, token)`.
- Your code expects `MediaPlayer.play(file)`, but the library offers `VlcLib.startStream(path)`.
- Your reporting expects JSON, but a vendor returns XML.

The naive fix is to litter your code with conversion calls everywhere you touch the foreign API — coupling your whole app to its shape:

```java
// SMELL: client is now coupled to the third-party API's vocabulary
StripeApi stripe = new StripeApi();
stripe.stripeCharge((int)(amount * 100), token);  // repeated everywhere, hard to swap later
```

Adapter isolates that translation in **one class**, so the rest of your code depends only on your own clean interface.

---

## 3. Structure

```
   Client ──depends on──▶ ┌──────────────────┐
                          │  <<interface>>   │
                          │  Target          │   (what YOUR code expects)
                          │  + request()     │
                          └────────┬─────────┘
                          implements│
                          ┌─────────┴─────────┐        wraps / delegates
                          │   Adapter         │──────────────────────────┐
                          │  + request() ─────┼─▶ adaptee.specificReq()   ▼
                          └───────────────────┘               ┌──────────────────┐
                                                              │   Adaptee        │
                                                              │ (3rd-party/legacy│
                                                              │  + specificReq() │
                                                              └──────────────────┘
```

| Role | What it is | Example |
|------|-----------|---------|
| **Target** | The interface the client wants | `PaymentProcessor` |
| **Adaptee** | The existing incompatible class | `StripeApi` |
| **Adapter** | Implements Target, delegates to Adaptee (translating) | `StripeAdapter` |
| **Client** | Uses Target, unaware of Adaptee | your app |

Two flavors: **object adapter** (holds the adaptee — composition, preferred) and **class adapter** (multiple inheritance — not possible in Java for classes; rare).

---

## 4. Canonical code example (Java)

```java
// 1. Target — the clean interface YOUR app talks to
interface PaymentProcessor {
    void pay(double amountInRupees);
}

// 2. Adaptee — third-party SDK you cannot modify (different vocabulary)
class StripeApi {
    void stripeCharge(int amountInPaise, String currency) {
        System.out.println("Stripe charged " + amountInPaise + " " + currency);
    }
}

// 3. Adapter — implements Target, translates to the Adaptee
class StripeAdapter implements PaymentProcessor {
    private final StripeApi stripe;             // composition (object adapter)
    StripeAdapter(StripeApi stripe) { this.stripe = stripe; }

    public void pay(double amountInRupees) {
        int paise = (int) Math.round(amountInRupees * 100);  // translate units
        stripe.stripeCharge(paise, "INR");                   // translate the call
    }
}

// 4. Client depends only on PaymentProcessor — never sees StripeApi
public class Main {
    public static void main(String[] args) {
        PaymentProcessor processor = new StripeAdapter(new StripeApi());
        processor.pay(499.0);   // Stripe charged 49900 INR
    }
}
```

Swapping Stripe for Razorpay later = write a `RazorpayAdapter`; the client doesn't change at all.

---

## 5. When to use it

- You must use an existing class whose **interface doesn't match** what your code needs.
- You **can't (or shouldn't) modify** the existing class (third-party, legacy, generated).
- You want to **isolate** your code from an external API so it's swappable.
- You're integrating **multiple providers** behind one common interface (each gets an adapter).

## When NOT to use it

- You **own** both sides and can just change one interface → do that instead.
- The mismatch is trivial (a rename) → maybe not worth a class.
- You actually need to **add behavior**, not translate → that's **Decorator**. Or **simplify a whole subsystem** → that's **Facade**.

---

## 6. Real systems that use Adapter (good to name-drop)

- **`java.util.Arrays.asList()`** — adapts an array to the `List` interface.
- **`InputStreamReader`** — adapts a byte `InputStream` to a char `Reader` (also cited for Decorator; here the point is *interface conversion*).
- **`Collections.list(Enumeration)`** — adapts legacy `Enumeration` to modern `List`.
- **SLF4J** — adapts to Log4j/Logback/JUL backends; **JDBC drivers** adapt DB-specific protocols to the JDBC interface.
- **Spring `HandlerAdapter`**, payment-gateway integrations, the universal "wrap the vendor SDK" pattern.

---

## 7. Adapter vs. lookalike patterns (interviewers probe this)

| Pattern | Core difference |
|---------|-----------------|
| **Decorator** | Decorator **keeps the same interface** and **adds behavior** (stackable). Adapter **changes the interface** to a different one and adds no behavior. |
| **Facade** | Facade gives a **new simplified** interface over a **whole subsystem** (convenience). Adapter makes **one existing** class conform to an **expected** interface (compatibility). Facade = simplify; Adapter = translate. |
| **Proxy** | Proxy keeps the **same interface** and controls access. Adapter **changes** the interface. |
| **Bridge** | Bridge is designed **up front** to separate abstraction from implementation. Adapter is applied **after the fact** to make incompatible things work together. |

The crisp distinction: **Adapter changes an interface; Decorator/Proxy keep it; Facade invents a simpler one.**

---

## 8. How it shows up in FAANG LLD rounds

- **Payment systems** → adapters for Stripe / Razorpay / PayPal behind one `PaymentProcessor`.
- **Notification systems** → adapters for Twilio (SMS), SendGrid (email), FCM (push).
- **Media player** → adapt different codec libraries to one `Player` interface.
- **Logging** → adapt multiple logging backends (the SLF4J story).
- **Third-party / legacy integration** → any "we must call this old/external API" sub-task.
- **Maps / geocoding / storage (S3 vs GCS)** → one interface, provider adapters.

**Where to introduce it:** when the design must call an external/legacy API with the "wrong" shape. Say: *"I'll define our own `PaymentProcessor` interface and write an adapter per provider, so the core stays decoupled and providers are swappable."*

---

## 9. How to talk about it (scoring phrases)

- "The client depends on **our** interface, not the vendor's — **Dependency Inversion**, and the vendor is swappable."
- "Each provider gets its own adapter — adding one doesn't touch the client — **Open/Closed**."
- "The translation lives in **one place** instead of leaking the vendor API across the codebase."
- "Adapter **changes** the interface; that's how it differs from Decorator and Facade."

---

## 10. Common mistakes (avoid these in the room)

1. **Confusing it with Facade** — Adapter conforms one class to an expected interface; Facade simplifies a whole subsystem.
2. **Confusing it with Decorator** — Adapter converts interface (no new behavior); Decorator adds behavior (same interface).
3. **Putting business logic in the adapter** — keep it to translation; logic belongs elsewhere.
4. **Leaking the adaptee** through the adapter (exposing its types) — defeats the isolation.
5. **One mega-adapter** for many unrelated classes — one adapter per adaptee/interface.

---

## 11. Worked problem — unifying multiple payment gateways

**Prompt:** *Your checkout must support several payment providers (Stripe, Razorpay), each with its own SDK and method signatures, and more will be added. The checkout code should not depend on any specific SDK.*

Adapter is the highlighted pattern. (A **Factory** can pick which adapter to build, and **Strategy** is conceptually nearby since each provider is an interchangeable processor — but the reason it *works* is the Adapter translating each SDK.)

### Step 1 — define the interface YOUR app owns
```java
interface PaymentProcessor {
    PaymentResult pay(double amountInRupees, String orderId);
}
record PaymentResult(boolean success, String reference) {}
```

### Step 2 — the incompatible third-party SDKs (cannot modify)
```java
class StripeSdk {
    String createCharge(long amountCents, String currency) { return "ch_123"; }
}
class RazorpaySdk {
    Map<String,Object> charge(int amountPaise, String receiptId) {
        return Map.of("status", "captured", "id", "pay_987");
    }
}
```

### Step 3 — one adapter per provider, each translating to PaymentProcessor
```java
class StripeAdapter implements PaymentProcessor {
    private final StripeSdk sdk;
    StripeAdapter(StripeSdk sdk) { this.sdk = sdk; }
    public PaymentResult pay(double rupees, String orderId) {
        long cents = Math.round(rupees * 100);          // unit translation
        String chargeId = sdk.createCharge(cents, "INR");
        return new PaymentResult(true, chargeId);        // shape translation
    }
}
class RazorpayAdapter implements PaymentProcessor {
    private final RazorpaySdk sdk;
    RazorpayAdapter(RazorpaySdk sdk) { this.sdk = sdk; }
    public PaymentResult pay(double rupees, String orderId) {
        int paise = (int) Math.round(rupees * 100);
        Map<String,Object> resp = sdk.charge(paise, orderId);
        boolean ok = "captured".equals(resp.get("status"));
        return new PaymentResult(ok, (String) resp.get("id"));
    }
}
```

### Step 4 — a Factory picks the adapter; the client stays clean
```java
enum Provider { STRIPE, RAZORPAY }
class PaymentFactory {
    PaymentProcessor create(Provider p) {
        return switch (p) {
            case STRIPE   -> new StripeAdapter(new StripeSdk());
            case RAZORPAY -> new RazorpayAdapter(new RazorpaySdk());
        };
    }
}

// Client — knows ONLY PaymentProcessor
PaymentProcessor processor = new PaymentFactory().create(Provider.RAZORPAY);
PaymentResult result = processor.pay(499.0, "order_42");
```

### 🔍 Where Adapter is doing the work (the highlight)
- Each SDK speaks a **different language** — `createCharge(cents, currency)` returning a `String` vs `charge(paise, receipt)` returning a `Map`. The adapters **translate both the call shape and the data** (rupees→cents/paise, vendor response→`PaymentResult`) into the one `PaymentProcessor` interface. That translation *is* the Adapter pattern.
- The checkout client depends only on `PaymentProcessor` — it never imports `StripeSdk` or `RazorpaySdk`. The vendor coupling is quarantined inside the adapters → **Dependency Inversion**, fully swappable providers.
- Adding PayPal = **one new adapter** + one factory line; the client and existing adapters are untouched → **Open/Closed**.
- Note the division of labour: the **Factory** only *chooses* which adapter; the **Adapter** does the actual interface *conversion*. If the SDKs already matched our interface, we'd need only the Factory — the Adapter exists precisely because they don't.

---

## 12. 60-second recall summary

- **Problem:** you must use a class whose interface doesn't match what your code expects, and you can't change it.
- **Fix:** an Adapter implements your Target interface and delegates to the incompatible Adaptee, translating calls/data.
- **Wins:** decoupling from external/legacy APIs, swappable providers, Open/Closed.
- **Distinguish:** Adapter *changes* the interface; Decorator *adds behavior* (same interface); Facade *simplifies a subsystem*.
- **Real example to cite:** `Arrays.asList()`, `InputStreamReader`, SLF4J, JDBC drivers.

---

### Suggested next patterns to learn (in order, for LLD interviews)
1. Strategy ✅ · 2. Factory ✅ · 3. Observer ✅ · 4. State ✅ · 5. Decorator ✅ · 6. Builder ✅ · 7. Singleton ✅
8. Adapter ✅ (this one) — then Command / Chain of Responsibility / Facade
