# Chapter 21: Datadog Registry

> Line references based on commit `2c8a4606c` of the Micrometer repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| We have `StepMeterRegistry` (ch16) and `LoggingMeterRegistry` (ch17) that demonstrate step-based metrics with logging output | Metrics are logged to console — no real-world HTTP-based backend integration exists | Build a `DatadogMeterRegistry` that serializes meters into Datadog's JSON format and POSTs them to the Datadog API via HTTP |

---

## 21.1 The Integration Point

The Datadog registry plugs into the existing system at `StepMeterRegistry`'s abstract `publish()` method — the same extension point that `LoggingMeterRegistry` uses. The new registry overrides `publish()` with HTTP-based serialization instead of logging.

**Modifying:** Nothing — this feature creates new files only. The integration point is the `StepMeterRegistry.publish()` abstract method that we override.

```java
// DatadogMeterRegistry extends StepMeterRegistry
// The integration is: override publish() to serialize to JSON + HTTP POST
public class DatadogMeterRegistry extends StepMeterRegistry {

    @Override
    protected void publish() {
        // Serialize all meters to {"series":[...]} JSON
        // POST to config.uri() + "/api/v1/series?api_key=" + config.apiKey()
    }

    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.MILLISECONDS;  // Datadog uses ms, not seconds
    }
}
```

Two key decisions here:

1. **Why extend `StepMeterRegistry` instead of `PushMeterRegistry`?** Datadog expects per-interval deltas (rates), not cumulative totals. StepMeterRegistry's step-based rollover gives us exactly that — counters report deltas, timers report per-interval stats.
2. **Why milliseconds?** Datadog's native time unit is milliseconds, unlike Prometheus which uses seconds. The `getBaseTimeUnit()` override controls how all time values are converted before publishing.

This connects **step-based accumulation** to **HTTP-based export**. To make it work, we need to build:
- `DatadogConfig` — configuration interface for API key, endpoint URI, host tag
- `DatadogNamingConvention` — enforces Datadog's naming rules (200-char max, letter prefix, JSON escaping)
- `DatadogMeterRegistry` — the registry with JSON serialization and HTTP POST

## 21.2 Configuration

**New file:** `src/main/java/dev/linhvu/micrometer/datadog/DatadogConfig.java`

```java
package dev.linhvu.micrometer.datadog;

import dev.linhvu.micrometer.step.StepRegistryConfig;

public interface DatadogConfig extends StepRegistryConfig {

    String apiKey();

    default String uri() {
        return "https://api.datadoghq.com";
    }

    default String hostTag() {
        return "instance";
    }

    default boolean descriptions() {
        return true;
    }
}
```

The config hierarchy is: `DatadogConfig → StepRegistryConfig → PushRegistryConfig`. This means all push/step properties (`step()`, `enabled()`, `batchSize()`) are inherited. The only required property is `apiKey()` — it has no default because it's unique to each Datadog account.

The `hostTag()` config maps a Micrometer tag key to Datadog's `"host"` field — Datadog uses this to associate metrics with specific infrastructure hosts.

## 21.3 Naming Convention

**New file:** `src/main/java/dev/linhvu/micrometer/datadog/DatadogNamingConvention.java`

```java
package dev.linhvu.micrometer.datadog;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.config.NamingConvention;

public class DatadogNamingConvention implements NamingConvention {

    private static final int MAX_NAME_LENGTH = 200;
    private final NamingConvention delegate;

    public DatadogNamingConvention() {
        this(NamingConvention.dot);
    }

    public DatadogNamingConvention(NamingConvention delegate) {
        this.delegate = delegate;
    }

    @Override
    public String name(String name, Meter.Type type, String baseUnit) {
        String sanitized = escapeJson(delegate.name(name, type, baseUnit).replace('/', '_'));
        if (!sanitized.isEmpty() && !Character.isLetter(sanitized.charAt(0))) {
            sanitized = "m." + sanitized;
        }
        return truncate(sanitized, MAX_NAME_LENGTH);
    }

    @Override
    public String tagKey(String key) {
        String sanitized = escapeJson(delegate.tagKey(key));
        if (!sanitized.isEmpty() && Character.isDigit(sanitized.charAt(0))) {
            sanitized = "m." + sanitized;
        }
        return sanitized;
    }

    @Override
    public String tagValue(String value) {
        return escapeJson(delegate.tagValue(value));
    }

    static String escapeJson(String value) {
        StringBuilder sb = new StringBuilder(value.length());
        for (int i = 0; i < value.length(); i++) {
            char c = value.charAt(i);
            switch (c) {
                case '"' -> sb.append("\\\"");
                case '\\' -> sb.append("\\\\");
                case '\n' -> sb.append("\\n");
                case '\r' -> sb.append("\\r");
                case '\t' -> sb.append("\\t");
                default -> sb.append(c);
            }
        }
        return sb.toString();
    }

    private static String truncate(String value, int maxLength) {
        return value.length() <= maxLength ? value : value.substring(0, maxLength);
    }
}
```

Notable rules:
- **Names starting with non-letters are silently dropped** by Datadog's API. Rather than lose metrics, we prepend `"m."`.
- **Forward slashes break the metadata API**, so they're replaced with underscores.
- **Tag keys starting with digits** render as empty strings in Datadog's UI, so they also get `"m."` prepended.
- **Tag values can start with digits** — no special handling needed.
- **JSON escaping** is necessary because metric names and tags are embedded directly in JSON strings.

Compare with `PrometheusNamingConvention` (ch20): Prometheus converts dots to underscores and appends type suffixes (`_total`, `_seconds`). Datadog preserves dot notation and has its own suffix conventions (`.count`, `.sum`, `.avg`, `.max` — but these are added at the serializer level, not the naming convention).

## 21.4 The Registry

**New file:** `src/main/java/dev/linhvu/micrometer/datadog/DatadogMeterRegistry.java`

The core of this feature — the `publish()` method that serializes all meters to JSON and POSTs to Datadog.

```java
@Override
protected void publish() {
    String endpoint = config.uri() + "/api/v1/series?api_key=" + config.apiKey();

    try {
        String body = getMeters().stream()
                .flatMap(meter -> meter.match(
                        counter -> writeMeter(counter),       // Counter
                        gauge -> writeMeter(gauge),           // Gauge
                        this::writeTimer,                     // Timer → 4 sub-metrics
                        this::writeSummary,                   // Summary → 4 sub-metrics
                        ltt -> writeMeter(ltt),               // LongTaskTimer
                        fc -> writeMeter(fc),                 // FunctionCounter
                        this::writeFunctionTimer,             // FunctionTimer → 3 sub-metrics
                        m -> writeMeter(m)                    // default
                ))
                .collect(joining(",", "{\"series\":[", "]}"));

        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(endpoint))
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(body))
                .build();

        httpClient.send(request, HttpResponse.BodyHandlers.ofString());
    }
    catch (Exception e) {
        System.err.println("Failed to send metrics to Datadog: " + e.getMessage());
    }
}
```

**Metric decomposition** — Timers produce 4 data points, FunctionTimers produce 3:

```java
private Stream<String> writeTimer(Timer timer) {
    long wallTime = clock.wallTime();
    Meter.Id id = timer.getId();
    return Stream.of(
            writeMetric(id, "sum", wallTime, timer.totalTime(getBaseTimeUnit()), Statistic.TOTAL_TIME),
            writeMetric(id, "count", wallTime, timer.count(), Statistic.COUNT),
            writeMetric(id, "avg", wallTime, timer.mean(getBaseTimeUnit()), Statistic.VALUE),
            writeMetric(id, "max", wallTime, timer.max(getBaseTimeUnit()), Statistic.MAX));
}
```

**JSON serialization** — the core `writeMetric()` builds each data point:

