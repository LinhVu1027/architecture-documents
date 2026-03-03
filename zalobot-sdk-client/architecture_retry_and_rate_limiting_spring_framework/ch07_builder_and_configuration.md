# Chapter 7: Builder & Configuration — Exposing Retry and Throttle Settings

## What You'll Learn
- How to extend `ZaloBotClient.Builder` with retry and throttle configuration
- The principle of "sensible defaults with full override" for SDK design
- How `DefaultZaloBotClientBuilder` constructs `RetryTemplate` and `ConcurrencyThrottle`
- Spring Boot auto-configuration integration for retry properties
- How `mutate()` preserves retry/throttle settings for derived clients

---

## 7.1 The Problem: Configuration Must Be Easy

We've built powerful retry and throttle infrastructure, but users shouldn't need to understand `RetryPolicy.Builder`, `ExponentialBackOff`, or `ConcurrencyThrottle` to get started. The SDK should:

1. **Work out of the box** — sensible defaults for retry and throttle
2. **Allow simple customization** — `retryMaxAttempts(5)` without deep API knowledge
3. **Allow full control** — `retryPolicy(customPolicy)` for advanced users
4. **Be discoverable** — all options visible in the Builder's fluent API

---

## 7.2 Extended ZaloBotClient.Builder Interface

```java
// In ZaloBotClient interface, extend the existing Builder:

public interface ZaloBotClient {

    interface Builder {
        // ─── Existing ───
        Builder zaloBotUrl(ZaloBotUrl url);
        Builder botToken(String botToken);
        Builder requestFactory(ClientHttpRequestFactory requestFactory);
        Builder jsonMapper(JsonMapper jsonMapper);

        // ─── NEW: Retry Configuration ───

        /**
         * Set a custom retry policy. Overrides all other retry settings.
         */
        Builder retryPolicy(RetryPolicy retryPolicy);

        /**
         * Maximum number of retry attempts (default: 3).
         * Set to 0 to disable retry.
         */
        Builder retryMaxAttempts(int maxAttempts);

        /**
         * Base delay between retries (default: 1 second).
         */
        Builder retryDelay(Duration delay);

        /**
         * Backoff multiplier (default: 2.0 for exponential).
         * Set to 1.0 for fixed delay.
         */
        Builder retryMultiplier(double multiplier);

        /**
         * Maximum delay between retries (default: 30 seconds).
         */
        Builder retryMaxDelay(Duration maxDelay);

        /**
         * Overall timeout for all retry attempts (default: no timeout).
         */
        Builder retryTimeout(Duration timeout);

        /**
         * Register a retry listener for observability.
         */
        Builder retryListener(RetryListener listener);

        // ─── NEW: Throttle Configuration ───

        /**
         * Maximum concurrent API calls (default: unbounded).
         * Set to -1 for no limit, 0 to block all, or a positive number.
         */
        Builder concurrencyLimit(int limit);

        /**
         * Policy when concurrency limit is reached (default: BLOCK).
         */
        Builder throttlePolicy(ThrottlePolicy policy);

        ZaloBotClient build();
    }
}
```

---

## 7.3 DefaultZaloBotClientBuilder Implementation

