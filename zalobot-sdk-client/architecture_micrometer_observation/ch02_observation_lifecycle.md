# Chapter 2: The Observation Lifecycle — Start, Scope, Stop

## What You'll Learn
- The complete lifecycle of an Observation: create → start → scope → stop
- What `Context` is and why it's mutable
- How `Scope` puts observation data into ThreadLocal
- Why `error()` is separate from `stop()`
- The handler callback sequence

---

## 2.1 Starting from Scratch: A Minimal Observation

Let's build up understanding step by step. The simplest possible observation:

```java
ObservationRegistry registry = ObservationRegistry.create();
// (No handlers registered yet — this does nothing useful)

Observation observation = Observation.start("my.operation", registry);
try {
    doWork();
} finally {
    observation.stop();
}
```

This creates an observation, but with no handlers registered, nothing happens. The observation is effectively a no-op. **The power comes when handlers are registered.**

## 2.2 The Context: Mutable Data Bag

Every observation carries a `Context` — a mutable object that accumulates data throughout the observation's lifetime:

```java
// Context is like a Map with typed accessors
Observation.Context context = new Observation.Context();
context.setName("zalobot.client.request");
context.setContextualName("sendMessage");
context.setError(exception);
context.put(Timer.Sample.class, timerSample);  // Handler stores its data here
context.put(Span.class, span);                  // Another handler stores its data here
```

The key insight: **Context is shared between ALL handlers.** Each handler can read what others stored:

```
┌─────────────────────────────────────────────────────┐
│                 Observation.Context                  │
│  ─────────────────────────────────────────────────  │
│  name: "zalobot.client.request"                     │
│  contextualName: "sendMessage"                      │
│  error: null (or Throwable)                         │
│  lowCardinalityKeyValues: [method=sendMessage, ...]  │
│  highCardinalityKeyValues: [url=https://..., ...]   │
│  ─────────────────────────────────────────────────  │
│  Custom data (via put/get):                         │
│    Timer.Sample → (stored by metrics handler)       │
│    TracingContext → (stored by tracing handler)      │
│    Span → (stored by tracing handler)               │
└─────────────────────────────────────────────────────┘
```

> ★ **Insight** ─────────────────────────────────────
> - **Why Context is mutable, not immutable**: Handlers need to store intermediate state. The metrics handler stores a `Timer.Sample` on `onStart()` and retrieves it on `onStop()` to compute duration. If Context were immutable, each handler would need its own side-channel storage, defeating the purpose of a unified data carrier.
> - **Trade-off**: Mutability means handlers must be careful about thread safety. In practice, observations are typically used within a single thread (or with explicit `Scope` management), so this is acceptable.
> ─────────────────────────────────────────────────────

## 2.3 The Full Lifecycle Sequence

Here's what happens at each lifecycle stage, and which handler methods are called:

```java
// Step 1: CREATE (observation exists but hasn't started)
Observation observation = Observation.createNotStarted("my.op", registry);
// → No handler calls yet. Convention is resolved.

// Step 2: START (clock starts ticking)
observation.start();
// → handler.onStart(context)
//   Metrics handler: Timer.Sample sample = Timer.start(clock)
//   Tracing handler: Span span = tracer.nextSpan().start()

// Step 3: SCOPE (put observation into ThreadLocal)
try (Observation.Scope scope = observation.openScope()) {
    // → handler.onScopeOpened(context)
    //   Tracing handler: puts Span into CurrentTraceContext

    // Step 3a: ADD DATA (accumulate tags during the operation)
    observation.lowCardinalityKeyValue("method", "sendMessage");
    observation.highCardinalityKeyValue("chat.id", "12345");

    // Step 3b: ERROR (if something goes wrong)
    // observation.error(exception);
    // → handler.onError(context)
    //   Tracing handler: span.error(exception)

    doWork();
}
// → handler.onScopeClosed(context) (when scope.close() called by try-with-resources)
//   Tracing handler: closes CurrentTraceContext.Scope

// Step 4: STOP (finalize and record)
observation.stop();
// → ObservationFilter.map(context)  -- mutate context before handlers
// → Convention resolves final name + tags
// → handler.onStop(context)
//   Metrics handler: sample.stop(timer)  -- records duration
//   Tracing handler: span.tag(...).end() -- ends the span
```

## 2.4 Visualizing the Callback Sequence