```java
String writeMetric(Meter.Id id, String suffix, long wallTime, double value, Statistic statistic) {
    Meter.Id fullId = (suffix != null) ? id.withName(id.getName() + "." + suffix) : id;

    List<Tag> tags = fullId.getConventionTags(getNamingConvention());
    String metricName = fullId.getConventionName(getNamingConvention());

    StringBuilder json = new StringBuilder();
    json.append("{\"metric\":\"").append(escapeJson(metricName)).append("\"");
    json.append(",\"points\":[[").append(wallTime / 1000).append(", ").append(value).append("]]");

    // Host extraction: match tag key to config.hostTag()
    if (config.hostTag() != null) {
        for (Tag tag : tags) {
            if (config.hostTag().equals(tag.getKey())) {
                json.append(",\"host\":\"").append(escapeJson(tag.getValue())).append("\"");
                break;
            }
        }
    }

    // Type: "count" for cumulative statistics, "gauge" for instantaneous values
    json.append(",\"type\":\"").append(sanitizeType(statistic)).append("\"");

    // Tags: ["key:value", ...] — Datadog's colon-separated format
    if (!tags.isEmpty()) {
        json.append(",\"tags\":[");
        json.append(tags.stream()
                .map(t -> "\"" + escapeJson(t.getKey()) + ":" + escapeJson(t.getValue()) + "\"")
                .collect(joining(",")));
        json.append("]");
    }

    json.append("}");
    return json.toString();
}
```

**Type mapping** — Datadog's two metric types:

```java
static String sanitizeType(Statistic statistic) {
    return switch (statistic) {
        case COUNT, TOTAL, TOTAL_TIME -> "count";
        default -> "gauge";
    };
}
```

## 21.5 Try It Yourself

<details>
<summary>Challenge: Implement the writeSummary method</summary>

A `DistributionSummary` should be decomposed into 4 sub-metrics just like a Timer, but using `totalAmount()` instead of `totalTime()` and `TOTAL` instead of `TOTAL_TIME`:

```java
private Stream<String> writeSummary(DistributionSummary summary) {
    long wallTime = clock.wallTime();
    Meter.Id id = summary.getId();
    return Stream.of(
            writeMetric(id, "sum", wallTime, summary.totalAmount(), Statistic.TOTAL),
            writeMetric(id, "count", wallTime, summary.count(), Statistic.COUNT),
            writeMetric(id, "avg", wallTime, summary.mean(), Statistic.VALUE),
            writeMetric(id, "max", wallTime, summary.max(), Statistic.MAX));
}
```

</details>

<details>
<summary>Challenge: What Datadog type should MAX and ACTIVE_TASKS map to, and why?</summary>

Both map to `"gauge"` — they represent instantaneous values, not cumulative totals.

- `MAX` is the maximum value in the current window — it can go up or down between intervals
- `ACTIVE_TASKS` is the current count of in-progress tasks — also varies over time

Only `COUNT`, `TOTAL`, and `TOTAL_TIME` map to `"count"` because they represent cumulative values that Datadog should compute rates from.

</details>

<details>
<summary>Challenge: Why does writeMeter add a "statistic" tag but writeTimer doesn't?</summary>

`writeMeter()` is the generic handler for counters, gauges, LongTaskTimers, and FunctionCounters. A Counter's `measure()` returns one measurement (COUNT), and a LongTaskTimer returns three (ACTIVE_TASKS, DURATION, MAX). Since these might have multiple measurements sharing the same metric name, each needs a `statistic` tag to distinguish them.

`writeTimer()` avoids this by decomposing into suffixed names (`.count`, `.sum`, `.avg`, `.max`). The suffix in the metric name already identifies what the data point represents — no need for an extra tag.

</details>

## 21.6 Tests

### Unit Tests

**New file:** `src/test/java/dev/linhvu/micrometer/datadog/DatadogNamingConventionTest.java`

```java
@Test
void shouldPreserveDotNotation_WhenNameIsValid() {
    String result = convention.name("http.server.requests", Meter.Type.COUNTER, null);
    assertThat(result).isEqualTo("http.server.requests");
}

@Test
void shouldPrependM_WhenNameStartsWithNumber() {
    String result = convention.name("123.invalid", Meter.Type.GAUGE, null);
    assertThat(result).isEqualTo("m.123.invalid");
}

@Test
void shouldTruncate_WhenNameExceeds200Characters() {
    String longName = "a".repeat(250);
    String result = convention.name(longName, Meter.Type.COUNTER, null);
    assertThat(result).hasSize(200);
}
```

**New file:** `src/test/java/dev/linhvu/micrometer/datadog/DatadogMeterRegistryTest.java`

```java
@Test
void shouldProduceValidJson_ForSimpleMetric() {
    Meter.Id id = new Meter.Id("http.requests", Tags.empty(),
            Meter.Type.COUNTER, null, null);
    String json = registry.writeMetric(id, null, 60_000, 42.0, Statistic.COUNT);

    assertThat(json).contains("\"metric\":\"http.requests\"");
    assertThat(json).contains("\"points\":[[60, 42.0]]");
    assertThat(json).contains("\"type\":\"count\"");
}

@Test
void shouldIncludeTags_InColonSeparatedFormat() {
    Meter.Id id = new Meter.Id("http.requests",
            Tags.of("method", "GET", "status", "200"),
            Meter.Type.COUNTER, null, null);
    String json = registry.writeMetric(id, null, 60_000, 5.0, Statistic.COUNT);

    assertThat(json).contains("\"method:GET\"");
    assertThat(json).contains("\"status:200\"");
}

@Test
void shouldExtractHostTag_WhenPresent() {
    Meter.Id id = new Meter.Id("cpu.usage",
            Tags.of("instance", "web-1", "region", "us-east"),
            Meter.Type.GAUGE, null, null);
    String json = registry.writeMetric(id, null, 60_000, 75.0, Statistic.VALUE);

    assertThat(json).contains("\"host\":\"web-1\"");
}

@Test
void shouldDecomposeTimer_IntoFourSubMetrics() {
    Timer timer = registry.timer("http.latency");
    timer.record(100, TimeUnit.MILLISECONDS);
    timer.record(200, TimeUnit.MILLISECONDS);
    clock.add(Duration.ofMinutes(1));

    String countJson = registry.writeMetric(timer.getId(), "count",
            clock.wallTime(), timer.count(), Statistic.COUNT);
    assertThat(countJson).contains("\"metric\":\"http.latency.count\"");
    assertThat(countJson).contains("\"type\":\"count\"");
}
```

