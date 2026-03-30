# Simple Micrometer — API-First Learning Outline

> Learn Micrometer by building a simplified version from the **outside in** — start with
> the APIs that clients actually call, then implement the internal machinery behind each one.

## Quick Start

```bash
# After implementing Feature 1, a client can already do:
```
```java
MeterRegistry registry = new SimpleMeterRegistry();
Counter counter = Counter.builder("http.requests")
    .tag("method", "GET")
    .tag("status", "200")
    .register(registry);
counter.increment();
counter.increment(5);
System.out.println(counter.count()); // 6.0
```

## Architecture Overview

Micrometer lets application developers **record measurements** (counts, timings, current values)
using a vendor-neutral API, then **export those measurements** to any monitoring backend
(Prometheus, Datadog, Graphite, etc.) without changing application code. Think "SLF4J for
metrics" — you code against a facade, and the backend binding is a deployment decision.

**Real-world analogy**: Using Micrometer is like having a universal dashboard instrument panel
in your car. You glance at the speedometer (Gauge), the odometer (Counter), and the trip timer
(Timer) — all standardized interfaces. Under the hood, these instruments can feed their data
to your phone app, the dealership's diagnostic system, or a fleet management platform, all
simultaneously, without rewiring the dashboard.

## API Surface Map

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           CLIENT CODE                                     │
│  records: Counter, Timer, Gauge, DistributionSummary, LongTaskTimer       │
│  configures: MeterFilter, NamingConvention, MeterBinder                   │
│  queries: Search, Metrics (global facade)                                 │
└────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────────┘
     │          │          │          │          │          │
┌────▼───┐ ┌───▼───┐ ┌───▼───┐ ┌───▼───┐ ┌───▼───┐ ┌───▼──────────┐
│  F1    │ │  F2   │ │  F3   │ │  F4   │ │  F5   │ │  F6–F15      │
│Counter │ │Gauge  │ │Timer  │ │Dist.  │ │Filter │ │Naming, Comp- │
│Builder │ │Builder│ │Builder│ │Summary│ │Chain  │ │osite, Search │
│register│ │regist.│ │record │ │record │ │map/   │ │Binder, LTT,  │
│incr()  │ │value()│ │time() │ │       │ │accept │ │Push, Observe │
└────┬───┘ └───┬───┘ └───┬───┘ └───┬───┘ └───┬───┘ └───┬──────────┘
     │         │         │         │         │         │
┌────▼─────────▼─────────▼─────────▼─────────▼─────────▼──────────────┐
│                        SHARED INTERNALS                               │
│  Meter.Id, Tag/Tags, Clock, MeterRegistry (abstract),                 │
│  SimpleMeterRegistry, Measurement, Statistic,                         │
│  internal meter implementations (DefaultGauge, CumulativeCounter...)   │
└──────────────────────────────────────────────────────────────────────┘
```

## Call Chain Diagrams

### Counter Creation Call Chain
```
Client calls: Counter.builder("http.requests").tag("method","GET").register(registry)
  │
  ├─► [API Layer] Counter.Builder
  │     Stores name, tags, description, baseUnit. Terminal op: register(registry)
  │
  ├─► [API Layer] MeterRegistry.counter(Meter.Id)
  │     Calls registerMeterIfNecessary() with the counter's ID
  │
  ├─► [Registration Layer] MeterRegistry.getOrCreateMeter()
  │     1. Check pre-filter cache for existing meter → fast return if found
  │     2. Run MeterFilter.map() chain to transform the ID
  │     3. Run MeterFilter.accept() chain → return NoopCounter if denied
  │     4. Call newCounter(mappedId) on concrete registry subclass
  │     5. Store in meterMap + preFilterIdToMeterMap
  │
  └─► [Implementation Layer] SimpleMeterRegistry.newCounter(id)
        Creates a CumulativeCounter backed by a DoubleAdder
```

### Counter Increment Call Chain
```
Client calls: counter.increment(5.0)
  │
  └─► [Implementation Layer] CumulativeCounter.increment(amount)
        Directly adds to internal DoubleAdder — no dispatch, no overhead
```

### Timer Recording Call Chain
```
Client calls: timer.record(() -> doWork())
  │
  ├─► [API Layer] Timer.record(Runnable)
  │     Captures clock.monotonicTime() before and after execution
  │     Calls record(duration, TimeUnit.NANOSECONDS)
  │
  └─► [Implementation Layer] CumulativeTimer.record(amount, unit)
        Updates count (LongAdder), totalTime (DoubleAdder), max (TimeWindowMax)
```

### Gauge Observation Call Chain
```
Client calls: gauge.value()
  │
  ├─► [Implementation Layer] DefaultGauge.value()
  │     Retrieves weak reference to state object
  │     Applies ToDoubleFunction to get current value
  │     Returns NaN if object has been garbage collected
  │
  └─► (No storage — gauges are "pull-based", evaluated on read)
```

### Meter Filter Chain Call Chain
```
Client calls: registry.config().meterFilter(MeterFilter.deny(id -> id.getName().startsWith("jvm")))
  │
  ├─► [Config Layer] MeterRegistry.Config.meterFilter(filter)
  │     Appends filter to ordered list of MeterFilters
  │
  └─► [Applied during registration] MeterRegistry.getOrCreateMeter()
        For each filter in order:
          1. map(id) — transform name/tags (e.g., add common tags, rename)
          2. accept(id) — ACCEPT / DENY / NEUTRAL
          3. configure(id, config) — modify distribution statistics
        First DENY wins → returns noop meter