```java
package dev.linhvu.zalobot.client;

import java.time.Duration;

import dev.linhvu.zalobot.client.exception.*;
import dev.linhvu.zalobot.client.retry.*;
import dev.linhvu.zalobot.client.throttle.*;
import dev.linhvu.zalobot.client.util.ClassUtils;

final class DefaultZaloBotClientBuilder implements ZaloBotClient.Builder {

    // ─── Existing fields ───
    private ZaloBotUrl url = ZaloBotUrl.DEFAULT;
    private String botToken;
    private ClientHttpRequestFactory requestFactory;
    private JsonMapper jsonMapper;

    // ─── NEW: Retry fields ───
    private RetryPolicy retryPolicy;           // custom policy (overrides everything)
    private Integer retryMaxAttempts;           // null = use default (3)
    private Duration retryDelay;               // null = use default (1s)
    private Double retryMultiplier;            // null = use default (2.0)
    private Duration retryMaxDelay;            // null = use default (30s)
    private Duration retryTimeout;             // null = no timeout
    private Duration retryJitter;              // null = use default (200ms)
    private RetryListener retryListener;       // null = no-op

    // ─── NEW: Throttle fields ───
    private int concurrencyLimit = ConcurrencyThrottle.UNBOUNDED;
    private ThrottlePolicy throttlePolicy = ThrottlePolicy.BLOCK;

    DefaultZaloBotClientBuilder() {}

    // Copy constructor for mutate()
    DefaultZaloBotClientBuilder(DefaultZaloBotClientBuilder other) {
        this.url = other.url;
        this.botToken = other.botToken;
        this.requestFactory = other.requestFactory;
        this.jsonMapper = other.jsonMapper;
        this.retryPolicy = other.retryPolicy;
        this.retryMaxAttempts = other.retryMaxAttempts;
        this.retryDelay = other.retryDelay;
        this.retryMultiplier = other.retryMultiplier;
        this.retryMaxDelay = other.retryMaxDelay;
        this.retryTimeout = other.retryTimeout;
        this.retryJitter = other.retryJitter;
        this.retryListener = other.retryListener;
        this.concurrencyLimit = other.concurrencyLimit;
        this.throttlePolicy = other.throttlePolicy;
    }

    // ─── Existing setters ───
    @Override public Builder zaloBotUrl(ZaloBotUrl url) { this.url = url; return this; }
    @Override public Builder botToken(String token) { this.botToken = token; return this; }
    @Override public Builder requestFactory(ClientHttpRequestFactory f) { this.requestFactory = f; return this; }
    @Override public Builder jsonMapper(JsonMapper m) { this.jsonMapper = m; return this; }

    // ─── NEW: Retry setters ───
    @Override public Builder retryPolicy(RetryPolicy p) { this.retryPolicy = p; return this; }
    @Override public Builder retryMaxAttempts(int n) { this.retryMaxAttempts = n; return this; }
    @Override public Builder retryDelay(Duration d) { this.retryDelay = d; return this; }
    @Override public Builder retryMultiplier(double m) { this.retryMultiplier = m; return this; }
    @Override public Builder retryMaxDelay(Duration d) { this.retryMaxDelay = d; return this; }
    @Override public Builder retryTimeout(Duration d) { this.retryTimeout = d; return this; }
    @Override public Builder retryListener(RetryListener l) { this.retryListener = l; return this; }

    // ─── NEW: Throttle setters ───
    @Override public Builder concurrencyLimit(int l) { this.concurrencyLimit = l; return this; }
    @Override public Builder throttlePolicy(ThrottlePolicy p) { this.throttlePolicy = p; return this; }

    @Override
    public ZaloBotClient build() {
        // ... existing validation and factory selection ...
        if (this.botToken == null) {
            throw new IllegalArgumentException("'botToken' is required");
        }

        ClientHttpRequestFactory factory = resolveRequestFactory();
        JsonMapper mapper = this.jsonMapper != null ? this.jsonMapper : JsonMapper.builder().build();

        // ═══════════════════════════════════════
        // Build RetryTemplate
        // ═══════════════════════════════════════
        RetryTemplate retryTemplate = buildRetryTemplate();

        // ═══════════════════════════════════════
        // Build ConcurrencyThrottle
        // ═══════════════════════════════════════
        ConcurrencyThrottle throttle = new ConcurrencyThrottle(this.concurrencyLimit);
        throttle.setPolicy(this.throttlePolicy);

        return new DefaultZaloBotClient(
            this.url, this.botToken, factory, mapper, this, retryTemplate, throttle);
    }

    private RetryTemplate buildRetryTemplate() {
        RetryPolicy policy;

        if (this.retryPolicy != null) {
            // User provided a complete custom policy
            policy = this.retryPolicy;
        }
        else {
            // Build from individual settings with sensible defaults
            RetryPolicy.Builder builder = RetryPolicy.builder()
                .includes(ZaloBotClientException.class, ZaloBotApiException.class)
                .excludes(ZaloBotAuthenticationException.class,
                          ZaloBotSerializationException.class);

            builder.maxRetries(this.retryMaxAttempts != null ? this.retryMaxAttempts : 3);
            builder.delay(this.retryDelay != null ? this.retryDelay : Duration.ofSeconds(1));
            builder.multiplier(this.retryMultiplier != null ? this.retryMultiplier : 2.0);
            builder.maxDelay(this.retryMaxDelay != null ? this.retryMaxDelay : Duration.ofSeconds(30));
            builder.jitter(this.retryJitter != null ? this.retryJitter : Duration.ofMillis(200));

            if (this.retryTimeout != null) {
                builder.timeout(this.retryTimeout);
            }

            policy = builder.build();
        }

        RetryTemplate template = new RetryTemplate(policy);
        if (this.retryListener != null) {
            template.setRetryListener(this.retryListener);
        }
        return template;
    }

    private ClientHttpRequestFactory resolveRequestFactory() {
        if (this.requestFactory != null) {
            return this.requestFactory;
        }
        // Auto-detect: OkHttp3 → JDK HttpClient
        if (ClassUtils.isPresent("okhttp3.OkHttpClient", getClass().getClassLoader())) {
            return new OkClientHttpRequestFactory();
        }
        return new JdkClientHttpRequestFactory();
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - The builder follows a **two-tier API** pattern: simple settings (`retryMaxAttempts`, `retryDelay`) for common cases, and a full `retryPolicy()` override for advanced cases. The simple settings are always translated into a `RetryPolicy` internally — there's only one code path in the client.
> - The exception filter defaults (`includes(ZaloBotClientException, ZaloBotApiException).excludes(ZaloBotAuthenticationException, ZaloBotSerializationException)`) are **hardcoded in the builder**, not in `RetryPolicy.withDefaults()`. This is deliberate: `RetryPolicy.withDefaults()` is a general-purpose default (retry everything), while the builder's default is SDK-specific (retry only network and API errors).
> - The `mutate()` pattern (copy constructor) preserves ALL settings, including retry and throttle config. This means `client.mutate().botToken(newToken).build()` creates a client with the same retry settings but a different token — important for multi-bot scenarios.
> ─────────────────────────────────────────────────────

---

## 7.4 Usage Examples

### Basic (all defaults)
```java
ZaloBotClient client = ZaloBotClient.builder()
    .botToken("my-token")
    .build();
