# Chapter 5: Link — Associate Related Spans Across Traces

> **What you'll build**: The `Link` class and `Span.Builder.addLink()` method — the mechanism
> for associating spans that belong to different traces, commonly used in batch processing and
> fan-in/fan-out patterns.

---

## 1. The API Contract

After this chapter, a client can write:

```java
// Link a processing span to an incoming message's trace context
Span processingSpan = tracer.spanBuilder()
    .name("process-message")
    .addLink(new Link(messageContext))
    .start();

// Link with metadata about the association
Span batchSpan = tracer.spanBuilder()
    .name("process-batch")
    .addLink(new Link(messageContext, Map.of("batch.index", 3)))
    .start();

// Link directly from a Span (extracts its context automatically)
Link link = new Link(producerSpan);
```

**Without links, traces are trees**: Chapters 1–4 built a model where each span has exactly
one parent, forming a tree rooted at the entry span. But some operations — especially batch
processing — don't fit this model. A batch consumer processes messages from *multiple*
independent traces. Links let the batch processing span say "I'm related to these other
traces" without claiming a parent-child relationship.

### The Single Type

```
Link                                    ← holds TraceContext + optional tags
└── used by: Span.Builder.addLink()     ← attaches links before starting a span
```

**Why a concrete class instead of an interface?** `Link` is pure data — a `TraceContext`
reference and a `Map<String, Object>` of tags. There's no behavioral variation to model,
no bridge-specific logic, and no reason for multiple implementations. A final concrete class
with value semantics (`equals`/`hashCode`) is the right choice here.

### The Tag Type: Map\<String, Object\> vs Map\<String, String\>

Link tags use `Map<String, Object>` — broader than span tags (`Map<String, String>`). This
follows OpenTelemetry's attribute model where link attributes can be strings, numbers, or
booleans. The broader type allows richer metadata on the association:

```java
new Link(context, Map.of(
    "batch.index", 3,           // Integer — not possible with String tags
    "source", "queue-orders",   // String
    "retry", true               // Boolean
));
```

---

## 2. Client Usage & Tests

### Test: Create a link from TraceContext

The most common use case — link to a span in another trace via its context:

```java
@Test
void shouldCreateLink_fromTraceContext() {
    TraceContext context = buildContext("trace-abc", "span-def", null, true);

    Link link = new Link(context);

    assertThat(link.getTraceContext()).isSameAs(context);
    assertThat(link.getTags()).isEmpty();
}
```

### Test: Create a link with tags

Tags provide metadata about *why* the association exists:

```java
@Test
void shouldCreateLink_fromTraceContextAndTags() {
    TraceContext context = buildContext("trace-abc", "span-def", null, true);
    Map<String, Object> tags = Map.of("batch.index", 3, "source", "queue-a");

    Link link = new Link(context, tags);

    assertThat(link.getTraceContext()).isSameAs(context);
    assertThat(link.getTags()).containsEntry("batch.index", 3);
    assertThat(link.getTags()).containsEntry("source", "queue-a");
}
```

### Test: Create a link from a Span

Convenience constructor — extracts `span.context()` automatically:

```java
@Test
void shouldCreateLink_fromSpan() {
    TraceContext context = buildContext("trace-abc", "span-def", null, true);
    Span span = new StubSpan(context);

    Link link = new Link(span);

    assertThat(link.getTraceContext()).isSameAs(context);
    assertThat(link.getTags()).isEmpty();
}
```

**Why accept `Span` in addition to `TraceContext`?** In practice, clients often have a
`Span` reference (the producer span they received), not a `TraceContext`. Having both
constructor overloads avoids the ceremony of `new Link(span.context())` — a small
ergonomic win that appears throughout the real framework.

### Test: NOOP link

```java
@Test
void noopLink_shouldHaveNoopContext() {
    assertThat(Link.NOOP.getTraceContext()).isSameAs(TraceContext.NOOP);
    assertThat(Link.NOOP.getTags()).isEmpty();
}
```

**The NOOP chain continues**: `Link.NOOP` points to `TraceContext.NOOP`, maintaining the
project's pattern where every type has a safe no-op constant that cascades properly.

### Test: Equality is based on context and tags