```

## Features

### Feature 1: Counter & MeterRegistry — Counting Things ❲Tier 1❳
✅ Complete

**API Contract** (what clients call):
```java
// Create a registry
MeterRegistry registry = new SimpleMeterRegistry();

// Create and use a counter
Counter counter = Counter.builder("http.requests.total")
    .tag("method", "GET")
    .tag("status", "200")
    .description("Total HTTP requests")
    .baseUnit("requests")
    .register(registry);

counter.increment();
counter.increment(5.0);
double count = counter.count(); // 6.0

// Shorthand
Counter shorthand = registry.counter("cache.misses", "cache", "users");
```

**What it does**: Lets clients create named, dimensionally-tagged counters to track monotonically increasing values (request counts, error counts, bytes sent).

**Depth layers** (what gets built to support this API):

| Layer | Simplified Class            | Real Framework Class                                         | Purpose                                   |
|-------|-----------------------------|--------------------------------------------------------------|-------------------------------------------|
| API   | `Counter`                   | `io.micrometer.core.instrument.Counter`                      | Counter interface + Builder               |
| API   | `Meter`                     | `io.micrometer.core.instrument.Meter`                        | Base meter interface, Id, Type            |
| API   | `Tag` / `Tags`              | `io.micrometer.core.instrument.Tag` / `Tags`                 | Dimensional metadata                      |
| API   | `MeterRegistry`             | `io.micrometer.core.instrument.MeterRegistry`                | Abstract registry with registration logic |
| API   | `Clock`                     | `io.micrometer.core.instrument.Clock`                        | Time source abstraction                   |
| API   | `Measurement` / `Statistic` | `io.micrometer.core.instrument.Measurement` / `Statistic`    | Meter readout types                       |
| Impl  | `SimpleMeterRegistry`       | `io.micrometer.core.instrument.simple.SimpleMeterRegistry`   | In-memory registry for testing            |
| Impl  | `CumulativeCounter`         | `io.micrometer.core.instrument.cumulative.CumulativeCounter` | Monotonically increasing counter          |
| Impl  | `NoopCounter`               | `io.micrometer.core.instrument.noop.NoopCounter`             | Null-object for denied counters           |

**Real source files**: `Counter.java`, `Meter.java`, `Tag.java`, `Tags.java`, `MeterRegistry.java`, `Clock.java`, `Measurement.java`, `Statistic.java`, `SimpleMeterRegistry.java`, `CumulativeCounter.java`, `NoopCounter.java`

**Depends on**: None (foundation feature)
**Complexity**: High · 11 new classes (but this is the foundation everything else builds on)
**Java version**: 17

**Concrete deliverables**:
- [x] `api/Meter.java` — base interface with Id, Type, measure()
- [x] `api/Tag.java` — key-value pair interface
- [x] `api/Tags.java` — immutable sorted tag collection
- [x] `api/Counter.java` — counter interface + fluent Builder
- [x] `api/MeterRegistry.java` — abstract registry with meter lifecycle
- [x] `api/Clock.java` — time source interface
- [x] `api/Measurement.java` — statistic + value pair
- [x] `api/Statistic.java` — enum of statistic types
- [x] `internal/CumulativeCounter.java` — counter backed by DoubleAdder
- [x] `internal/NoopCounter.java` — null-object counter
- [x] `internal/SimpleMeterRegistry.java` — in-memory test registry
- [x] `api/CounterTest.java` — client-perspective test
- [x] `internal/SimpleMeterRegistryTest.java` — registry unit test
- [x] `docs-v2/ch01_counter_and_registry.md` — tutorial chapter

---

### Feature 2: Gauge — Observing Current Values ❲Tier 1❳
✅ Complete

**API Contract** (what clients call):
```java
// Track a field via function reference
List<String> queue = new ArrayList<>();
Gauge gauge = Gauge.builder("queue.size", queue, List::size)
    .description("Current queue depth")
    .register(registry);

queue.add("item1");
queue.add("item2");
double size = gauge.value(); // 2.0

// Convenience: wrap a Number supplier
AtomicInteger connections = new AtomicInteger(0);
Gauge.builder("active.connections", () -> connections)
    .register(registry);

// Shorthand on registry
registry.gauge("cache.size", myCache, Cache::size);
```

**What it does**: Lets clients observe the current value of something — queue depth, active connections, memory usage — by sampling a function or number on demand.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class                                  | Purpose                   |
|-------|------------------|-------------------------------------------------------|---------------------------|
| API   | `Gauge`          | `io.micrometer.core.instrument.Gauge`                 | Gauge interface + Builder |
| Impl  | `DefaultGauge`   | `io.micrometer.core.instrument.internal.DefaultGauge` | WeakReference-based gauge |
| Impl  | `NoopGauge`      | `io.micrometer.core.instrument.noop.NoopGauge`        | Null-object gauge         |

**Real source files**: `Gauge.java`, `DefaultGauge.java`, `NoopGauge.java`

**Depends on**: Feature 1 (reuses MeterRegistry, Meter.Id, Tag/Tags, SimpleMeterRegistry)
**Complexity**: Medium · 3 new classes, 2 modified (MeterRegistry, SimpleMeterRegistry)
**Java version**: 17

**Concrete deliverables**:
- [x] `api/Gauge.java` — gauge interface + fluent Builder with weak/strong reference option
- [x] `internal/DefaultGauge.java` — WeakReference-backed gauge implementation
- [x] `internal/NoopGauge.java` — null-object gauge
- [x] `api/GaugeTest.java` — client-perspective test (including GC behavior)
- [x] `docs-v2/ch02_gauge.md` — tutorial chapter

---

### Feature 3: Timer — Measuring Durations ❲Tier 1❳
✅ Complete

**API Contract** (what clients call):
```java
Timer timer = Timer.builder("http.request.duration")
    .tag("method", "GET")
    .description("HTTP request latency")
    .register(registry);

