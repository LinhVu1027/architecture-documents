# Chapter 5: Client Integration — Wiring Retry into DefaultZaloBotClient

## What You'll Learn
- Where to inject `RetryTemplate` in the existing client architecture
- How to wrap `exchangeInternal()` without changing the public API
- Why retry belongs at the `exchangeInternal` level, not at each API method
- How to handle `RetryException` unwrapping for a clean caller experience
- The complete modified `DefaultZaloBotClient` with retry support

---

## 5.1 The Problem: Where Does Retry Go?

There are three possible integration points:

```
Option A: Each API method
─────────────────────────
client.sendMessage()       ← retry here?
  └→ DefaultRequestBodySpec
       └→ DefaultResponseSpec.call()
            └→ exchangeInternal()

Option B: ResponseSpec.call()
─────────────────────────────
client.sendMessage()
  └→ DefaultRequestBodySpec
       └→ DefaultResponseSpec.call()  ← retry here?
            └→ exchangeInternal()

Option C: exchangeInternal()            ✓ BEST
────────────────────────────
client.sendMessage()
  └→ DefaultRequestBodySpec
       └→ DefaultResponseSpec.call()
            └→ exchangeInternal()     ← retry here!
```

**Option C is correct** because:
- `exchangeInternal()` is the single gateway for ALL API calls
- It already handles exception mapping (IOException → ZaloBotClientException, error codes → ZaloBotApiException)
- Retrying here means every API method (`getMe()`, `sendMessage()`, `getUpdates()`, etc.) gets retry for free
- The retry wraps the complete request-response cycle, including serialization and deserialization

---

## 5.2 The Integration Plan

```
Before:
  exchangeInternal() {
      request = createRequest();
      response = request.execute();    ← single attempt
      return deserialize(response);
  }

After:
  exchangeInternal() {
      return retryTemplate.execute(() -> {
          request = createRequest();
          response = request.execute();   ← inside retry loop
          return deserialize(response);
      });
  }
```

The key insight: the **entire request/response cycle** is inside the retryable lambda. This means:
- A new HTTP request is created on each attempt (not reusing a stale connection)
- Serialization happens on each attempt (correct for POST bodies)
- The `RetryPolicy` filters on the actual exception type thrown by the error-mapping logic

---

## 5.3 Modified DefaultZaloBotClient

```java
package dev.linhvu.zalobot.client;

// ... existing imports ...
import dev.linhvu.zalobot.client.retry.RetryException;
import dev.linhvu.zalobot.client.retry.RetryPolicy;
import dev.linhvu.zalobot.client.retry.RetryTemplate;

final class DefaultZaloBotClient implements ZaloBotClient {

    private final ZaloBotUrl url;
    private final String botToken;
    private final ClientHttpRequestFactory clientHttpRequestFactory;
    private final JsonMapper jsonMapper;
    private final DefaultZaloBotClientBuilder builder;
    private final RetryTemplate retryTemplate;         // ← NEW

    public DefaultZaloBotClient(ZaloBotUrl url,
            String botToken,
            ClientHttpRequestFactory clientHttpRequestFactory,
            JsonMapper jsonMapper,
            DefaultZaloBotClientBuilder builder,
            RetryTemplate retryTemplate) {             // ← NEW parameter
        this.url = url;
        this.botToken = botToken;
        this.clientHttpRequestFactory = clientHttpRequestFactory;
        this.jsonMapper = jsonMapper;
        this.builder = builder;
        this.retryTemplate = retryTemplate;            // ← NEW
    }

    // ... getMe(), getUpdates(), sendMessage(), etc. — UNCHANGED ...

    private <N> ZaloApiResponse<N> exchangeInternal(
            HttpMethod method, String methodPath,
            Map<String, String> headers, Object body, Class<N> clazz) {

        try {
            // ══════════════════════════════════════════════════
            // Wrap the entire request/response in RetryTemplate
            // ══════════════════════════════════════════════════
            return this.retryTemplate.execute(() -> {
                return doExchange(method, methodPath, headers, body, clazz);
            });
        }
        catch (RetryException e) {
            // ══════════════════════════════════════════════════
            // Unwrap RetryException to preserve existing API
            // ══════════════════════════════════════════════════
            Throwable cause = e.getCause();
            if (cause instanceof ZaloBotException zbe) {
                throw zbe;  // Rethrow the original SDK exception
            }
            throw new ZaloBotClientException(
                "Request failed after " + e.getRetryCount() + " retries: "
                    + cause.getMessage(), cause);
        }
    }

    /**
     * The actual HTTP exchange logic — extracted from the old exchangeInternal().
     * This method is called by RetryTemplate on each attempt.
     */
    private <N> ZaloApiResponse<N> doExchange(
            HttpMethod method, String methodPath,
            Map<String, String> headers, Object body, Class<N> clazz) {

        URI uri = buildUri(methodPath);
        ClientHttpRequest request = this.clientHttpRequestFactory.createRequest(uri, method);

        Map<String, String> requestHeaders = request.getHeaders();
        requestHeaders.putAll(headers);

        if (body != null) {
            try {
                byte[] bodyBytes = this.jsonMapper.writeValueAsBytes(body);
                OutputStream outputStream = request.getBody();
                outputStream.write(bodyBytes);
                outputStream.flush();
            }
            catch (IOException e) {
                throw new ZaloBotSerializationException(
                    "Failed to serialize request body", e);
            }
        }

        try (ClientHttpResponse response = request.execute()) {
            int httpStatus = response.getStatusCode();
            JavaType javaType = this.jsonMapper.getTypeFactory()
                .constructParametricType(ZaloApiResponse.class, clazz);
            ZaloApiResponse<N> apiResponse = this.jsonMapper
                .readValue(response.getBody(), javaType);

            if (apiResponse != null && !apiResponse.ok()) {
                ZaloErrorCode code = ZaloErrorCode.fromCode(apiResponse.errorCode());
                String description = code.getDescription();
                if (code.isAuthenticationError()) {
                    throw new ZaloBotAuthenticationException(
                        httpStatus, apiResponse.errorCode(), description);
                }
                throw new ZaloBotApiException(
                    httpStatus, apiResponse.errorCode(), description);
            }
            return apiResponse;
        }
        catch (ZaloBotException e) {
            throw e;  // SDK exceptions pass through for retry filtering
        }
        catch (IOException e) {
            throw new ZaloBotClientException(
                "HTTP request failed: " + e.getMessage(), e);
        }
    }

    // Inner classes DefaultRequestBodySpec, DefaultResponseSpec — UNCHANGED
}
```