```
Time ──────────────────────────────────────────────────►

observation     handler         handler
 .start() ─────► onStart() ─────► onStart()
                 (metrics)       (tracing)
    │
    ▼
 .openScope() ─► onScopeOpened() ► onScopeOpened()
    │
    │  [user code executes]
    │  [.error(ex) possible] ──► onError() ──► onError()
    │
    ▼
 scope.close() ► onScopeClosed() ► onScopeClosed()
    │
    ▼
 .stop() ──────► [filters run] ─► onStop() ──► onStop()
                                  (metrics)    (tracing)
```

## 2.5 Why error() and stop() Are Separate

You might wonder: why not just call `stop(exception)`? The separation is deliberate:

```java
try (Observation.Scope scope = observation.openScope()) {
    ZaloApiResponse<N> response = doExchange();
    // Error case 1: API returned error but HTTP succeeded
    if (!response.ok()) {
        observation.error(new ZaloBotApiException(...));
        // But we STILL call stop() — we want to record the duration even for errors
    }
    return response;
} catch (IOException e) {
    // Error case 2: HTTP request itself failed
    observation.error(e);
    throw e;
} finally {
    observation.stop();  // ALWAYS stop — records duration regardless of success/failure
}
```

The pattern: `error()` records WHAT went wrong; `stop()` records HOW LONG it took. Both are always called.

> ★ **Insight** ─────────────────────────────────────
> - **error() before stop() is a contract**: The metrics handler uses context.getError() in `onStop()` to add an "exception" tag to the timer. If you called `stop()` first, the error would be lost. This ordering is the same across Spring Framework, Spring Kafka, and all Micrometer integrations.
> - **When error() is called without stop()**: This is a bug. Observation will leak (timer never recorded, span never ended). Always ensure `stop()` is in a `finally` block.
> ─────────────────────────────────────────────────────

## 2.6 The NOOP Optimization

When `ObservationRegistry.NOOP` is used (or when an `ObservationPredicate` disables the observation), the entire lifecycle becomes zero-cost:

```java
// When registry is NOOP:
Observation observation = Observation.start("my.op", ObservationRegistry.NOOP);
// Returns Observation.NOOP — a singleton that does nothing

observation.start();        // no-op
observation.openScope();    // returns Scope.NOOP
observation.error(ex);      // no-op
observation.stop();         // no-op
```

This is critical for library design: **your SDK always instruments, but if the user doesn't configure an ObservationRegistry, there's zero overhead.**

```
┌────────────────────────────────────────────────────────────────┐
│ Library code (Zalobot SDK)              User's application     │
│                                                                │
│ observation = ZaloBotClientObservation   ┌─ With Spring Boot   │
│   .API_REQUEST                          │  Actuator: real      │
│   .observation(...)                     │  metrics + traces    │
│                                         │                      │
│ // Always the same code                 ├─ Without Actuator:   │
│ observation.start();                    │  NOOP, zero cost     │
│ try { ... } finally {                   │                      │
│   observation.stop();                   └─ Custom: user's own  │
│ }                                          handlers            │
└────────────────────────────────────────────────────────────────┘
```

## 2.7 Connection to Real Micrometer

| Concept | Real Source |
|---------|------------|
| Observation lifecycle | `Observation.java` — `start()`, `stop()`, `error()`, `openScope()` methods |
| SimpleObservation (default impl) | `micrometer/micrometer-observation/src/main/java/io/micrometer/observation/SimpleObservation.java` |
| Context put/get | `Observation.Context` inner class — `put(Object key, Object value)`, `get(Class<T> key)` |
| NOOP optimization | `Observation.NOOP` field, `ObservationRegistry.NOOP` field |
| Handler onStart/onStop | `ObservationHandler.java` — all default methods |
| Timer.Sample stored in context | `DefaultMeterObservationHandler.onStart()` → `context.put(Timer.Sample.class, sample)` |
| Span stored in context | `DefaultTracingObservationHandler.onStart()` → `context.put(TracingContext.class, tracingContext)` |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Context** | Mutable data bag shared across all handlers; stores name, tags, error, and handler-specific data |
| **start()** | Signals handlers to begin (metrics starts Timer.Sample, tracing starts Span) |
| **openScope()** | Puts observation into ThreadLocal (makes trace context visible to downstream code) |
| **error()** | Records what went wrong (exception); must be called BEFORE stop() |
| **stop()** | Finalizes observation (records duration, ends span); always call in finally block |
| **NOOP** | Zero-overhead singleton when observation is disabled; critical for library design |

**Next: Chapter 3** — The Convention pattern: how naming and tagging are separated from the observation itself.