// Record explicit duration
timer.record(Duration.ofMillis(250));

// Record a block of code
timer.record(() -> handleRequest());

// Record with return value
String result = timer.record(() -> computeResult());

// Stopwatch pattern
Timer.Sample sample = Timer.start(registry);
// ... do work ...
sample.stop(timer);

// Read statistics
long count = timer.count();
double total = timer.totalTime(TimeUnit.MILLISECONDS);
double max = timer.max(TimeUnit.MILLISECONDS);
```

**What it does**: Lets clients measure how long operations take — request latency, query duration, processing time — with automatic time unit conversion and statistical aggregation.

**Depth layers**:

| Layer | Simplified Class  | Real Framework Class                                       | Purpose                            |
|-------|-------------------|------------------------------------------------------------|------------------------------------|
| API   | `Timer`           | `io.micrometer.core.instrument.Timer`                      | Timer interface + Builder + Sample |
| Impl  | `CumulativeTimer` | `io.micrometer.core.instrument.cumulative.CumulativeTimer` | Count/total/max tracking           |
| Impl  | `NoopTimer`       | `io.micrometer.core.instrument.noop.NoopTimer`             | Null-object timer                  |
| Impl  | `TimeWindowMax`   | `io.micrometer.core.instrument.distribution.TimeWindowMax` | Decaying max tracker               |

**Real source files**: `Timer.java`, `CumulativeTimer.java`, `NoopTimer.java`, `TimeWindowMax.java`, `AbstractTimer.java`, `TimeUtils.java`

**Depends on**: Feature 1 (reuses MeterRegistry, Meter.Id, Clock)
**Complexity**: Medium · 4 new classes, 2 modified (MeterRegistry, SimpleMeterRegistry)
**Java version**: 17

**Concrete deliverables**:
- [x] `api/Timer.java` — timer interface + Builder + Sample inner class
- [x] `internal/CumulativeTimer.java` — count/totalTime/max tracking
- [x] `internal/NoopTimer.java` — null-object timer
- [x] `internal/TimeWindowMax.java` — rotating-window maximum tracker
- [x] `api/TimerTest.java` — client-perspective test
- [x] `docs-v2/ch03_timer.md` — tutorial chapter

---

### Feature 4: Distribution Summary — Value Distributions ❲Tier 2❳
✅ Complete

**API Contract** (what clients call):
```java
DistributionSummary summary = DistributionSummary.builder("http.response.size")
    .tag("uri", "/api/users")
    .description("Response payload sizes")
    .baseUnit("bytes")
    .register(registry);

summary.record(1024);
summary.record(2048);

long count = summary.count();       // 2
double total = summary.totalAmount(); // 3072.0
double max = summary.max();          // 2048.0
double mean = summary.mean();        // 1536.0
```

**What it does**: Lets clients record the distribution of values (response sizes, batch sizes, queue depths) — like a Timer but for non-time quantities. Tracks count, total, max, and mean.

**Depth layers**:

| Layer | Simplified Class                | Real Framework Class                                                     | Purpose             |
|-------|---------------------------------|--------------------------------------------------------------------------|---------------------|
| API   | `DistributionSummary`           | `io.micrometer.core.instrument.DistributionSummary`                      | Interface + Builder |
| Impl  | `CumulativeDistributionSummary` | `io.micrometer.core.instrument.cumulative.CumulativeDistributionSummary` | Count/total/max     |
| Impl  | `NoopDistributionSummary`       | `io.micrometer.core.instrument.noop.NoopDistributionSummary`             | Null-object         |

**Real source files**: `DistributionSummary.java`, `CumulativeDistributionSummary.java`, `NoopDistributionSummary.java`, `AbstractDistributionSummary.java`

**Depends on**: Feature 1 (MeterRegistry, Meter.Id), Feature 3 (reuses TimeWindowMax for max tracking)
**Complexity**: Low · 3 new classes, 2 modified
**Java version**: 17

**Concrete deliverables**:
- [x] `api/DistributionSummary.java` — interface + Builder
- [x] `internal/CumulativeDistributionSummary.java` — count/total/max implementation
- [x] `internal/NoopDistributionSummary.java` — null-object
- [x] `api/DistributionSummaryTest.java` — client-perspective test
- [x] `docs-v2/ch04_distribution_summary.md` — tutorial chapter

---

### Feature 5: Meter Filtering & Transformation ❲Tier 2❳
✅ Complete

**API Contract** (what clients call):
```java
MeterRegistry registry = new SimpleMeterRegistry();

// Add common tags to every meter
registry.config().commonTags("app", "order-service", "env", "prod");

// Deny metrics by name prefix
registry.config().meterFilter(MeterFilter.denyNameStartsWith("jvm.gc"));

// Rename a tag across all meters
registry.config().meterFilter(MeterFilter.renameTag("http", "status", "http.status"));

// Custom accept/deny logic
registry.config().meterFilter(new MeterFilter() {
    @Override
    public MeterFilterReply accept(Meter.Id id) {
        if (id.getName().startsWith("internal.")) return MeterFilterReply.DENY;
        return MeterFilterReply.NEUTRAL;
    }
});