```java
@Test
void shouldBeEqual_whenSameContextAndTags() {
    TraceContext ctx = buildContext("trace-1", "span-1", null, true);
    Link link1 = new Link(ctx, Map.of("key", "val"));
    Link link2 = new Link(ctx, Map.of("key", "val"));

    assertThat(link1).isEqualTo(link2);
    assertThat(link1.hashCode()).isEqualTo(link2.hashCode());
}

@Test
void shouldNotBeEqual_whenDifferentContext() {
    TraceContext ctx1 = buildContext("trace-1", "span-1", null, true);
    TraceContext ctx2 = buildContext("trace-2", "span-2", null, true);

    assertThat(new Link(ctx1)).isNotEqualTo(new Link(ctx2));
}

@Test
void shouldNotBeEqual_whenDifferentTags() {
    TraceContext ctx = buildContext("trace-1", "span-1", null, true);

    assertThat(new Link(ctx, Map.of("a", "1")))
        .isNotEqualTo(new Link(ctx, Map.of("a", "2")));
}
```

**Why `equals`/`hashCode`?** Links are stored in collections (lists on spans, sets in
exporters). Proper value semantics enable deduplication and assertion-based testing. Two
links are equal when they point to the same trace context with the same tags.

### Test: Span.Builder.addLink — NOOP default behavior

```java
@Test
void noopBuilder_addLink_shouldReturnThis() {
    Span.Builder builder = Span.Builder.NOOP;
    Link link = new Link(TraceContext.NOOP);

    Span.Builder result = builder.addLink(link);
    assertThat(result).isSameAs(builder);
}

@Test
void noopBuilder_addLink_shouldStillStartNoopSpan() {
    Span span = Span.Builder.NOOP
            .name("batch-process")
            .addLink(new Link(buildContext("trace-1", "span-1", null, true)))
            .start();

    assertThat(span.isNoop()).isTrue();
}
```

**Why test the NOOP path?** The `addLink` method is a `default` method on the `Builder`
interface — meaning *all* existing Builder implementations inherit it automatically. The
NOOP test confirms that calling `addLink` in the fluent chain doesn't break anything.
When we build the SimpleTracer (Feature 6) and Brave bridge (Feature 8), they'll override
this to actually record/propagate links.

---

## 3. Implementing the Call Chain — Top-Down

### Layer 1: Link Class (the data carrier)

Link is a single-layer feature — there's no dispatch, processing, or infrastructure
layer. It's pure data:

```java
public class Link {

    public static final Link NOOP = new Link(TraceContext.NOOP, Collections.emptyMap());

    private final TraceContext traceContext;
    private final Map<String, Object> tags;

    public Link(TraceContext traceContext, Map<String, Object> tags) {
        this.traceContext = traceContext;
        this.tags = tags;
    }

    public Link(TraceContext traceContext) {
        this(traceContext, Collections.emptyMap());
    }

    public Link(Span span, Map<String, Object> tags) {
        this(span.context(), tags);
    }

    public Link(Span span) {
        this(span.context(), Collections.emptyMap());
    }

    public TraceContext getTraceContext() { return this.traceContext; }
    public Map<String, Object> getTags() { return this.tags; }

    // equals, hashCode, toString ...
}
```

**Design decisions**:
- **Four constructors** — `(TraceContext)`, `(TraceContext, Map)`, `(Span)`, `(Span, Map)`.
  The `Span` overloads are convenience — they delegate to `span.context()`. This matches the
  real framework exactly.
- **Immutable** — fields are `final`, no setters. Once created, a Link never changes. This
  is safe for concurrent use and consistent with `TraceContext`'s immutability.
- **`Collections.emptyMap()`** — tagless constructors use an empty immutable map, not `null`.
  This avoids null checks throughout the codebase.
- **`NOOP` constant** — `new Link(TraceContext.NOOP, Collections.emptyMap())`. Used by
  default implementations that need to return "something" without real data.

### Layer 2: Span.Builder.addLink — the integration point

The `addLink` method is added to the existing `Span.Builder` interface as a **default method**:

```java
interface Builder {
    // ... existing methods ...

    default Builder addLink(Link link) {
        return this;
    }

    Span start();
}
```

**Why `default`?** This method was added in v1.1.0, after `Span.Builder` was already
released. Making it a `default` method means:
1. Existing implementations (test fakes, bridges) **continue to compile** without changes
2. Implementations that *want* link support can **override** the default
3. The no-op behavior (ignoring the link) is a safe default for implementations that
   don't support links

This is the textbook use of Java 8 default methods for **interface evolution** — extending
a published interface without breaking existing implementations.