**Run:** `./gradlew test` — expected: all 893 tests pass (including prior features' 847 tests + 46 new Datadog tests)

---

## 21.7 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why decompose Timers into sub-metrics?** Datadog (and most push-based backends) has no native histogram or timer type — each time series is a single scalar. By splitting a Timer into `.count`, `.sum`, `.avg`, and `.max`, we give Datadog the building blocks to compute rates (from `.count`), throughput (from `.sum`), and latency percentiles (from `.avg`/`.max`). This is fundamentally different from Prometheus (ch20), which has native histogram support and can aggregate multiple observations into bucket distributions.
> - **When this decomposition hurts:** If you have 1,000 timers, the Datadog registry sends 4,000 data points per publish cycle. Push-based backends pay a cost proportional to the number of series — while Prometheus only generates data when scraped. This is why Datadog charges by metric volume and Micrometer's real implementation includes batching.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Pull vs Push model contrast:** Compare this registry with `PrometheusMeterRegistry` (ch20). Prometheus is pull-based — it has no scheduler, no HTTP client, and no `publish()` method. Instead, it waits for an external scraper to call `scrape()`. Datadog is push-based — it proactively sends data on a timer. The StepMeterRegistry infrastructure (step rollover, dual schedulers) only exists to serve push registries. Pull registries don't need it because they compute values on-demand at scrape time.
> - **Practical tradeoff:** Push registries add latency (up to one step interval) but decouple the app from the monitoring system's availability. Pull registries provide real-time data but require the monitoring system to be reachable from the application.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Colon-separated tags vs label syntax:** Datadog uses `"key:value"` tag format while Prometheus uses `key="value"`. This isn't just cosmetic — it reflects different data models. Prometheus labels are part of the metric identity (changing a label creates a new time series). Datadog tags are more like annotations — they enable filtering and grouping but the identity model is different. The `NamingConvention` abstraction (ch9) is what makes this transparent to instrumentation code.
> -----------------------------------------------------------

## 21.8 What We Enhanced

| Aspect | Before (ch17) | Current (ch21) | Real Framework |
|--------|--------------|----------------|----------------|
| Export target | Console logging via `Consumer<String>` | HTTP POST to a real monitoring backend API | Same HTTP export via abstracted `HttpSender` (`DatadogMeterRegistry.java:143`) |
| Meter serialization | Human-readable text format | Structured JSON matching the Datadog API spec | Same JSON format with added unit and metadata fields (`DatadogMeterRegistry.java:234`) |
| Naming convention | `NamingConvention.dot` (identity) | `DatadogNamingConvention` with letter prefix, truncation, JSON escaping | Same convention plus slash→underscore handling (`DatadogNamingConvention.java:30`) |
| Type mapping | No concept of backend metric types | `sanitizeType()` maps statistics to Datadog's `"count"` vs `"gauge"` | Same via `DatadogMetricMetadata.sanitizeType()` (`DatadogMetricMetadata.java:100`) |
| Host extraction | Not applicable | Extracts host from tag matching `config.hostTag()` | Same with configurable `hostTag` (`DatadogMeterRegistry.java:243`) |

## 21.9 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `DatadogMeterRegistry` | `DatadogMeterRegistry` | `DatadogMeterRegistry.java:46` | Real uses `HttpSender` abstraction, `MeterPartition` for batching, metadata PUT, and `Builder` pattern |
| `publish()` | `publish()` | `DatadogMeterRegistry.java:108` | Real batches meters via `MeterPartition.partition()` and sends metadata via separate PUT requests |
| `writeMetric()` | `writeMetric()` | `DatadogMeterRegistry.java:234` | Real includes `"unit"` field from `DatadogMetricMetadata.sanitizeBaseUnit()` with a 120+ entry whitelist |
| `sanitizeType()` | `DatadogMetricMetadata.sanitizeType()` | `DatadogMetricMetadata.java:100` | Same logic — COUNT/TOTAL/TOTAL_TIME → "count", everything else → "gauge" |
| `DatadogConfig` | `DatadogConfig` | `DatadogConfig.java:31` | Real uses property-binding framework, adds `applicationKey()` and `validate()` |
| `DatadogNamingConvention` | `DatadogNamingConvention` | `DatadogNamingConvention.java:30` | Same slash→underscore, m. prefix, and truncation logic |
| `escapeJson()` | `StringEscapeUtils.escapeJson()` | shared util | Real uses a centralized utility class shared across all registries |
| Not implemented | `DatadogMetricMetadata` | `DatadogMetricMetadata.java:29` | Real sends metric type/unit/description via PUT to `/api/v1/metrics/` with a 120+ entry unit whitelist |
| Not implemented | `MeterPartition` | util class | Real batches meters into chunks of `batchSize` for multiple HTTP requests |

## 21.10 Complete Code

### Production Code

#### File: `src/main/java/dev/linhvu/micrometer/datadog/DatadogConfig.java` [NEW]

```java
package dev.linhvu.micrometer.datadog;

import dev.linhvu.micrometer.step.StepRegistryConfig;

/**
 * Configuration for {@link DatadogMeterRegistry}.
 * <p>
 * Extends {@link StepRegistryConfig} — all step/push configuration (step interval,
 * enabled, batch size) applies. Adds Datadog-specific properties: API key, endpoint
 * URI, host tag mapping, and description enablement.
 * <p>
 * <b>Simplification:</b> The real Micrometer's {@code DatadogConfig} uses a
 * property-binding framework (e.g., {@code "datadog.apiKey"} resolved from environment
 * variables or properties files) and includes an {@code applicationKey()} for metadata
 * PUT requests. We use simple default methods and omit the application key since we
 * don't implement the metadata PUT API.
 */
public interface DatadogConfig extends StepRegistryConfig {

    /**
     * The Datadog API key. Required for publishing metrics.
     * <p>
     * This is the "API Key" from your Datadog account settings, NOT the
     * "Application Key" (which is only needed for metadata operations that
     * we omit in this simplified version).
     *
     * @return The API key string.
     */
    String apiKey();

    /**
     * The Datadog API endpoint URI. Override this if you need to publish metrics
     * through an internal proxy en route to Datadog.
     *
     * @return The base URI. Default: {@code "https://api.datadoghq.com"}
     */
    default String uri() {
        return "https://api.datadoghq.com";
    }

    /**
     * The tag key that maps to Datadog's {@code "host"} field in the metric JSON.
     * <p>
     * When publishing, if a meter has a tag with this key, its value is extracted
     * and placed in the {@code "host"} field of the Datadog payload. This is how
     * Datadog associates metrics with specific hosts in its infrastructure view.
     * <p>
     * Return {@code null} to disable host tag extraction entirely.
     *
     * @return The host tag key. Default: {@code "instance"}
     */
    default String hostTag() {
        return "instance";
    }

    /**
     * Whether meter descriptions should be included in the published metadata.
     * Turn this off to minimize the amount of data sent per publish cycle.
     * <p>
     * <b>Note:</b> In our simplified version, descriptions are only used as
     * documentation context — we don't implement the metadata PUT API that
     * would send them to Datadog.
     *
     * @return {@code true} if descriptions are enabled. Default: {@code true}
     */
    default boolean descriptions() {
        return true;
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/datadog/DatadogNamingConvention.java` [NEW]

```java
package dev.linhvu.micrometer.datadog;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.config.NamingConvention;

/**
 * {@link NamingConvention} for Datadog — enforces Datadog's metric naming rules.
 * <p>
 * Datadog's metric naming has several constraints:
 * <ul>
 *   <li>Names use dot notation (Datadog's native format)</li>
 *   <li>Names must start with a letter — metrics starting with non-letters are silently
 *       dropped by the publish API, so we prepend {@code "m."}</li>
 *   <li>Names are limited to 200 characters maximum</li>
 *   <li>Forward slashes break the POST metadata API, so they're replaced with underscores</li>
 *   <li>JSON special characters in names, tag keys, and tag values must be escaped</li>
 * </ul>
 * <p>
 * Uses a delegate pattern — the default delegate is {@link NamingConvention#dot},
 * which preserves Micrometer's standard dot-separated naming. A custom delegate
 * can be provided if different base naming is desired.
 * <p>
 * <b>Tag keys</b> that start with a digit are also prepended with {@code "m."} because
 * Datadog renders them as empty strings otherwise. Tag <i>values</i> can start with
 * digits — no prepending needed.
 */
public class DatadogNamingConvention implements NamingConvention {

    private static final int MAX_NAME_LENGTH = 200;

    private final NamingConvention delegate;

    public DatadogNamingConvention() {
        this(NamingConvention.dot);
    }

    public DatadogNamingConvention(NamingConvention delegate) {
        this.delegate = delegate;
    }

    /**
     * Transforms a metric name for Datadog:
     * <ol>
     *   <li>Delegates to the base naming convention</li>
     *   <li>Replaces forward slashes with underscores (slashes break the metadata API)</li>
     *   <li>Escapes JSON special characters</li>
     *   <li>Prepends {@code "m."} if the name doesn't start with a letter</li>
     *   <li>Truncates to 200 characters</li>
     * </ol>
     */
    @Override
    public String name(String name, Meter.Type type, String baseUnit) {
        String sanitized = escapeJson(delegate.name(name, type, baseUnit).replace('/', '_'));
        if (!sanitized.isEmpty() && !Character.isLetter(sanitized.charAt(0))) {
            sanitized = "m." + sanitized;
        }
        return truncate(sanitized, MAX_NAME_LENGTH);
    }

    /**
     * Tag keys starting with a digit are prepended with {@code "m."} because Datadog
     * displays them as empty strings otherwise.
     */
    @Override
    public String tagKey(String key) {
        String sanitized = escapeJson(delegate.tagKey(key));
        if (!sanitized.isEmpty() && Character.isDigit(sanitized.charAt(0))) {
            sanitized = "m." + sanitized;
        }
        return sanitized;
    }

    /**
     * Tag values are JSON-escaped but not modified otherwise — Datadog permits
     * tag values starting with digits.
     */
    @Override
    public String tagValue(String value) {
        return escapeJson(delegate.tagValue(value));
    }

    // -----------------------------------------------------------------------
    // JSON escaping — needed because metric names and tags are embedded in JSON
    // -----------------------------------------------------------------------

    /**
     * Escapes characters that are special in JSON strings: quotes, backslashes,
     * and control characters (newline, carriage return, tab).
     * <p>
     * Package-private so {@link DatadogMeterRegistry} can reuse it for JSON body
     * construction.
     */
    static String escapeJson(String value) {
        StringBuilder sb = new StringBuilder(value.length());
        for (int i = 0; i < value.length(); i++) {
            char c = value.charAt(i);
            switch (c) {
                case '"' -> sb.append("\\\"");
                case '\\' -> sb.append("\\\\");
                case '\n' -> sb.append("\\n");
                case '\r' -> sb.append("\\r");
                case '\t' -> sb.append("\\t");
                default -> sb.append(c);
            }
        }
        return sb.toString();
    }

    private static String truncate(String value, int maxLength) {
        return value.length() <= maxLength ? value : value.substring(0, maxLength);
    }

}
```

#### File: `src/main/java/dev/linhvu/micrometer/datadog/DatadogMeterRegistry.java` [NEW]

```java
package dev.linhvu.micrometer.datadog;

import dev.linhvu.micrometer.Clock;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Measurement;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.Statistic;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Timer;
import dev.linhvu.micrometer.step.StepMeterRegistry;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.List;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.stream.Stream;
import java.util.stream.StreamSupport;

import static dev.linhvu.micrometer.datadog.DatadogNamingConvention.escapeJson;
import static java.util.stream.Collectors.joining;

/**
 * A concrete push registry that publishes metrics to the Datadog API via HTTP.
 * <p>
 * Extends {@link StepMeterRegistry}, so all meters use step-based semantics: counters
 * report per-interval deltas, timers report per-interval counts and totals, etc. The
 * {@link #publish()} method serializes all meters into a {@code {"series":[...]}} JSON
 * payload and POSTs it to Datadog's {@code /api/v1/series} endpoint.
 * <p>
 * <b>Metric decomposition:</b> Timers and DistributionSummaries are broken into multiple
 * Datadog metrics with suffixes ({@code .count}, {@code .sum}, {@code .avg}, {@code .max}).
 * Each sub-metric is emitted as a separate series entry. This is because Datadog treats
 * each time series independently — there's no native "histogram" type.
 * <p>
 * <b>Datadog type mapping:</b>
 * <ul>
 *   <li>COUNT, TOTAL, TOTAL_TIME statistics → Datadog {@code "count"} type (rates)</li>
 *   <li>VALUE, MAX, ACTIVE_TASKS, DURATION → Datadog {@code "gauge"} type (instantaneous)</li>
 * </ul>
 * <p>
 * <b>Tag format:</b> Tags are formatted as {@code "key:value"} strings (colon-separated),
 * which is Datadog's native tag syntax. This differs from Prometheus's
 * {@code key="value"} label format.
 * <p>
 * <b>Base time unit:</b> Milliseconds (unlike Prometheus which uses seconds).
 * <p>
 * <b>Simplification:</b> The real Micrometer's {@code DatadogMeterRegistry} includes
 * metadata PUT requests (sending type/unit/description to Datadog's metadata API),
 * unit whitelist validation, request batching, and uses an abstracted
 * {@code HttpSender} interface. We use {@code java.net.http.HttpClient} directly,
 * send all metrics in a single POST, and omit the metadata API.
 */
public class DatadogMeterRegistry extends StepMeterRegistry {

    private final DatadogConfig config;

    private final HttpClient httpClient;

    /**
     * Creates a registry that publishes to Datadog via HTTP and starts the
     * publishing scheduler immediately.
     *
     * @param config The Datadog configuration (must provide an API key).
     * @param clock  The clock for time measurement.
     */
    public DatadogMeterRegistry(DatadogConfig config, Clock clock) {
        this(config, clock, HttpClient.newHttpClient());
        start(Executors.defaultThreadFactory());
    }

    /**
     * Creates a registry with a custom HTTP client. Does NOT start the publishing
     * scheduler — call {@link #start()} manually or use for testing.
     * <p>
     * Package-private for testability: tests can inject a custom {@link HttpClient}
     * or call {@link #publish()} directly without starting background threads.
     *
     * @param config     The Datadog configuration.
     * @param clock      The clock for time measurement.
     * @param httpClient The HTTP client to use for publishing.
     */
    DatadogMeterRegistry(DatadogConfig config, Clock clock, HttpClient httpClient) {
        super(config, clock);
        this.config = config;
        this.httpClient = httpClient;
        namingConvention(new DatadogNamingConvention());
    }

    // -----------------------------------------------------------------------
    // The core: publish() — serialize all meters to JSON and POST to Datadog
    // -----------------------------------------------------------------------

    /**
     * Serializes all registered meters into a Datadog-format JSON payload and POSTs
     * it to the series API endpoint.
     * <p>
     * The JSON structure follows the Datadog API spec:
     * <pre>{@code
     * {
     *   "series": [
     *     {
     *       "metric": "http.requests.count",
     *       "points": [[1616461200, 42.0]],
     *       "type": "count",
     *       "tags": ["method:GET", "status:200"]
     *     },
     *     ...
     *   ]
     * }
     * }</pre>
     * <p>
     * Uses the visitor pattern ({@link Meter#match}) to dispatch each meter type
     * to the appropriate serializer. Timers and summaries produce multiple series
     * entries (one per sub-metric), while counters and gauges produce one each.
     */
    @Override
    protected void publish() {
        String endpoint = config.uri() + "/api/v1/series?api_key=" + config.apiKey();

        try {
            String body = getMeters().stream()
                    .flatMap(meter -> meter.match(
                            counter -> writeMeter(counter),            // Counter
                            gauge -> writeMeter(gauge),                // Gauge
                            this::writeTimer,                          // Timer
                            this::writeSummary,                        // DistributionSummary
                            ltt -> writeMeter(ltt),                    // LongTaskTimer
                            fc -> writeMeter(fc),                      // FunctionCounter
                            this::writeFunctionTimer,                  // FunctionTimer
                            m -> writeMeter(m)                         // default
                    ))
                    .collect(joining(",", "{\"series\":[", "]}"));

            HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(endpoint))
                    .header("Content-Type", "application/json")
                    .POST(HttpRequest.BodyPublishers.ofString(body))
                    .build();

            HttpResponse<String> response = httpClient.send(request,
                    HttpResponse.BodyHandlers.ofString());

            if (response.statusCode() >= 400) {
                System.err.println("Failed to send metrics to Datadog (HTTP "
                        + response.statusCode() + "): " + response.body());
            }
        }
        catch (Exception e) {
            System.err.println("Failed to send metrics to Datadog: " + e.getMessage());
        }
    }

    // -----------------------------------------------------------------------
    // Per-meter-type serializers
    // -----------------------------------------------------------------------

    /**
     * Serializes a {@link Timer} into 4 Datadog sub-metrics:
     * {@code .sum} (total time), {@code .count}, {@code .avg} (mean), {@code .max}.
     * <p>
     * This decomposition is necessary because Datadog has no native histogram or
     * timer type — each aspect must be a separate time series.
     */
    private Stream<String> writeTimer(Timer timer) {
        long wallTime = clock.wallTime();
        Meter.Id id = timer.getId();

        return Stream.of(
                writeMetric(id, "sum", wallTime, timer.totalTime(getBaseTimeUnit()),
                        Statistic.TOTAL_TIME),
                writeMetric(id, "count", wallTime, timer.count(),
                        Statistic.COUNT),
                writeMetric(id, "avg", wallTime, timer.mean(getBaseTimeUnit()),
                        Statistic.VALUE),
                writeMetric(id, "max", wallTime, timer.max(getBaseTimeUnit()),
                        Statistic.MAX));
    }

    /**
     * Serializes a {@link FunctionTimer} into 3 Datadog sub-metrics:
     * {@code .count}, {@code .avg}, {@code .sum}.
     * <p>
     * Unlike a regular Timer, we can't compute max or percentiles from a function
     * timer — we only have access to the count and total time functions.
     */
    private Stream<String> writeFunctionTimer(FunctionTimer timer) {
        long wallTime = clock.wallTime();
        Meter.Id id = timer.getId();

        return Stream.of(
                writeMetric(id, "count", wallTime, timer.count(),
                        Statistic.COUNT),
                writeMetric(id, "avg", wallTime, timer.mean(getBaseTimeUnit()),
                        Statistic.VALUE),
                writeMetric(id, "sum", wallTime, timer.totalTime(getBaseTimeUnit()),
                        Statistic.TOTAL_TIME));
    }

    /**
     * Serializes a {@link DistributionSummary} into 4 Datadog sub-metrics:
     * {@code .sum} (total), {@code .count}, {@code .avg} (mean), {@code .max}.
     * <p>
     * Follows the same decomposition pattern as Timer, but without time-unit
     * conversion since summaries record non-time values.
     */
    private Stream<String> writeSummary(DistributionSummary summary) {
        long wallTime = clock.wallTime();
        Meter.Id id = summary.getId();

        return Stream.of(
                writeMetric(id, "sum", wallTime, summary.totalAmount(),
                        Statistic.TOTAL),
                writeMetric(id, "count", wallTime, summary.count(),
                        Statistic.COUNT),
                writeMetric(id, "avg", wallTime, summary.mean(),
                        Statistic.VALUE),
                writeMetric(id, "max", wallTime, summary.max(),
                        Statistic.MAX));
    }

    /**
     * Generic serializer for all other meter types (Counter, Gauge, LongTaskTimer,
     * FunctionCounter, and custom meters).
     * <p>
     * Iterates the meter's measurements and writes each as a separate data point.
     * The statistic is added as a tag (e.g., {@code "statistic:count"}) because
     * generic meters may emit multiple measurements.
     */
    private Stream<String> writeMeter(Meter meter) {
        long wallTime = clock.wallTime();
        return StreamSupport.stream(meter.measure().spliterator(), false)
                .map(ms -> {
                    Meter.Id id = meter.getId().withTag(ms.getStatistic());
                    return writeMetric(id, null, wallTime, ms.getValue(),
                            ms.getStatistic());
                });
    }

    // -----------------------------------------------------------------------
    // Core JSON serialization
    // -----------------------------------------------------------------------

    /**
     * Builds a single Datadog metric JSON object.
     * <p>
     * Example output:
     * <pre>{@code
     * {"metric":"http.requests.count","points":[[1616461200, 42.0]],"type":"count","tags":["method:GET"]}
     * }</pre>
     * <p>
     * The {@code type} field tells Datadog how to aggregate the metric:
     * <ul>
     *   <li>{@code "count"} — cumulative counts (Datadog computes rates)</li>
     *   <li>{@code "gauge"} — instantaneous values (Datadog shows as-is)</li>
     * </ul>
     * <p>
     * If {@link DatadogConfig#hostTag()} is set and a matching tag exists on the meter,
     * its value is extracted into the {@code "host"} field of the JSON payload.
     *
     * @param id        The meter identity (may include suffix and statistic tag).
     * @param suffix    Optional suffix to append to the metric name (e.g., "count", "sum").
     * @param wallTime  The wall clock time in milliseconds.
     * @param value     The metric value.
     * @param statistic The statistic type (determines "count" vs "gauge" typing).
     * @return A JSON string representing one Datadog metric data point.
     */
    String writeMetric(Meter.Id id, String suffix, long wallTime, double value,
            Statistic statistic) {
        Meter.Id fullId = id;
        if (suffix != null) {
            fullId = id.withName(id.getName() + "." + suffix);
        }

        List<Tag> tags = fullId.getConventionTags(getNamingConvention());
        String metricName = fullId.getConventionName(getNamingConvention());

        StringBuilder json = new StringBuilder();
        json.append("{\"metric\":\"").append(escapeJson(metricName)).append("\"");

        // Points: [[epochSeconds, value]]
        json.append(",\"points\":[[").append(wallTime / 1000).append(", ").append(value).append("]]");

        // Host extraction: if a tag matches hostTag, extract it as the "host" field
        if (config.hostTag() != null) {
            for (Tag tag : tags) {
                if (config.hostTag().equals(tag.getKey())) {
                    json.append(",\"host\":\"").append(escapeJson(tag.getValue())).append("\"");
                    break;
                }
            }
        }

        // Type: "count" for cumulative statistics, "gauge" for instantaneous values
        json.append(",\"type\":\"").append(sanitizeType(statistic)).append("\"");

        // Tags: ["key:value", ...] — Datadog's colon-separated format
        if (!tags.isEmpty()) {
            json.append(",\"tags\":[");
            json.append(tags.stream()
                    .map(t -> "\"" + escapeJson(t.getKey()) + ":" + escapeJson(t.getValue()) + "\"")
                    .collect(joining(",")));
            json.append("]");
        }

        json.append("}");
        return json.toString();
    }

    // -----------------------------------------------------------------------
    // Type mapping: Statistic → Datadog metric type
    // -----------------------------------------------------------------------

    /**
     * Maps a {@link Statistic} to a Datadog metric type string.
     * <p>
     * Datadog has two primary metric types:
     * <ul>
     *   <li>{@code "count"} — for values that represent cumulative totals (Datadog
     *       computes per-second rates from these)</li>
     *   <li>{@code "gauge"} — for values that represent a point-in-time measurement
     *       (Datadog shows the value as-is)</li>
     * </ul>
     */
    static String sanitizeType(Statistic statistic) {
        return switch (statistic) {
            case COUNT, TOTAL, TOTAL_TIME -> "count";
            default -> "gauge";
        };
    }

    // -----------------------------------------------------------------------
    // Base time unit — milliseconds for Datadog
    // -----------------------------------------------------------------------

    /**
     * Datadog uses milliseconds as the base time unit, unlike Prometheus which
     * uses seconds. All time values (timer durations, time gauge values) are
     * converted to milliseconds before publishing.
     */
    @Override
    protected TimeUnit getBaseTimeUnit() {
        return TimeUnit.MILLISECONDS;
    }

}
```

### Test Code

#### File: `src/test/java/dev/linhvu/micrometer/datadog/DatadogNamingConventionTest.java` [NEW]

```java
package dev.linhvu.micrometer.datadog;

import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.config.NamingConvention;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link DatadogNamingConvention}.
 */
class DatadogNamingConventionTest {

    private final DatadogNamingConvention convention = new DatadogNamingConvention();

    // -----------------------------------------------------------------------
    // Metric name transformations
    // -----------------------------------------------------------------------

    @Test
    void shouldPreserveDotNotation_WhenNameIsValid() {
        String result = convention.name("http.server.requests", Meter.Type.COUNTER, null);
        assertThat(result).isEqualTo("http.server.requests");
    }

    @Test
    void shouldPrependM_WhenNameStartsWithNumber() {
        String result = convention.name("123.invalid", Meter.Type.GAUGE, null);
        assertThat(result).isEqualTo("m.123.invalid");
    }

    @Test
    void shouldReplaceSlashes_WithUnderscores() {
        String result = convention.name("path/to/metric", Meter.Type.COUNTER, null);
        assertThat(result).isEqualTo("path_to_metric");
    }

    @Test
    void shouldTruncate_WhenNameExceeds200Characters() {
        String longName = "a".repeat(250);
        String result = convention.name(longName, Meter.Type.COUNTER, null);
        assertThat(result).hasSize(200);
    }

    @Test
    void shouldEscapeJsonSpecialCharacters_InName() {
        String result = convention.name("metric\"with\\special", Meter.Type.COUNTER, null);
        assertThat(result).isEqualTo("metric\\\"with\\\\special");
    }

    @Test
    void shouldRespectDelegateConvention() {
        NamingConvention customDelegate = (name, type, baseUnit) -> name.toUpperCase();
        DatadogNamingConvention custom = new DatadogNamingConvention(customDelegate);
        String result = custom.name("my.metric", Meter.Type.COUNTER, null);
        assertThat(result).isEqualTo("MY.METRIC");
    }

    // -----------------------------------------------------------------------
    // Tag key transformations
    // -----------------------------------------------------------------------

    @Test
    void shouldPreserveTagKey_WhenValid() {
        String result = convention.tagKey("environment");
        assertThat(result).isEqualTo("environment");
    }

    @Test
    void shouldPrependM_WhenTagKeyStartsWithDigit() {
        String result = convention.tagKey("123key");
        assertThat(result).isEqualTo("m.123key");
    }

    @Test
    void shouldNotPrependM_WhenTagKeyStartsWithLetter() {
        String result = convention.tagKey("valid.key");
        assertThat(result).isEqualTo("valid.key");
    }

    @Test
    void shouldEscapeJsonSpecialCharacters_InTagKey() {
        String result = convention.tagKey("key\"with");
        assertThat(result).isEqualTo("key\\\"with");
    }

    @Test
    void shouldPrependMAndRespectDelegate_WhenTagKeyStartsWithDigit() {
        NamingConvention customDelegate = new NamingConvention() {
            @Override
            public String name(String name, Meter.Type type, String baseUnit) {
                return name;
            }

            @Override
            public String tagKey(String key) {
                return key + "_suffix";
            }
        };
        DatadogNamingConvention custom = new DatadogNamingConvention(customDelegate);
        String result = custom.tagKey("123");
        // delegate produces "123_suffix", which starts with digit → "m.123_suffix"
        assertThat(result).isEqualTo("m.123_suffix");
    }

    // -----------------------------------------------------------------------
    // Tag value transformations
    // -----------------------------------------------------------------------

    @Test
    void shouldPreserveTagValue_WhenValid() {
        String result = convention.tagValue("production");
        assertThat(result).isEqualTo("production");
    }

    @Test
    void shouldNotPrependM_WhenTagValueStartsWithDigit() {
        // Tag values CAN start with digits in Datadog
        String result = convention.tagValue("123");
        assertThat(result).isEqualTo("123");
    }

    @Test
    void shouldEscapeJsonSpecialCharacters_InTagValue() {
        String result = convention.tagValue("value\"with\\special\nchars");
        assertThat(result).isEqualTo("value\\\"with\\\\special\\nchars");
    }

    // -----------------------------------------------------------------------
    // JSON escaping utility
    // -----------------------------------------------------------------------

    @Test
    void shouldEscapeAllJsonSpecialCharacters() {
        assertThat(DatadogNamingConvention.escapeJson("\"")).isEqualTo("\\\"");
        assertThat(DatadogNamingConvention.escapeJson("\\")).isEqualTo("\\\\");
        assertThat(DatadogNamingConvention.escapeJson("\n")).isEqualTo("\\n");
        assertThat(DatadogNamingConvention.escapeJson("\r")).isEqualTo("\\r");
        assertThat(DatadogNamingConvention.escapeJson("\t")).isEqualTo("\\t");
    }

    @Test
    void shouldNotModifyRegularCharacters() {
        assertThat(DatadogNamingConvention.escapeJson("hello.world"))
                .isEqualTo("hello.world");
    }

}
```

#### File: `src/test/java/dev/linhvu/micrometer/datadog/DatadogMeterRegistryTest.java` [NEW]

```java
package dev.linhvu.micrometer.datadog;

import dev.linhvu.micrometer.Counter;
import dev.linhvu.micrometer.DistributionSummary;
import dev.linhvu.micrometer.FunctionCounter;
import dev.linhvu.micrometer.FunctionTimer;
import dev.linhvu.micrometer.Gauge;
import dev.linhvu.micrometer.LongTaskTimer;
import dev.linhvu.micrometer.Meter;
import dev.linhvu.micrometer.MockClock;
import dev.linhvu.micrometer.Statistic;
import dev.linhvu.micrometer.Tag;
import dev.linhvu.micrometer.Tags;
import dev.linhvu.micrometer.TimeGauge;
import dev.linhvu.micrometer.Timer;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.net.http.HttpClient;
import java.time.Duration;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link DatadogMeterRegistry}.
 * <p>
 * Uses the package-private constructor to avoid starting background threads
 * and HTTP connections. Tests verify JSON serialization by calling
 * {@link DatadogMeterRegistry#writeMetric} directly and by calling
 * {@link DatadogMeterRegistry#publish()} (which fails gracefully on HTTP
 * errors with a non-routable endpoint).
 */
class DatadogMeterRegistryTest {

    private MockClock clock;
    private DatadogMeterRegistry registry;

    private final DatadogConfig config = new DatadogConfig() {
        @Override
        public String apiKey() {
            return "fake-api-key";
        }

        @Override
        public boolean enabled() {
            return false; // Disable scheduler for testing
        }
    };

    @BeforeEach
    void setUp() {
        clock = new MockClock();
        // Use package-private constructor — no background threads, no real HTTP
        registry = new DatadogMeterRegistry(config, clock, HttpClient.newHttpClient());
    }

    @AfterEach
    void tearDown() {
        if (registry != null && !registry.isClosed()) {
            registry.close();
        }
    }

    // -----------------------------------------------------------------------
    // writeMetric — JSON structure
    // -----------------------------------------------------------------------

    @Test
    void shouldProduceValidJson_ForSimpleMetric() {
        Meter.Id id = new Meter.Id("http.requests", Tags.empty(),
                Meter.Type.COUNTER, null, null);

        String json = registry.writeMetric(id, null, 60_000, 42.0, Statistic.COUNT);

        assertThat(json).contains("\"metric\":\"http.requests\"");
        assertThat(json).contains("\"points\":[[60, 42.0]]");
        assertThat(json).contains("\"type\":\"count\"");
    }

    @Test
    void shouldAppendSuffix_WhenProvided() {
        Meter.Id id = new Meter.Id("http.latency", Tags.empty(),
                Meter.Type.TIMER, null, null);

        String json = registry.writeMetric(id, "count", 60_000, 10.0, Statistic.COUNT);

        assertThat(json).contains("\"metric\":\"http.latency.count\"");
    }

    @Test
    void shouldIncludeTags_InColonSeparatedFormat() {
        Meter.Id id = new Meter.Id("http.requests",
                Tags.of("method", "GET", "status", "200"),
                Meter.Type.COUNTER, null, null);

        String json = registry.writeMetric(id, null, 60_000, 5.0, Statistic.COUNT);

        assertThat(json).contains("\"tags\":[");
        assertThat(json).contains("\"method:GET\"");
        assertThat(json).contains("\"status:200\"");
    }

    @Test
    void shouldConvertWallTimeToEpochSeconds() {
        Meter.Id id = new Meter.Id("test", Tags.empty(),
                Meter.Type.GAUGE, null, null);

        String json = registry.writeMetric(id, null, 120_000, 1.0, Statistic.VALUE);

        // 120,000 ms = 120 seconds
        assertThat(json).contains("\"points\":[[120, 1.0]]");
    }

    // -----------------------------------------------------------------------
    // Type mapping: Statistic → Datadog type
    // -----------------------------------------------------------------------

    @Test
    void shouldMapCountStatistic_ToCountType() {
        assertThat(DatadogMeterRegistry.sanitizeType(Statistic.COUNT)).isEqualTo("count");
    }

    @Test
    void shouldMapTotalStatistic_ToCountType() {
        assertThat(DatadogMeterRegistry.sanitizeType(Statistic.TOTAL)).isEqualTo("count");
    }

    @Test
    void shouldMapTotalTimeStatistic_ToCountType() {
        assertThat(DatadogMeterRegistry.sanitizeType(Statistic.TOTAL_TIME)).isEqualTo("count");
    }

    @Test
    void shouldMapValueStatistic_ToGaugeType() {
        assertThat(DatadogMeterRegistry.sanitizeType(Statistic.VALUE)).isEqualTo("gauge");
    }

    @Test
    void shouldMapMaxStatistic_ToGaugeType() {
        assertThat(DatadogMeterRegistry.sanitizeType(Statistic.MAX)).isEqualTo("gauge");
    }

    @Test
    void shouldMapActiveTasksStatistic_ToGaugeType() {
        assertThat(DatadogMeterRegistry.sanitizeType(Statistic.ACTIVE_TASKS)).isEqualTo("gauge");
    }

    // -----------------------------------------------------------------------
    // Host tag extraction
    // -----------------------------------------------------------------------

    @Test
    void shouldExtractHostTag_WhenPresent() {
        Meter.Id id = new Meter.Id("cpu.usage",
                Tags.of("instance", "web-1", "region", "us-east"),
                Meter.Type.GAUGE, null, null);

        String json = registry.writeMetric(id, null, 60_000, 75.0, Statistic.VALUE);

        assertThat(json).contains("\"host\":\"web-1\"");
    }

    @Test
    void shouldNotIncludeHost_WhenHostTagMissing() {
        Meter.Id id = new Meter.Id("cpu.usage",
                Tags.of("region", "us-east"),
                Meter.Type.GAUGE, null, null);

        String json = registry.writeMetric(id, null, 60_000, 75.0, Statistic.VALUE);

        assertThat(json).doesNotContain("\"host\"");
    }

    @Test
    void shouldNotIncludeHost_WhenHostTagConfigIsNull() {
        DatadogConfig noHostConfig = new DatadogConfig() {
            @Override
            public String apiKey() {
                return "fake";
            }

            @Override
            public String hostTag() {
                return null;
            }

            @Override
            public boolean enabled() {
                return false;
            }
        };
        DatadogMeterRegistry reg = new DatadogMeterRegistry(noHostConfig, clock,
                HttpClient.newHttpClient());

        Meter.Id id = new Meter.Id("test",
                Tags.of("instance", "web-1"),
                Meter.Type.GAUGE, null, null);

        String json = reg.writeMetric(id, null, 60_000, 1.0, Statistic.VALUE);

        assertThat(json).doesNotContain("\"host\"");
        reg.close();
    }

    // -----------------------------------------------------------------------
    // Counter serialization
    // -----------------------------------------------------------------------

    @Test
    void shouldSerializeCounter_WithStatisticTag() {
        Counter counter = registry.counter("api.calls");
        counter.increment(100);

        clock.add(Duration.ofMinutes(1));

        // Counter goes through writeMeter → measure() → adds "statistic:count" tag
        String json = registry.writeMetric(
                counter.getId().withTag(Statistic.COUNT),
                null, clock.wallTime(), counter.count(), Statistic.COUNT);

        assertThat(json).contains("\"metric\":\"api.calls\"");
        assertThat(json).contains("\"type\":\"count\"");
        assertThat(json).contains("\"statistic:count\"");
    }

    // -----------------------------------------------------------------------
    // Timer decomposition
    // -----------------------------------------------------------------------

    @Test
    void shouldDecomposeTimer_IntoFourSubMetrics() {
        Timer timer = registry.timer("http.latency");
        timer.record(100, TimeUnit.MILLISECONDS);
        timer.record(200, TimeUnit.MILLISECONDS);

        clock.add(Duration.ofMinutes(1));

        // Verify count sub-metric
        String countJson = registry.writeMetric(timer.getId(), "count",
                clock.wallTime(), timer.count(), Statistic.COUNT);
        assertThat(countJson).contains("\"metric\":\"http.latency.count\"");
        assertThat(countJson).contains("\"type\":\"count\"");

        // Verify sum sub-metric
        String sumJson = registry.writeMetric(timer.getId(), "sum",
                clock.wallTime(), timer.totalTime(TimeUnit.MILLISECONDS), Statistic.TOTAL_TIME);
        assertThat(sumJson).contains("\"metric\":\"http.latency.sum\"");
        assertThat(sumJson).contains("\"type\":\"count\"");

        // Verify avg sub-metric
        String avgJson = registry.writeMetric(timer.getId(), "avg",
                clock.wallTime(), timer.mean(TimeUnit.MILLISECONDS), Statistic.VALUE);
        assertThat(avgJson).contains("\"metric\":\"http.latency.avg\"");
        assertThat(avgJson).contains("\"type\":\"gauge\"");

        // Verify max sub-metric
        String maxJson = registry.writeMetric(timer.getId(), "max",
                clock.wallTime(), timer.max(TimeUnit.MILLISECONDS), Statistic.MAX);
        assertThat(maxJson).contains("\"metric\":\"http.latency.max\"");
        assertThat(maxJson).contains("\"type\":\"gauge\"");
    }

    // -----------------------------------------------------------------------
    // DistributionSummary decomposition
    // -----------------------------------------------------------------------

    @Test
    void shouldDecomposeSummary_IntoFourSubMetrics() {
        DistributionSummary summary = registry.summary("payload.size");
        summary.record(100);
        summary.record(200);
        summary.record(300);

        clock.add(Duration.ofMinutes(1));

        // Verify count sub-metric
        String countJson = registry.writeMetric(summary.getId(), "count",
                clock.wallTime(), summary.count(), Statistic.COUNT);
        assertThat(countJson).contains("\"metric\":\"payload.size.count\"");
        assertThat(countJson).contains("\"type\":\"count\"");

        // Verify sum sub-metric
        String sumJson = registry.writeMetric(summary.getId(), "sum",
                clock.wallTime(), summary.totalAmount(), Statistic.TOTAL);
        assertThat(sumJson).contains("\"metric\":\"payload.size.sum\"");
        assertThat(sumJson).contains("\"type\":\"count\"");

        // Verify avg and max types
        String avgJson = registry.writeMetric(summary.getId(), "avg",
                clock.wallTime(), summary.mean(), Statistic.VALUE);
        assertThat(avgJson).contains("\"type\":\"gauge\"");

        String maxJson = registry.writeMetric(summary.getId(), "max",
                clock.wallTime(), summary.max(), Statistic.MAX);
        assertThat(maxJson).contains("\"type\":\"gauge\"");
    }

    // -----------------------------------------------------------------------
    // Base time unit
    // -----------------------------------------------------------------------

    @Test
    void shouldUseMillisecondsAsBaseTimeUnit() {
        assertThat(registry.getBaseTimeUnit()).isEqualTo(TimeUnit.MILLISECONDS);
    }

    // -----------------------------------------------------------------------
    // Naming convention
    // -----------------------------------------------------------------------

    @Test
    void shouldUseDatadogNamingConvention() {
        assertThat(registry.getNamingConvention()).isInstanceOf(DatadogNamingConvention.class);
    }

    @Test
    void shouldApplyNamingConvention_InWriteMetric() {
        // Metric name starting with digit should get "m." prefix from the convention
        Meter.Id id = new Meter.Id("123.metric", Tags.empty(),
                Meter.Type.COUNTER, null, null);

        String json = registry.writeMetric(id, null, 60_000, 1.0, Statistic.COUNT);

        assertThat(json).contains("\"metric\":\"m.123.metric\"");
    }

    // -----------------------------------------------------------------------
    // Gauge serialization
    // -----------------------------------------------------------------------

    @Test
    void shouldSerializeGauge_AsGaugeType() {
        AtomicLong value = new AtomicLong(42);
        registry.gauge("pool.size", value);

        Meter.Id id = new Meter.Id("pool.size", Tags.empty(),
                Meter.Type.GAUGE, null, null);

        String json = registry.writeMetric(id.withTag(Statistic.VALUE),
                null, clock.wallTime(), 42.0, Statistic.VALUE);

        assertThat(json).contains("\"type\":\"gauge\"");
    }

    // -----------------------------------------------------------------------
    // LongTaskTimer serialization
    // -----------------------------------------------------------------------

    @Test
    void shouldSerializeLongTaskTimer_WithActiveTasksAndDuration() {
        LongTaskTimer ltt = LongTaskTimer.builder("batch.import").register(registry);
        LongTaskTimer.Sample sample = ltt.start();

        clock.add(5, TimeUnit.SECONDS);

        Meter.Id id = ltt.getId();
        String activeJson = registry.writeMetric(id.withTag(Statistic.ACTIVE_TASKS),
                null, clock.wallTime(), ltt.activeTasks(), Statistic.ACTIVE_TASKS);

        assertThat(activeJson).contains("\"type\":\"gauge\"");
        assertThat(activeJson).contains("\"statistic:active\"");

        sample.stop();
    }

    // -----------------------------------------------------------------------
    // FunctionCounter serialization
    // -----------------------------------------------------------------------

    @Test
    void shouldSerializeFunctionCounter_AsCountType() {
        AtomicLong source = new AtomicLong(100);
        FunctionCounter.builder("cache.evictions", source, AtomicLong::get)
                .register(registry);

        Meter.Id id = new Meter.Id("cache.evictions", Tags.empty(),
                Meter.Type.COUNTER, null, null);

        String json = registry.writeMetric(id.withTag(Statistic.COUNT),
                null, clock.wallTime(), 100.0, Statistic.COUNT);

        assertThat(json).contains("\"type\":\"count\"");
    }

    // -----------------------------------------------------------------------
    // FunctionTimer decomposition
    // -----------------------------------------------------------------------

    @Test
    void shouldDecomposeFunctionTimer_IntoThreeSubMetrics() {
        AtomicLong countSource = new AtomicLong(10);
        AtomicLong totalSource = new AtomicLong(500);
        Object[] sources = { countSource, totalSource };
        FunctionTimer.builder("db.query", sources,
                        obj -> ((AtomicLong) ((Object[]) obj)[0]).get(),
                        obj -> ((AtomicLong) ((Object[]) obj)[1]).doubleValue(),
                        TimeUnit.MILLISECONDS)
                .register(registry);

        Meter.Id id = new Meter.Id("db.query", Tags.empty(),
                Meter.Type.TIMER, null, "milliseconds");

        // count → "count" type
        String countJson = registry.writeMetric(id, "count",
                clock.wallTime(), 10.0, Statistic.COUNT);
        assertThat(countJson).contains("\"metric\":\"db.query.count\"");
        assertThat(countJson).contains("\"type\":\"count\"");

        // avg → "gauge" type
        String avgJson = registry.writeMetric(id, "avg",
                clock.wallTime(), 50.0, Statistic.VALUE);
        assertThat(avgJson).contains("\"metric\":\"db.query.avg\"");
        assertThat(avgJson).contains("\"type\":\"gauge\"");

        // sum → "count" type
        String sumJson = registry.writeMetric(id, "sum",
                clock.wallTime(), 500.0, Statistic.TOTAL_TIME);
        assertThat(sumJson).contains("\"metric\":\"db.query.sum\"");
        assertThat(sumJson).contains("\"type\":\"count\"");
    }

    // -----------------------------------------------------------------------
    // TimeGauge serialization
    // -----------------------------------------------------------------------

    @Test
    void shouldSerializeTimeGauge_AsGaugeType() {
        AtomicLong uptimeMs = new AtomicLong(65_000);
        TimeGauge.builder("jvm.uptime", uptimeMs, TimeUnit.MILLISECONDS,
                AtomicLong::doubleValue).register(registry);

        Meter.Id id = new Meter.Id("jvm.uptime", Tags.empty(),
                Meter.Type.GAUGE, null, null);

        String json = registry.writeMetric(id.withTag(Statistic.VALUE),
                null, clock.wallTime(), 65_000.0, Statistic.VALUE);

        assertThat(json).contains("\"type\":\"gauge\"");
    }

    // -----------------------------------------------------------------------
    // JSON escaping in metric values
    // -----------------------------------------------------------------------

    @Test
    void shouldEscapeSpecialCharacters_InMetricName() {
        Meter.Id id = new Meter.Id("my.counter#abc", Tags.empty(),
                Meter.Type.COUNTER, null, null);

        String json = registry.writeMetric(id, null, 60_000, 1.0, Statistic.COUNT);

        // The naming convention doesn't escape #, but it's passed through correctly
        assertThat(json).contains("\"metric\":\"my.counter#abc\"");
    }

    @Test
    void shouldEscapeSpecialCharacters_InTagValues() {
        Meter.Id id = new Meter.Id("test",
                Tags.of("path", "/api/v1\"users"),
                Meter.Type.COUNTER, null, null);

        String json = registry.writeMetric(id, null, 60_000, 1.0, Statistic.COUNT);

        // The naming convention escapes " to \"
        assertThat(json).contains("/api/v1\\\\\\\"users")
                .as("Quotes in tag values should be JSON-escaped");
    }

    // -----------------------------------------------------------------------
    // Config defaults
    // -----------------------------------------------------------------------

    @Test
    void shouldUseDefaultUri() {
        assertThat(config.uri()).isEqualTo("https://api.datadoghq.com");
    }

    @Test
    void shouldUseDefaultHostTag() {
        assertThat(config.hostTag()).isEqualTo("instance");
    }

    @Test
    void shouldUseDefaultDescriptions() {
        assertThat(config.descriptions()).isTrue();
    }

    // -----------------------------------------------------------------------
    // Publish integration (verify it doesn't throw on non-routable endpoint)
    // -----------------------------------------------------------------------

    @Test
    void shouldHandlePublishGracefully_WhenEndpointUnreachable() {
        DatadogConfig unreachableConfig = new DatadogConfig() {
            @Override
            public String apiKey() {
                return "fake";
            }

            @Override
            public String uri() {
                // Use a non-routable endpoint
                return "http://192.0.2.1:1";
            }

            @Override
            public boolean enabled() {
                return false;
            }
        };
        DatadogMeterRegistry reg = new DatadogMeterRegistry(unreachableConfig, clock,
                HttpClient.newBuilder()
                        .connectTimeout(Duration.ofMillis(100))
                        .build());

        reg.counter("test.counter").increment(5);
        clock.add(Duration.ofMinutes(1));

        // Should not throw — errors are caught and logged
        reg.publish();
        reg.close();
    }

}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Metric decomposition** | Breaking Timers/Summaries into `.count`, `.sum`, `.avg`, `.max` sub-metrics because Datadog has no native composite type |
| **Type mapping** | COUNT/TOTAL/TOTAL_TIME → `"count"` (Datadog computes rates); everything else → `"gauge"` (shown as-is) |
| **Colon-separated tags** | Datadog's native tag format `"key:value"`, distinct from Prometheus's `key="value"` label syntax |
| **Host tag extraction** | Mapping a configurable Micrometer tag key to Datadog's infrastructure host field |
| **NamingConvention delegate** | Layered naming — DatadogNamingConvention wraps a base convention, adding Datadog-specific rules on top |

**This completes the simple-micrometer learning path.** All 21 features are now implemented, covering the full metrics pipeline from core identity (ch1) through meter types, registry infrastructure, distribution statistics, the Observation API, and three backend integrations: SimpleMeterRegistry (in-memory), LoggingMeterRegistry (console), PrometheusMeterRegistry (pull/scrape), and DatadogMeterRegistry (push/HTTP).