// Counter registered after filters are applied
Counter counter = registry.counter("http.requests", "method", "GET");
// counter.getId().getTags() includes "app"="order-service", "env"="prod"
```

**What it does**: Lets clients globally transform meter names/tags, accept or deny meter registration, and add common tags — all configured centrally on the registry, applied automatically during registration.

**Depth layers**:

| Layer        | Simplified Class           | Real Framework Class                                    | Purpose                                |
|--------------|----------------------------|---------------------------------------------------------|----------------------------------------|
| API          | `MeterFilter`              | `io.micrometer.core.instrument.config.MeterFilter`      | Filter interface + factory methods     |
| API          | `MeterFilterReply`         | `io.micrometer.core.instrument.config.MeterFilterReply` | ACCEPT/DENY/NEUTRAL enum               |
| Registration | `MeterRegistry` (modified) | `MeterRegistry.getOrCreateMeter()`                      | Apply filter chain during registration |

**Real source files**: `MeterFilter.java`, `MeterFilterReply.java`, `MeterRegistry.java` (filter application in `getOrCreateMeter`)

**Depends on**: Features 1–4 (filters apply to all meter types during registration)
**Complexity**: Medium · 2 new classes, 1 modified (MeterRegistry to add filter chain)
**Java version**: 17

**Concrete deliverables**:
- [x] `api/MeterFilter.java` — filter interface + static factory methods
- [x] `api/MeterFilterReply.java` — ACCEPT/DENY/NEUTRAL enum
- [x] `api/MeterFilterTest.java` — tests for filter chain behavior
- [x] `docs-v2/ch05_meter_filter.md` — tutorial chapter

---

### Feature 6: Naming Conventions ❲Tier 2❳
✅ Complete

**API Contract** (what clients call):
```java
MeterRegistry registry = new SimpleMeterRegistry();

// Use snake_case naming (e.g., for Prometheus)
registry.config().namingConvention(NamingConvention.snakeCase);

Counter counter = registry.counter("httpServerRequests", "statusCode", "200");
// Convention-formatted: http_server_requests with tag status_code=200
String name = counter.getId().getConventionName(NamingConvention.snakeCase);
// → "http_server_requests"
```

**What it does**: Lets registry implementations or users configure how meter names and tag keys are formatted for a specific backend — camelCase for some, snake_case for Prometheus, dot.case for Graphite, etc.

**Depth layers**:

| Layer | Simplified Class      | Real Framework Class                                    | Purpose                               |
|-------|-----------------------|---------------------------------------------------------|---------------------------------------|
| API   | `NamingConvention`    | `io.micrometer.core.instrument.config.NamingConvention` | Convention interface + built-in impls |
| API   | `Meter.Id` (modified) | `Meter.Id.getConventionName()`                          | Convention-aware name formatting      |

**Real source files**: `NamingConvention.java`, `Meter.java` (Id inner class)

**Depends on**: Feature 1 (Meter.Id, MeterRegistry.Config)
**Complexity**: Low · 1 new class, 1 modified (Meter.Id to add convention methods)
**Java version**: 17

**Concrete deliverables**:
- [x] `api/NamingConvention.java` — interface + dot/snakeCase/camelCase implementations
- [x] `api/NamingConventionTest.java` — test convention formatting
- [x] `docs-v2/ch06_naming_convention.md` — tutorial chapter

---

### Feature 7: Composite Registry — Multi-Backend Publishing ❲Tier 2❳
✅ Complete

**API Contract** (what clients call):
```java
SimpleMeterRegistry simpleRegistry = new SimpleMeterRegistry();
// Imagine: PrometheusRegistry prometheusRegistry = new PrometheusRegistry();

CompositeMeterRegistry composite = new CompositeMeterRegistry();
composite.add(simpleRegistry);
// composite.add(prometheusRegistry);

// Meters registered on composite are mirrored to all child registries
Counter counter = composite.counter("http.requests", "method", "GET");
counter.increment();

// Both child registries see the counter
double simpleCount = simpleRegistry.counter("http.requests", "method", "GET").count(); // 1.0
```

**What it does**: Lets clients publish metrics to multiple monitoring backends simultaneously by registering meters on a composite registry that delegates to all child registries.

**Depth layers**:

| Layer | Simplified Class               | Real Framework Class                                                   | Purpose                      |
|-------|--------------------------------|------------------------------------------------------------------------|------------------------------|
| API   | `CompositeMeterRegistry`       | `io.micrometer.core.instrument.composite.CompositeMeterRegistry`       | Multi-registry coordinator   |
| Impl  | `CompositeCounter`             | `io.micrometer.core.instrument.composite.CompositeCounter`             | Delegates to child counters  |
| Impl  | `CompositeGauge`               | `io.micrometer.core.instrument.composite.CompositeGauge`               | Delegates to child gauges    |
| Impl  | `CompositeTimer`               | `io.micrometer.core.instrument.composite.CompositeTimer`               | Delegates to child timers    |
| Impl  | `CompositeDistributionSummary` | `io.micrometer.core.instrument.composite.CompositeDistributionSummary` | Delegates to child summaries |

**Real source files**: `CompositeMeterRegistry.java`, `CompositeCounter.java`, `CompositeGauge.java`, `CompositeTimer.java`, `CompositeDistributionSummary.java`

**Depends on**: Features 1–4 (needs all meter types to wrap)
**Complexity**: High · 5 new classes
**Java version**: 17

**Concrete deliverables**:
- [x] `api/CompositeMeterRegistry.java` — composite registry
- [x] `internal/CompositeCounter.java` — delegating counter
- [x] `internal/CompositeGauge.java` — delegating gauge
- [x] `internal/CompositeTimer.java` — delegating timer
- [x] `internal/CompositeDistributionSummary.java` — delegating summary
- [x] `api/CompositeMeterRegistryTest.java` — multi-registry test
- [x] `docs-v2/ch07_composite_registry.md` — tutorial chapter

---

### Feature 8: Meter Search & Discovery ❲Tier 2❳
✅ Complete

**API Contract** (what clients call):
```java
MeterRegistry registry = new SimpleMeterRegistry();
registry.counter("http.requests", "method", "GET").increment(10);
registry.counter("http.requests", "method", "POST").increment(5);
registry.timer("http.duration", "method", "GET");

