# Chapter 1: The Problem — Why You Need Observability in a Bot SDK

## What You'll Learn
- Why a bot SDK without observability is a "black box" in production
- What questions operators need answered about their bot's behavior
- Why traditional logging alone is insufficient
- The difference between metrics, traces, and observations

---

## 1.1 The Black Box Problem

Imagine you've deployed a Zalo bot in production. Users start complaining that the bot is "slow" or "not responding." You check the logs and see... nothing useful. The bot is running, the polling loop is active, but you have no idea:

- How long does each API call to Zalo take?
- How many messages per second is the bot processing?
- What percentage of API calls are failing?
- Is the bounded queue filling up?
- Are there bursts of timeouts?

Your `ZaloUpdateListenerContainer` is a sophisticated producer-consumer system, but right now it's a **black box**:

```
┌────────────────────────────────────────────────────────────┐
│                    Black Box                                │
│                                                            │
│   Polling Thread ──?──> Queue ──?──> Processing Threads    │
│        │                  │                │                │
│     How fast?        How full?        How slow?            │
│     Errors?           Depth?          Failures?            │
│                                                            │
│   HTTP Client ──?──> Zalo API                              │
│        │                                                   │
│     Latency? Status? Throughput?                           │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

## 1.2 Why Logging Isn't Enough

You might think: "I'll just add more log statements." Let's try:

```java
// In DefaultZaloBotClient.exchangeInternal()
logger.info("Calling {} at {}", methodPath, System.currentTimeMillis());
// ... execute HTTP request ...
logger.info("Got response from {} in {}ms, status={}", methodPath, duration, httpStatus);
```

Problems with this approach:

| Problem | Why It Matters |
|---------|---------------|
| **No aggregation** | You can't easily compute p99 latency from log lines |
| **No alerting** | Can't set "alert when error rate > 5%" from text logs |
| **No dashboards** | Grafana/Prometheus can't consume log lines as metrics |
| **No correlation** | If bot calls Zalo API which calls another service, how do you trace the full request? |
| **Performance cost** | String formatting for every request adds overhead |
| **Volume** | At 1000 msg/sec, log files explode |

## 1.3 What We Actually Need

We need **three complementary signals**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Observability Signals                         │
│                                                                 │
│  1. METRICS (aggregated counters/timers)                        │
│     "How many requests/sec? What's the p99 latency?"            │
│     → Prometheus, Grafana dashboards                            │
│                                                                 │
│  2. TRACES (per-request execution path)                         │
│     "This specific request took 2s — where was the time spent?" │
│     → Zipkin, Jaeger, Tempo                                     │
│                                                                 │
│  3. LOGS (contextual text)                                      │
│     "What happened during this trace? Error details?"           │
│     → Correlated with trace IDs for drill-down                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 1.4 The Naive Approach: Instrument Twice

Without Micrometer's Observation API, you'd instrument separately for metrics and traces:

```java
// ❌ Naive approach: separate instrumentation for metrics and tracing
private <N> ZaloApiResponse<N> exchangeInternal(...) {
    // Metrics instrumentation
    Timer.Sample sample = Timer.start(meterRegistry);

    // Tracing instrumentation
    Span span = tracer.nextSpan().name("zalobot " + methodPath).start();

    try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
        ZaloApiResponse<N> response = doExchange(...);
        span.tag("status", "ok");
        return response;
    } catch (Exception e) {
        span.error(e);
        throw e;
    } finally {
        sample.stop(Timer.builder("zalobot.client.requests")
            .tag("method", methodPath)
            .register(meterRegistry));
        span.end();
    }
}
```

Problems:
1. **Two code paths** — must keep metrics and tracing in sync
2. **Coupling** — code now depends on both Micrometer metrics AND a tracing library
3. **No correlation** — metrics and traces aren't automatically linked
4. **Hard to extend** — adding logging correlation requires yet another integration

## 1.5 The Solution Preview: One Observation, Multiple Outputs

Micrometer's Observation API solves this with a single instrumentation point:

```java
// ✅ One observation → metrics + traces + logs automatically
private <N> ZaloApiResponse<N> exchangeInternal(...) {
    Observation observation = ZaloBotClientObservation.API_REQUEST
        .observation(convention, DEFAULT_CONVENTION,
            () -> new ZaloBotClientContext(methodPath, method),
            observationRegistry);

    observation.start();
    try (Observation.Scope scope = observation.openScope()) {
        ZaloApiResponse<N> response = doExchange(...);
        return response;
    } catch (Exception e) {
        observation.error(e);
        throw e;
    } finally {
        observation.stop();
    }
}
```

One instrumentation point, but **handlers** independently produce:
- **Metrics handler**: Records a Timer with tags (method, status, error)
- **Tracing handler**: Creates a Span with the same tags
- **Logging handler**: Adds trace_id/span_id to MDC

> ★ **Insight** ─────────────────────────────────────
> - **Why unified observation beats separate instrumentation**: The #1 production debugging workflow is "alert on metric anomaly → drill into trace → read correlated logs." If metrics and traces are instrumented separately, they can drift apart (different tag names, different error conditions), breaking this workflow. A single observation guarantees they're always in sync.
> - **When NOT to use Observation**: For very fine-grained, hot-path metrics (e.g., counting cache hits in a tight loop), raw `Counter.increment()` may be more appropriate. Observation adds ~100ns overhead per invocation — negligible for HTTP calls, but noticeable at millions of ops/sec.
> ─────────────────────────────────────────────────────

## 1.6 Connection to Real Micrometer

| Concept | Real Source File |
|---------|-----------------|
| Observation API | `micrometer/micrometer-observation/src/main/java/io/micrometer/observation/Observation.java` |
| NOOP Observation | `Observation.NOOP` field in same file |
| ObservationRegistry | `micrometer/micrometer-observation/src/main/java/io/micrometer/observation/ObservationRegistry.java` |
| DefaultMeterObservationHandler | `micrometer/micrometer-core/src/main/java/io/micrometer/core/instrument/observation/DefaultMeterObservationHandler.java` |
| DefaultTracingObservationHandler | `tracing/micrometer-tracing/src/main/java/io/micrometer/tracing/handler/DefaultTracingObservationHandler.java` |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Black Box Problem** | Without observability, production bot behavior is opaque |
| **Three Signals** | Metrics (aggregated), Traces (per-request), Logs (contextual) |
| **Dual Instrumentation** | Instrumenting metrics and tracing separately leads to drift and coupling |
| **Observation API** | Single instrumentation point that feeds multiple handlers |

**Next: Chapter 2** — How the Observation lifecycle works: start, scope, stop, and how handlers react.