> ★ **Insight** ─────────────────────────────────────
> - The `RetryException` unwrapping in `exchangeInternal()` is critical for **backward compatibility**. Without it, callers who catch `ZaloBotApiException` would suddenly need to catch `RetryException` instead. By unwrapping, the public API contract is preserved: callers still see `ZaloBotClientException`, `ZaloBotApiException`, etc.
> - This unwrapping is exactly what Spring's `RetryTemplate.invoke(Supplier)` does — it catches `RetryException` and rethrows the original `RuntimeException`. Our version is slightly different because `ZaloBotException` is an unchecked exception hierarchy, so we check for it specifically.
> - **Why create a new request on each attempt**: HTTP connections can be in a bad state after a failed attempt (half-written body, closed stream). Creating a fresh request via `clientHttpRequestFactory.createRequest()` ensures each attempt starts clean. This is why the entire exchange (not just `request.execute()`) is inside the retryable lambda.
> ─────────────────────────────────────────────────────

---

## 5.4 Exception Flow Through Retry

```
Attempt 1: doExchange() → IOException → ZaloBotClientException
  ↓
RetryPolicy.shouldRetry(ZaloBotClientException) → true
  ↓ (backoff sleep)
Attempt 2: doExchange() → IOException → ZaloBotClientException
  ↓
RetryPolicy.shouldRetry(ZaloBotClientException) → true
  ↓ (backoff sleep)
Attempt 3: doExchange() → success → ZaloApiResponse
  ↓
Return to caller ✓


--- OR (non-retryable error) ---

Attempt 1: doExchange() → ZaloBotAuthenticationException
  ↓
RetryPolicy.shouldRetry(ZaloBotAuthenticationException) → false
  ↓
RetryException(cause=ZaloBotAuthenticationException)
  ↓
exchangeInternal() unwraps → throw ZaloBotAuthenticationException
  ↓
Caller receives ZaloBotAuthenticationException immediately ✓
```

---

## 5.5 Default RetryPolicy for the Client

The builder (Chapter 7) will configure this, but here's the recommended default:

```java
// In DefaultZaloBotClientBuilder, when no custom retry is configured:
RetryPolicy defaultPolicy = RetryPolicy.builder()
    .includes(ZaloBotClientException.class, ZaloBotApiException.class)
    .excludes(ZaloBotAuthenticationException.class, ZaloBotSerializationException.class)
    .delay(Duration.ofSeconds(1))
    .multiplier(2.0)
    .maxDelay(Duration.ofSeconds(30))
    .maxRetries(3)
    .jitter(Duration.ofMillis(200))
    .build();

RetryTemplate retryTemplate = new RetryTemplate(defaultPolicy);
```

---

## 5.6 Connection to Real Spring Framework

Spring Framework does NOT wire `RetryTemplate` into `RestClient` or `RestTemplate` automatically — retry is applied externally by the user. Our approach is more opinionated: we build retry INTO the client by default.

| Aspect | Spring Framework | Zalobot SDK |
|--------|-----------------|-------------|
| Retry integration | User wraps calls with `retryTemplate.execute()` | Built into `exchangeInternal()` |
| Exception unwrapping | `invoke(Supplier)` unwraps to RuntimeException | `exchangeInternal()` unwraps to `ZaloBotException` |
| Default policy | No default (user must configure) | Sensible default with exception filtering |
| Opt-out | Don't use RetryTemplate | Set `RetryPolicy.withMaxRetries(0)` or provide no-op policy |

This is a deliberate design choice: an SDK should be resilient by default. Users can still disable retry if needed.

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Integration point** | `exchangeInternal()` — the single gateway for all API calls |
| **Full-cycle retry** | Each retry creates a new request, re-serializes, re-deserializes |
| **Exception unwrapping** | `RetryException` → original `ZaloBotException` for backward compatibility |
| **Non-retryable fast exit** | Auth/serialization errors bypass retry immediately |
| **doExchange()** | Extracted method containing the actual HTTP logic (called by retry loop) |

**Next: Chapter 6** — We'll add concurrency throttling to prevent overwhelming the Zalo API with too many simultaneous requests.