// Find meters by name and tags
Counter getCounter = registry.find("http.requests")
    .tag("method", "GET")
    .counter();
// getCounter.count() == 10.0

// Get all meters matching a name
Collection<Meter> meters = registry.find("http.requests").meters();

// Required search — throws if not found
Counter required = registry.get("http.requests")
    .tag("method", "GET")
    .counter();
```

**What it does**: Lets clients query registered meters by name and tags — useful for testing assertions, building dashboards, or introspecting what's been registered.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class                                  | Purpose                    |
|-------|------------------|-------------------------------------------------------|----------------------------|
| API   | `Search`         | `io.micrometer.core.instrument.search.Search`         | Fluent meter query builder |
| API   | `RequiredSearch` | `io.micrometer.core.instrument.search.RequiredSearch` | Throws-on-missing variant  |

**Real source files**: `Search.java`, `RequiredSearch.java`

**Depends on**: Feature 1 (MeterRegistry.find/get, meter storage)
**Complexity**: Low · 2 new classes, 1 modified (MeterRegistry to add find/get methods)
**Java version**: 17

**Concrete deliverables**:
- [x] `api/Search.java` — fluent meter query
- [x] `api/RequiredSearch.java` — throws-on-missing variant
- [x] `api/SearchTest.java` — search & discovery tests
- [x] `docs-v2/ch08_meter_search.md` — tutorial chapter

---

### Feature 9: MeterBinder — Reusable Instrumentation Modules ❲Tier 2❳
✅ Complete

**API Contract** (what clients call):
```java
MeterRegistry registry = new SimpleMeterRegistry();

// Bind a reusable instrumentation module
new JvmMemoryMetrics().bindTo(registry);

// Now registry has gauges for JVM memory metrics
Gauge heapUsed = registry.find("jvm.memory.used")
    .tag("area", "heap")
    .gauge();

// Custom binder
MeterBinder myBinder = reg -> {
    Gauge.builder("app.queue.size", queue, Queue::size)
        .register(reg);
};
myBinder.bindTo(registry);
```

**What it does**: Lets library authors package reusable instrumentation as composable modules that register their own meters when bound to a registry — the standard pattern for JVM metrics, HTTP client metrics, cache metrics, etc.

**Depth layers**:

| Layer   | Simplified Class   | Real Framework Class                                        | Purpose              |
|---------|--------------------|-------------------------------------------------------------|----------------------|
| API     | `MeterBinder`      | `io.micrometer.core.instrument.binder.MeterBinder`          | Functional interface |
| Example | `JvmMemoryMetrics` | `io.micrometer.core.instrument.binder.jvm.JvmMemoryMetrics` | Sample binder        |

**Real source files**: `MeterBinder.java`, `JvmMemoryMetrics.java`, `JvmGcMetrics.java`

**Depends on**: Features 1–2 (uses Counter, Gauge, MeterRegistry)
**Complexity**: Low · 2 new classes
**Java version**: 17

**Concrete deliverables**:
- [x] `api/MeterBinder.java` — functional interface
- [x] `api/binder/JvmMemoryMetrics.java` — sample binder implementation
- [x] `api/MeterBinderTest.java` — binder integration test
- [x] `docs-v2/ch09_meter_binder.md` — tutorial chapter

---

### Feature 10: Distribution Statistics & Histograms ❲Tier 3❳
✅ Complete

**API Contract** (what clients call):
```java
// Timer with percentile publishing
Timer timer = Timer.builder("http.request.duration")
    .publishPercentiles(0.5, 0.95, 0.99)
    .register(registry);

timer.record(Duration.ofMillis(100));
timer.record(Duration.ofMillis(200));
timer.record(Duration.ofMillis(300));

// Read histogram snapshot
HistogramSnapshot snapshot = timer.takeSnapshot();
ValueAtPercentile[] percentiles = snapshot.percentileValues();
// percentiles[0] ≈ {percentile=0.5, value≈200ms}

// Distribution summary with SLO boundaries
DistributionSummary summary = DistributionSummary.builder("http.response.size")
    .serviceLevelObjectives(1024, 4096, 16384)
    .publishPercentileHistogram()
    .register(registry);