---

## 4. Try It Yourself

1. **Create a `LinkableSpanBuilder`** — write a `Span.Builder` implementation that collects
   links in a `List<Link>` and exposes them via a `getLinks()` method. Use it to verify
   that multiple links can be added to a single span builder.

2. **Write a batch processing scenario** — simulate a message batch where 5 messages each
   have their own trace context. Create a "process-batch" span and link it to all 5 message
   contexts. Verify that each link has the correct trace ID.

3. **Explore tag types** — create links with different tag value types (`String`, `Integer`,
   `Boolean`, `Double`) and verify they round-trip through `getTags()`. Consider: what
   would break if tags were `Map<String, String>` instead?

---

## 5. Why This Works

### Value Object Pattern
`Link` is a textbook value object — identity is determined by its contents (`traceContext`
+ `tags`), not by reference. Two `Link` instances with the same context and tags are
considered equal. This makes them safe to store in `Set`s and use in assertions.

### Constructor Telescoping
The four constructors form a telescoping pattern — each convenience constructor delegates
to the "full" constructor `(TraceContext, Map)`. This ensures consistent initialization
regardless of which constructor the client uses. The `Span`-accepting constructors simply
extract `span.context()` before delegating.

### Default Methods for API Evolution
`Span.Builder.addLink()` demonstrates how Java's default methods solve the
**interface evolution problem**. Before Java 8, adding a method to a published interface
broke every implementation. Default methods let the API grow while existing code keeps
working. The tracing facade uses this pattern extensively because it must remain compatible
with bridge implementations maintained by different teams.

---

## 6. What We Enhanced

| Component | State After Ch05 | Simplifications |
|-----------|-------------------|-----------------|
| `Link` | Concrete class with 4 constructors, `NOOP`, `equals`/`hashCode`/`toString` | — (matches real framework) |
| `Span.Builder.addLink(Link)` | Default no-op method on the interface | No collection support yet — actual storage comes in Feature 6 (SimpleSpanBuilder) |

**What's NOT connected yet**: `Link` is defined and `addLink()` exists on the Builder
interface, but no Builder implementation stores links yet. When we build SimpleTracer
(Feature 6), `SimpleSpanBuilder.addLink()` will collect links in a `List<Link>`. The
Brave bridge (Feature 8) will map links to Brave's native link support.

**What this feature enables**: Feature 6 depends on Features 1–5, and Feature 2
(`FinishedSpan`) references `Link` in its `getLinks()` method. With `Link` now defined,
the `FinishedSpan` interface is fully resolvable.

---

## 7. Connection to Real Framework

| Simplified Class | Real Framework Class | Source File |
|------------------|----------------------|-------------|
| `io.simpletracing.Link` | `io.micrometer.tracing.Link` | `micrometer-tracing/src/main/java/io/micrometer/tracing/Link.java` |
| `io.simpletracing.Span.Builder.addLink(Link)` | `io.micrometer.tracing.Span.Builder.addLink(Link)` | `micrometer-tracing/src/main/java/io/micrometer/tracing/Span.java` |

**Real framework extras we skipped**:
- Nothing significant — this is one of the simplest types in the framework. Our `Link` is
  a near-exact replica of the real one. The only difference is cosmetic: the real framework
  has `@since 1.1.0` annotations on every constructor.

---

## 8. Complete Code

### `simple-tracing-api/src/main/java/io/simpletracing/Link.java` [NEW]