// → 3 retries, exponential backoff 1s→2s→4s, 200ms jitter, exclude auth errors
```

### Custom retry count
```java
ZaloBotClient client = ZaloBotClient.builder()
    .botToken("my-token")
    .retryMaxAttempts(5)
    .build();
```

### Disable retry
```java
ZaloBotClient client = ZaloBotClient.builder()
    .botToken("my-token")
    .retryMaxAttempts(0)
    .build();
```

### With concurrency limit
```java
ZaloBotClient client = ZaloBotClient.builder()
    .botToken("my-token")
    .concurrencyLimit(10)           // max 10 concurrent API calls
    .throttlePolicy(ThrottlePolicy.BLOCK)
    .build();
```

### Full custom policy
```java
RetryPolicy aggressive = RetryPolicy.builder()
    .includes(ZaloBotClientException.class)
    .delay(Duration.ofMillis(500))
    .multiplier(3.0)
    .maxDelay(Duration.ofMinutes(1))
    .maxRetries(10)
    .jitter(Duration.ofMillis(500))
    .timeout(Duration.ofMinutes(5))
    .build();

ZaloBotClient client = ZaloBotClient.builder()
    .botToken("my-token")
    .retryPolicy(aggressive)
    .retryListener(new RetryListener() {
        @Override
        public void onRetryFailure(RetryPolicy p, Retryable<?> r, Throwable t) {
            metrics.increment("zalobot.retry.failure");
        }
    })
    .concurrencyLimit(5)
    .build();
```

---

## 7.5 Spring Boot Auto-Configuration

For Spring Boot users, expose retry settings as properties:

```java
// In ZaloBotListenerAutoConfiguration or a new ZaloBotRetryAutoConfiguration:

@Bean
@ConditionalOnMissingBean
public ZaloBotClient zaloBotClient(ZaloBotClient.Builder builder) {
    return builder.build();
}

@Bean
@Scope("prototype")
public ZaloBotClient.Builder zaloBotClientBuilder(
        @Value("${zalobot.bot-token}") String botToken,
        @Value("${zalobot.retry.max-attempts:3}") int maxAttempts,
        @Value("${zalobot.retry.delay:1000}") long delayMs,
        @Value("${zalobot.retry.multiplier:2.0}") double multiplier,
        @Value("${zalobot.retry.max-delay:30000}") long maxDelayMs,
        @Value("${zalobot.retry.timeout:0}") long timeoutMs,
        @Value("${zalobot.concurrency.limit:-1}") int concurrencyLimit) {

    ZaloBotClient.Builder builder = ZaloBotClient.builder()
        .botToken(botToken)
        .retryMaxAttempts(maxAttempts)
        .retryDelay(Duration.ofMillis(delayMs))
        .retryMultiplier(multiplier)
        .retryMaxDelay(Duration.ofMillis(maxDelayMs))
        .concurrencyLimit(concurrencyLimit);

    if (timeoutMs > 0) {
        builder.retryTimeout(Duration.ofMillis(timeoutMs));
    }
    return builder;
}
```

**application.yml**:
```yaml
zalobot:
  bot-token: ${ZALO_BOT_TOKEN}
  retry:
    max-attempts: 5
    delay: 2000
    multiplier: 2.0
    max-delay: 60000
    timeout: 120000
  concurrency:
    limit: 10
```

---

## 7.6 Connection to Real Spring Framework

| Simplified | Real Spring Source | Key Differences |
|-----------|-------------------|-----------------|
| Builder retry methods | Not in Spring (Spring doesn't build retry into clients) | Our innovation: SDK-specific convenience |
| `buildRetryTemplate()` | Similar to `RetryAnnotationBeanPostProcessor.buildRetryPolicy()` | We build from Builder fields; Spring builds from annotation attributes |
| Default exception filter | Not in Spring's `RetryPolicy.withDefaults()` | Our default is SDK-specific; Spring's is generic |
| Spring Boot properties | `@Retryable` annotation attributes | We use `@Value` + properties; Spring uses annotation + SpEL |
| `mutate()` copy constructor | Common pattern in Spring's builders (e.g., `WebClient.mutate()`) | Same concept: derive a modified client from existing |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Two-tier API** | Simple settings for common cases, full policy override for advanced |
| **Sensible defaults** | 3 retries, 1s exponential backoff, auth excluded — works out of the box |
| **retryMaxAttempts(0)** | Disables retry entirely |
| **retryPolicy()** | Full override — ignores all other retry settings |
| **concurrencyLimit(-1)** | Unbounded (no throttling) — the default |
| **mutate()** | Copy all settings (including retry/throttle) to derive a new client |
| **Spring Boot properties** | Externalized configuration for production tuning |

**Next: Chapter 8** — We'll migrate `zalobot-listener`'s existing `ExponentialBackOff` to use the new shared `BackOff` abstraction from `zalobot-client`.