```

**What it does**: Lets clients configure rich distribution statistics — client-side percentile computation, SLO boundary histograms, and min/max expected value clamping — for Timers and DistributionSummaries.

**Depth layers**:

| Layer | Simplified Class              | Real Framework Class                                                     | Purpose                     |
|-------|-------------------------------|--------------------------------------------------------------------------|-----------------------------|
| API   | `DistributionStatisticConfig` | `io.micrometer.core.instrument.distribution.DistributionStatisticConfig` | Histogram/percentile config |
| API   | `HistogramSupport`            | `io.micrometer.core.instrument.distribution.HistogramSupport`            | takeSnapshot() interface    |
| API   | `HistogramSnapshot`           | `io.micrometer.core.instrument.distribution.HistogramSnapshot`           | Snapshot readout            |
| API   | `ValueAtPercentile`           | `io.micrometer.core.instrument.distribution.ValueAtPercentile`           | Percentile data point       |
| API   | `CountAtBucket`               | `io.micrometer.core.instrument.distribution.CountAtBucket`               | Histogram bucket count      |
| Impl  | `PercentileHistogram`         | `io.micrometer.core.instrument.distribution.FixedBoundaryHistogram`      | Bucket-based histogram      |

**Real source files**: `DistributionStatisticConfig.java`, `HistogramSupport.java`, `HistogramSnapshot.java`, `ValueAtPercentile.java`, `CountAtBucket.java`, `FixedBoundaryHistogram.java`, `TimeWindowPercentileHistogram.java`

**Depends on**: Features 3–4 (enriches Timer and DistributionSummary)
**Complexity**: High · 6 new classes, 2 modified (Timer, DistributionSummary builders)
**Java version**: 17

**Concrete deliverables**:
- [x] `api/DistributionStatisticConfig.java` — histogram/percentile configuration
- [x] `api/HistogramSupport.java` — takeSnapshot() interface
- [x] `api/HistogramSnapshot.java` — snapshot value object
- [x] `api/ValueAtPercentile.java` — percentile data point
- [x] `api/CountAtBucket.java` — histogram bucket count
- [x] `internal/FixedBoundaryHistogram.java` — simple histogram impl
- [x] `api/DistributionStatisticTest.java` — percentile/histogram tests
- [x] `docs-v2/ch10_distribution_statistics.md` — tutorial chapter

---

### Feature 11: Long Task Timer — Tracking In-Flight Work ❲Tier 2❳
✅ Complete

**API Contract** (what clients call):
```java
LongTaskTimer longTimer = LongTaskTimer.builder("data.migration")
    .tag("table", "users")
    .register(registry);

// Start tracking a long-running task
LongTaskTimer.Sample task = longTimer.start();

// While task is running, observe:
int active = longTimer.activeTasks();    // 1
double duration = longTimer.duration(TimeUnit.SECONDS);  // elapsed time
double max = longTimer.max(TimeUnit.SECONDS);

// When done:
task.stop();  // active tasks → 0
```

**What it does**: Lets clients track currently-in-flight long-running operations (migrations, batch jobs, large file transfers) — unlike Timer which records completed durations, LongTaskTimer reports on tasks that haven't finished yet.

**Depth layers**:

| Layer | Simplified Class       | Real Framework Class                                          | Purpose                      |
|-------|------------------------|---------------------------------------------------------------|------------------------------|
| API   | `LongTaskTimer`        | `io.micrometer.core.instrument.LongTaskTimer`                 | Interface + Builder + Sample |
| Impl  | `DefaultLongTaskTimer` | `io.micrometer.core.instrument.internal.DefaultLongTaskTimer` | Active task set tracking     |
| Impl  | `NoopLongTaskTimer`    | `io.micrometer.core.instrument.noop.NoopLongTaskTimer`        | Null-object                  |

**Real source files**: `LongTaskTimer.java`, `DefaultLongTaskTimer.java`, `NoopLongTaskTimer.java`

**Depends on**: Feature 1 (MeterRegistry, Clock)
**Complexity**: Medium · 3 new classes, 2 modified (MeterRegistry.More, SimpleMeterRegistry)
**Java version**: 17

**Concrete deliverables**:
- [x] `api/LongTaskTimer.java` — interface + Builder + Sample
- [x] `internal/DefaultLongTaskTimer.java` — active task tracking implementation
- [x] `internal/NoopLongTaskTimer.java` — null-object
- [x] `api/LongTaskTimerTest.java` — client-perspective test
- [x] `docs-v2/ch11_long_task_timer.md` — tutorial chapter

---

### Feature 12: Function-Tracking Meters ❲Tier 3❳
✅ Complete

**API Contract** (what clients call):
```java
// Wrap an existing monotonic counter (e.g., from a library's internal stats)
Cache cache = createCache();
FunctionCounter hitCounter = FunctionCounter
    .builder("cache.hits", cache, Cache::getHitCount)
    .register(registry);

// Wrap existing timing stats (count + total time)
FunctionTimer functionTimer = FunctionTimer
    .builder("cache.gets", cache,
        Cache::getRequestCount,        // count function
        Cache::getTotalLoadTime,       // total time function
        TimeUnit.NANOSECONDS)
    .register(registry);

// Time-valued gauge
TimeGauge uptime = TimeGauge
    .builder("process.uptime", runtime,
        TimeUnit.MILLISECONDS,
        RuntimeMXBean::getUptime)
    .register(registry);