```java
package io.simpletracing;

import java.util.Collections;
import java.util.Map;
import java.util.Objects;

/**
 * Represents an association between spans that may belong to different traces.
 * Links are used in batch processing where a single span processes messages from
 * multiple independent traces — each incoming message's context is linked to the
 * processing span.
 *
 * <p>A link holds a {@link TraceContext} (identifying the linked span) and optional
 * tags (metadata about the association).
 *
 * <p>Usage example:
 * <pre>{@code
 * Span batchSpan = tracer.nextSpan().name("process-batch").start();
 * for (TraceContext msgCtx : incomingMessages) {
 *     Span processing = tracer.spanBuilder()
 *         .name("process-message")
 *         .addLink(new Link(msgCtx))
 *         .start();
 *     // ... process message ...
 *     processing.end();
 * }
 * }</pre>
 *
 * <p>Maps to: {@code io.micrometer.tracing.Link}
 *
 * @since 1.1.0
 * @see Span.Builder#addLink(Link)
 */
public class Link {

    /**
     * A no-op link pointing to {@link TraceContext#NOOP} with no tags.
     */
    public static final Link NOOP = new Link(TraceContext.NOOP, Collections.emptyMap());

    private final TraceContext traceContext;

    private final Map<String, Object> tags;

    /**
     * Creates a link to the given trace context with associated tags.
     * @param traceContext the linked span's trace context
     * @param tags metadata about the association
     */
    public Link(TraceContext traceContext, Map<String, Object> tags) {
        this.traceContext = traceContext;
        this.tags = tags;
    }

    /**
     * Creates a link to the given trace context with no tags.
     * @param traceContext the linked span's trace context
     */
    public Link(TraceContext traceContext) {
        this(traceContext, Collections.emptyMap());
    }

    /**
     * Creates a link to the given span's trace context with associated tags.
     * @param span the linked span
     * @param tags metadata about the association
     */
    public Link(Span span, Map<String, Object> tags) {
        this(span.context(), tags);
    }

    /**
     * Creates a link to the given span's trace context with no tags.
     * @param span the linked span
     */
    public Link(Span span) {
        this(span.context(), Collections.emptyMap());
    }

    /**
     * Returns the trace context of the linked span.
     * @return trace context (never null)
     */
    public TraceContext getTraceContext() {
        return this.traceContext;
    }

    /**
     * Returns the tags associated with this link.
     * @return tags map (never null, may be empty)
     */
    public Map<String, Object> getTags() {
        return this.tags;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        Link link = (Link) o;
        return Objects.equals(this.traceContext, link.traceContext) && Objects.equals(this.tags, link.tags);
    }

    @Override
    public int hashCode() {
        return Objects.hash(this.traceContext, this.tags);
    }

    @Override
    public String toString() {
        return "Link{traceContext=" + this.traceContext + ", tags=" + this.tags + "}";
    }

}
```

### `simple-tracing-api/src/main/java/io/simpletracing/Span.java` [MODIFIED]

Added `addLink(Link)` default method to `Span.Builder`:

```java
// Added inside the Builder interface, before start():

        /**
         * Adds a {@link Link} to the span being built. Links associate spans across
         * different traces — commonly used in batch processing where one span processes
         * messages from multiple independent traces.
         *
         * <p>This is a default method that does nothing — bridge implementations override
         * it to wire the link into the underlying tracing library.
         * @param link the link to add
         * @return this for chaining
         * @since 1.1.0
         */
        default Builder addLink(Link link) {
            return this;
        }
```

### `simple-tracing-api/src/test/java/io/simpletracing/LinkTest.java` [NEW]