```

**What it does**: Lets clients wrap externally-managed counters, timers, and time values — when you can't call `increment()` or `record()` yourself because the source of truth lives in another library's API (cache stats, connection pool metrics, JMX beans).

**Depth layers**:

| Layer | Simplified Class                        | Real Framework Class                                                 | Purpose                     |
|-------|-----------------------------------------|----------------------------------------------------------------------|-----------------------------|
| API   | `FunctionCounter`                       | `io.micrometer.core.instrument.FunctionCounter`                      | Interface + Builder         |
| API   | `FunctionTimer`                         | `io.micrometer.core.instrument.FunctionTimer`                        | Interface + Builder         |
| API   | `TimeGauge`                             | `io.micrometer.core.instrument.TimeGauge`                            | Interface + Builder         |
| Impl  | `CumulativeFunctionCounter`             | `io.micrometer.core.instrument.cumulative.CumulativeFunctionCounter` | Wraps ToDoubleFunction      |
| Impl  | `CumulativeFunctionTimer`               | `io.micrometer.core.instrument.cumulative.CumulativeFunctionTimer`   | Wraps count+total functions |
| Impl  | `DefaultTimeGauge` (reuse DefaultGauge) | `io.micrometer.core.instrument.internal.DefaultGauge`                | Time-scaled gauge           |

**Real source files**: `FunctionCounter.java`, `FunctionTimer.java`, `TimeGauge.java`, `CumulativeFunctionCounter.java`, `CumulativeFunctionTimer.java`

**Depends on**: Features 1–3 (MeterRegistry, Gauge pattern, Timer pattern)
**Complexity**: Medium · 5 new classes, 2 modified (MeterRegistry.More, SimpleMeterRegistry)
**Java version**: 17

**Concrete deliverables**:
- [x] `api/FunctionCounter.java` — interface + Builder
- [x] `api/FunctionTimer.java` — interface + Builder
- [x] `api/TimeGauge.java` — interface + Builder
- [x] `internal/CumulativeFunctionCounter.java` — function-wrapping counter
- [x] `internal/CumulativeFunctionTimer.java` — function-wrapping timer
- [x] `api/FunctionMeterTest.java` — client-perspective test
- [x] `docs-v2/ch12_function_tracking_meters.md` — tutorial chapter

---

### Feature 13: Global Metrics Facade ❲Tier 2❳
✅ Complete

**API Contract** (what clients call):
```java
// Register backend registries with the global composite
Metrics.addRegistry(new SimpleMeterRegistry());
// Metrics.addRegistry(new PrometheusRegistry());

// Use static convenience methods anywhere in your code
Metrics.counter("app.events", "type", "login").increment();
Metrics.timer("app.processing").record(() -> doWork());
Metrics.gauge("app.queue.size", queue, Queue::size);

// Access the global registry directly if needed
CompositeMeterRegistry global = Metrics.globalRegistry;
```

**What it does**: Provides a static facade over a global `CompositeMeterRegistry` so that application code can record metrics without passing registry references around — the "just works" entry point for simple use cases.

**Depth layers**:

| Layer | Simplified Class | Real Framework Class                    | Purpose                   |
|-------|------------------|-----------------------------------------|---------------------------|
| API   | `Metrics`        | `io.micrometer.core.instrument.Metrics` | Static convenience facade |

**Real source files**: `Metrics.java`

**Depends on**: Features 1–4 (delegates all meter types), Feature 7 (uses CompositeMeterRegistry)
**Complexity**: Low · 1 new class
**Java version**: 17

**Concrete deliverables**:
- [x] `api/Metrics.java` — static global facade
- [x] `api/MetricsTest.java` — global registry test
- [x] `docs-v2/ch13_global_metrics_facade.md` — tutorial chapter

---

### Feature 14: Push Registry — Backend Export ❲Tier 3❳
✅ Complete

**API Contract** (what clients call):
```java
// Implementing a custom push-based backend:
public class LoggingMeterRegistry extends PushMeterRegistry {
    public LoggingMeterRegistry(PushRegistryConfig config, Clock clock) {
        super(config, clock);
    }

    @Override
    protected void publish() {
        getMeters().forEach(meter -> {
            System.out.println(meter.getId() + " → " + meter.measure());
        });
    }

    // ... implement newCounter(), newTimer(), etc.
}

// Client usage:
PushRegistryConfig config = key -> switch (key) {
    case "push.step" -> "PT10S";   // publish every 10 seconds
    default -> null;
};
LoggingMeterRegistry registry = new LoggingMeterRegistry(config);
registry.start();  // begin scheduled publishing

Counter c = registry.counter("events");
c.increment();
// After 10s, publish() is called automatically
registry.stop();
```

**What it does**: Provides the base class for monitoring backends that periodically push/export metrics (Datadog, InfluxDB, OTLP) — handles the scheduling, lifecycle, and base-time-unit conversion so backend implementors just override `publish()`.

**Depth layers**:

| Layer | Simplified Class       | Real Framework Class                                         | Purpose                      |
|-------|------------------------|--------------------------------------------------------------|------------------------------|
| API   | `PushMeterRegistry`    | `io.micrometer.core.instrument.push.PushMeterRegistry`       | Abstract push-based registry |
| API   | `PushRegistryConfig`   | `io.micrometer.core.instrument.push.PushRegistryConfig`      | Configuration interface      |
| Impl  | `LoggingMeterRegistry` | `io.micrometer.core.instrument.logging.LoggingMeterRegistry` | Example push registry        |

**Real source files**: `PushMeterRegistry.java`, `PushRegistryConfig.java`, `LoggingMeterRegistry.java`

**Depends on**: Features 1–4 (extends MeterRegistry, creates all meter types)
**Complexity**: Medium · 3 new classes
**Java version**: 17

**Concrete deliverables**:
- [x] `api/PushMeterRegistry.java` — abstract scheduled-publish registry
- [x] `api/PushRegistryConfig.java` — configuration with step/batchSize
- [x] `internal/LoggingMeterRegistry.java` — example push registry
- [x] `api/PushMeterRegistryTest.java` — publish lifecycle test
- [x] `docs-v2/ch14_push_registry.md` — tutorial chapter

---

### Feature 15: Observation API — Unified Instrumentation ❲Tier 3❳
✅ Complete

**API Contract** (what clients call):
```java
ObservationRegistry observationRegistry = ObservationRegistry.create();

// Register handlers (e.g., one that creates Timer metrics)
observationRegistry.observationConfig()
    .observationHandler(new SimpleTimerHandler(meterRegistry));

// Instrument a block of code
Observation.createNotStarted("http.request", observationRegistry)
    .lowCardinalityKeyValue("method", "GET")
    .highCardinalityKeyValue("uri", "/api/users/123")
    .observe(() -> handleRequest());

// Or manual lifecycle
Observation observation = Observation.start("db.query", observationRegistry);
try {
    observation.lowCardinalityKeyValue("db", "postgres");
    executeQuery();
} catch (Exception e) {
    observation.error(e);
    throw e;
} finally {
    observation.stop();
}
```

**What it does**: Provides a higher-level instrumentation API that decouples code instrumentation from the specific signal type — a single `Observation` can produce metrics (via Timer), traces (via Span), and logs simultaneously, based on which handlers are registered.

**Depth layers**:

| Layer | Simplified Class      | Real Framework Class                            | Purpose                    |
|-------|-----------------------|-------------------------------------------------|----------------------------|
| API   | `Observation`         | `io.micrometer.observation.Observation`         | Core observation interface |
| API   | `ObservationRegistry` | `io.micrometer.observation.ObservationRegistry` | Registry + config          |
| API   | `ObservationHandler`  | `io.micrometer.observation.ObservationHandler`  | Lifecycle callbacks        |
| API   | `Observation.Context` | `io.micrometer.observation.Observation.Context` | Observation state carrier  |
| Impl  | `SimpleObservation`   | `io.micrometer.observation.SimpleObservation`   | Default implementation     |

**Real source files**: `Observation.java`, `ObservationRegistry.java`, `ObservationHandler.java`, `SimpleObservation.java`, `ObservationConfig.java`

**Depends on**: Features 1, 3 (typically produces Counter/Timer metrics via handlers)
**Complexity**: High · 5 new classes
**Java version**: 17

**Concrete deliverables**:
- [x] `api/Observation.java` — observation interface + Context
- [x] `api/ObservationRegistry.java` — registry + config
- [x] `api/ObservationHandler.java` — lifecycle callback interface
- [x] `internal/SimpleObservation.java` — default implementation
- [x] `api/ObservationTest.java` — end-to-end observation test
- [x] `docs-v2/ch15_observation_api.md` — tutorial chapter

---

## Implementation Notes

### Simplification Strategy

| Real Framework Concept                                                 | Simplified Version                                         | Why Simplified                                                                                 |
|------------------------------------------------------------------------|------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| `PreFilterIdToMeterMap` cache + `meterMap` double lookup               | Single `ConcurrentHashMap<Meter.Id, Meter>`                | Optimization for hot-path re-registration; not needed to understand the API                    |
| `StepCounter` / `StepTimer` (rate-normalized)                          | Only `CumulativeCounter` / `CumulativeTimer`               | Step-normalization is a backend concern; cumulative is easier to reason about                  |
| `PauseDetector` (GC pause compensation)                                | Omitted                                                    | JVM-specific optimization that doesn't affect API understanding                                |
| `DoubleAdder` / `LongAdder`                                            | `AtomicDouble` / `AtomicLong` or simple `DoubleAdder`      | Same concept, we'll use java.util.concurrent.atomic directly                                   |
| `TimeWindowMax` with rotating bucfers                                  | Simple `AtomicLong`-based max with optional reset          | The rotation prevents stale max values; our simplified version is adequate for learning        |
| `SyntheticAssociations` (derivative meter tracking)                    | Omitted                                                    | Internal bookkeeping for histogram gauges; not visible through API                             |
| `MeterProvider` (since 1.12.0)                                         | Deferred                                                   | Convenience wrapper; builders + register() teach the same concepts                             |
| Full `module-info.java`                                                | Omitted                                                    | Module system boundaries not needed for learning                                               |
| Thread safety in `CompositeMeterRegistry` child management             | Simplified locking                                         | Real version handles concurrent add/remove with CopyOnWriteArrayList + careful synchronization |
| `ObservationConvention` / `ObservationFilter` / `ObservationPredicate` | Deferred                                                   | Advanced Observation customization; core lifecycle is sufficient                               |
| 23 registry implementations                                            | 1 (SimpleMeterRegistry) + 1 example (LoggingMeterRegistry) | Backend specifics are orthogonal to understanding the core API                                 |

### In Scope
- All Tier 1 meter types: Counter, Gauge, Timer, DistributionSummary
- Core registration lifecycle: Builder → register() → meter
- Meter.Id with dimensional Tags
- MeterFilter chain (map, accept, configure)
- NamingConvention formatting
- Composite pattern for multi-backend publishing
- Search/discovery API
- MeterBinder pattern
- Distribution statistics: percentiles and histograms
- LongTaskTimer for in-flight work
- Function-tracking meters (FunctionCounter, FunctionTimer, TimeGauge)
- Global Metrics facade
- PushMeterRegistry base for backend export
- Observation API core lifecycle

### Out of Scope
- Backend-specific registry implementations (Prometheus, Datadog, etc.)
- Step-normalized meters (StepCounter, StepTimer)
- GC pause detection and compensation
- Kotlin extensions
- AspectJ/AOP integration (@Timed, @Counted aspects)
- micrometer-tracing (Span, Tracer — separate repo)
- context-propagation (ContextSnapshot — separate repo)
- MeterProvider and dynamic tag binding
- Exemplar support
- High-cardinality tag detection
- OSGi support

### Java Version
- Base: Java 17
- All features use Java 17

## Status Markers
- ⬜ = not started
- ✅ = implemented, tests passing, tutorial written