```java
package io.simpletracing;

import java.util.Map;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Client-perspective tests for Feature 5: Link.
 *
 * <p>These tests verify that {@link Link} correctly associates spans across traces
 * — holding a {@link TraceContext} and optional tags for the linked span.
 */
class LinkTest {

    // ── Construction Tests ────────────────────────────────────────────────

    @Test
    void shouldCreateLink_fromTraceContext() {
        TraceContext context = buildContext("trace-abc", "span-def", null, true);

        Link link = new Link(context);

        assertThat(link.getTraceContext()).isSameAs(context);
        assertThat(link.getTags()).isEmpty();
    }

    @Test
    void shouldCreateLink_fromTraceContextAndTags() {
        TraceContext context = buildContext("trace-abc", "span-def", null, true);
        Map<String, Object> tags = Map.of("batch.index", 3, "source", "queue-a");

        Link link = new Link(context, tags);

        assertThat(link.getTraceContext()).isSameAs(context);
        assertThat(link.getTags()).containsEntry("batch.index", 3);
        assertThat(link.getTags()).containsEntry("source", "queue-a");
    }

    @Test
    void shouldCreateLink_fromSpan() {
        TraceContext context = buildContext("trace-abc", "span-def", null, true);
        Span span = new StubSpan(context);

        Link link = new Link(span);

        assertThat(link.getTraceContext()).isSameAs(context);
        assertThat(link.getTags()).isEmpty();
    }

    @Test
    void shouldCreateLink_fromSpanAndTags() {
        TraceContext context = buildContext("trace-abc", "span-def", null, true);
        Span span = new StubSpan(context);
        Map<String, Object> tags = Map.of("priority", "high");

        Link link = new Link(span, tags);

        assertThat(link.getTraceContext()).isSameAs(context);
        assertThat(link.getTags()).containsEntry("priority", "high");
    }

    // ── NOOP Tests ────────────────────────────────────────────────────────

    @Test
    void noopLink_shouldHaveNoopContext() {
        assertThat(Link.NOOP.getTraceContext()).isSameAs(TraceContext.NOOP);
        assertThat(Link.NOOP.getTags()).isEmpty();
    }

    // ── Equality Tests ────────────────────────────────────────────────────

    @Test
    void shouldBeEqual_whenSameContextAndTags() {
        TraceContext ctx = buildContext("trace-1", "span-1", null, true);
        Link link1 = new Link(ctx, Map.of("key", "val"));
        Link link2 = new Link(ctx, Map.of("key", "val"));

        assertThat(link1).isEqualTo(link2);
        assertThat(link1.hashCode()).isEqualTo(link2.hashCode());
    }

    @Test
    void shouldNotBeEqual_whenDifferentContext() {
        TraceContext ctx1 = buildContext("trace-1", "span-1", null, true);
        TraceContext ctx2 = buildContext("trace-2", "span-2", null, true);

        Link link1 = new Link(ctx1);
        Link link2 = new Link(ctx2);

        assertThat(link1).isNotEqualTo(link2);
    }

    @Test
    void shouldNotBeEqual_whenDifferentTags() {
        TraceContext ctx = buildContext("trace-1", "span-1", null, true);

        Link link1 = new Link(ctx, Map.of("a", "1"));
        Link link2 = new Link(ctx, Map.of("a", "2"));

        assertThat(link1).isNotEqualTo(link2);
    }

    @Test
    void shouldNotBeEqual_toNull() {
        Link link = new Link(TraceContext.NOOP);
        assertThat(link).isNotEqualTo(null);
    }

    // ── toString Test ─────────────────────────────────────────────────────

    @Test
    void toString_shouldContainContextAndTags() {
        TraceContext ctx = buildContext("trace-abc", "span-def", null, true);
        Link link = new Link(ctx, Map.of("key", "val"));

        String str = link.toString();
        assertThat(str).contains("traceContext=");
        assertThat(str).contains("tags=");
    }

    // ── Span.Builder.addLink Tests ────────────────────────────────────────

    @Test
    void noopBuilder_addLink_shouldReturnThis() {
        Span.Builder builder = Span.Builder.NOOP;
        Link link = new Link(TraceContext.NOOP);

        Span.Builder result = builder.addLink(link);

        assertThat(result).isSameAs(builder);
    }

    @Test
    void noopBuilder_addLink_shouldStillStartNoopSpan() {
        Span span = Span.Builder.NOOP
                .name("batch-process")
                .addLink(new Link(buildContext("trace-1", "span-1", null, true)))
                .start();

        assertThat(span.isNoop()).isTrue();
    }

    // ── Helpers ───────────────────────────────────────────────────────────

    private TraceContext buildContext(String traceId, String spanId, String parentId, Boolean sampled) {
        return new TraceContext() {
            @Override public String traceId() { return traceId; }
            @Override public String parentId() { return parentId; }
            @Override public String spanId() { return spanId; }
            @Override public Boolean sampled() { return sampled; }

            @Override
            public boolean equals(Object o) {
                if (this == o) return true;
                if (!(o instanceof TraceContext that)) return false;
                return java.util.Objects.equals(traceId(), that.traceId())
                        && java.util.Objects.equals(spanId(), that.spanId())
                        && java.util.Objects.equals(parentId(), that.parentId())
                        && java.util.Objects.equals(sampled(), that.sampled());
            }

            @Override
            public int hashCode() {
                return java.util.Objects.hash(traceId(), spanId(), parentId(), sampled());
            }
        };
    }

    private static class StubSpan implements Span {
        private final TraceContext context;
        StubSpan(TraceContext context) { this.context = context; }
        @Override public boolean isNoop() { return false; }
        @Override public TraceContext context() { return context; }
        @Override public Span start() { return this; }
        @Override public Span name(String name) { return this; }
        @Override public Span event(String value) { return this; }
        @Override public Span tag(String key, String value) { return this; }
        @Override public Span error(Throwable throwable) { return this; }
        @Override public void end() { }
        @Override public void abandon() { }
        @Override public Span remoteServiceName(String remoteServiceName) { return this; }
    }

}
```
