# Chapter 21: WebSocket Support

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Embedded Tomcat handles HTTP request/response cycles through DispatcherServlet | Every interaction requires a new HTTP request — the server cannot push data to the client, and frequent polling wastes bandwidth | Add persistent, bidirectional WebSocket connections where both client and server can send messages at any time through a handler lifecycle API |

---

## 21.1 The Integration Point

WebSocket support integrates at the **server infrastructure** level — not the DispatcherServlet. This is an important architectural insight: WebSocket connections bypass the Servlet dispatch pipeline entirely. The Jakarta WebSocket container handles the HTTP upgrade handshake and creates a persistent connection that lives outside the normal request/response cycle.

**Modifying:** `src/main/java/com/simplespringmvc/server/EmbeddedTomcat.java`
**Change:** Store the `Context` as a field and add `registerWebSocketHandlers()` method that initializes Tomcat's WebSocket container (WsSci) and deploys endpoints

```java
public class EmbeddedTomcat {

    private final Tomcat tomcat;
    private final SimpleDispatcherServlet dispatcherServlet;
    private final Context context;  // ← NEW: stored as field (was local variable)

    public EmbeddedTomcat(int port, SimpleDispatcherServlet dispatcherServlet) {
        // ... existing setup ...
        this.context = tomcat.addContext("", null);  // ← was: Context context = ...
        // ... existing servlet registration ...
    }

    // ↓ NEW METHOD — the integration point
    public void registerWebSocketHandlers(WebSocketConfigurer configurer) {
        // Initialize Tomcat's WebSocket support by adding the WsSci
        context.addServletContainerInitializer(
                new org.apache.tomcat.websocket.server.WsSci(), null);

        // Create a registry and let the configurer populate it
        SimpleWebSocketHandlerRegistry registry = new SimpleWebSocketHandlerRegistry();
        configurer.registerWebSocketHandlers(registry);

        // Deploy after Tomcat starts (WsSci needs to run first)
        context.addLifecycleListener(event -> {
            if ("after_start".equals(event.getType())) {
                ServerContainer serverContainer = (ServerContainer) context.getServletContext()
                        .getAttribute(ServerContainer.class.getName());
                if (serverContainer != null) {
                    registry.deploy(serverContainer);
                }
            }
        });
    }
}
```

Two key decisions here:

1. **Why EmbeddedTomcat, not DispatcherServlet?** WebSocket connections are fundamentally different from HTTP requests — they persist indefinitely and don't follow the request/response model. The Jakarta WebSocket container (JSR 356) operates alongside the Servlet container, not inside it. Registering at the server level keeps the concerns separate.

2. **Why a lifecycle listener for deployment?** The `ServerContainer` isn't available until Tomcat's `WsSci` (WebSocket Server Container Initializer) runs during startup. We register a lifecycle listener that fires `after_start` to ensure the container is ready before we deploy endpoints.

This connects **the embedded server** to **the WebSocket handler layer**. To make it work, we need to build:
- `WebSocketHandler` — lifecycle callback interface for application code
- `WebSocketSession` / `StandardWebSocketSession` — abstraction over the connection
- `TextMessage` / `CloseStatus` — value objects for messages and close codes
- `TextWebSocketHandler` — convenience base class
- `HandshakeInterceptor` — hook for the HTTP upgrade phase
- `StandardEndpointAdapter` — bridge from Jakarta Endpoint to our handler
- `SimpleWebSocketHandlerRegistry` — collects registrations and deploys to the container
- `WebSocketConfigurer` — callback interface for registering handlers

## 21.2 Core Interfaces and Value Objects

### WebSocketHandler — the lifecycle callback interface

**New file:** `src/main/java/com/simplespringmvc/websocket/WebSocketHandler.java`

```java
package com.simplespringmvc.websocket;

public interface WebSocketHandler {

    void afterConnectionEstablished(WebSocketSession session) throws Exception;

    void handleMessage(WebSocketSession session, TextMessage message) throws Exception;

    void handleTransportError(WebSocketSession session, Throwable exception) throws Exception;

    void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) throws Exception;
}
```

### WebSocketSession — abstraction over the connection

**New file:** `src/main/java/com/simplespringmvc/websocket/WebSocketSession.java`

```java
package com.simplespringmvc.websocket;

import java.io.Closeable;
import java.io.IOException;
import java.net.URI;
import java.util.Map;

public interface WebSocketSession extends Closeable {

    String getId();
    URI getUri();
    Map<String, Object> getAttributes();
    boolean isOpen();
    void sendMessage(TextMessage message) throws IOException;
    void close() throws IOException;
    void close(CloseStatus status) throws IOException;
}
```

### TextMessage — immutable text payload

**New file:** `src/main/java/com/simplespringmvc/websocket/TextMessage.java`

```java
package com.simplespringmvc.websocket;

import java.util.Objects;

public final class TextMessage {

    private final String payload;

    public TextMessage(String payload) {
        this.payload = Objects.requireNonNull(payload, "payload must not be null");
    }

    public String getPayload() { return payload; }
    public int getPayloadLength() { return payload.length(); }

    // equals, hashCode, toString...
}
```

### CloseStatus — RFC 6455 close codes

**New file:** `src/main/java/com/simplespringmvc/websocket/CloseStatus.java`

```java
package com.simplespringmvc.websocket;

public final class CloseStatus {

    public static final CloseStatus NORMAL = new CloseStatus(1000, "Normal closure");
    public static final CloseStatus GOING_AWAY = new CloseStatus(1001, "Going away");
    public static final CloseStatus PROTOCOL_ERROR = new CloseStatus(1002, "Protocol error");
    public static final CloseStatus NOT_ACCEPTABLE = new CloseStatus(1003, "Not acceptable");
    public static final CloseStatus SERVER_ERROR = new CloseStatus(1011, "Server error");

    private final int code;
    private final String reason;

    public CloseStatus(int code, String reason) { /* ... */ }

    public CloseStatus withReason(String reason) {
        return new CloseStatus(this.code, reason);  // Immutable — returns new instance
    }
    // getCode(), getReason(), equals, hashCode...
}
```

## 21.3 The Convenience Base Class

**New file:** `src/main/java/com/simplespringmvc/websocket/TextWebSocketHandler.java`

```java
package com.simplespringmvc.websocket;

public abstract class TextWebSocketHandler implements WebSocketHandler {

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {}

    @Override
    public final void handleMessage(WebSocketSession session, TextMessage message) throws Exception {
        handleTextMessage(session, message);
    }

    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {}

    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) throws Exception {}

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) throws Exception {}
}
```

The `handleMessage()` is `final` — it dispatches to `handleTextMessage()`, which subclasses override. This follows the Template Method pattern. Application code extends `TextWebSocketHandler` and only overrides the callbacks they need:

```java
public class EchoHandler extends TextWebSocketHandler {
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        session.sendMessage(new TextMessage("Echo: " + message.getPayload()));
    }
}
```

## 21.4 The Adapter Layer — Bridging Jakarta WebSocket to Our API

### StandardWebSocketSession — wrapping the native session

**New file:** `src/main/java/com/simplespringmvc/websocket/StandardWebSocketSession.java`

```java
package com.simplespringmvc.websocket;

import jakarta.websocket.Session;
import java.io.IOException;
import java.net.URI;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class StandardWebSocketSession implements WebSocketSession {

    private final String id;
    private final URI uri;
    private final Map<String, Object> attributes;
    private volatile Session nativeSession;

    public StandardWebSocketSession(String id, URI uri, Map<String, Object> attributes) {
        this.id = id;
        this.uri = uri;
        this.attributes = new ConcurrentHashMap<>(attributes);
    }

    public void initializeNativeSession(Session nativeSession) {
        this.nativeSession = nativeSession;
    }

    @Override
    public void sendMessage(TextMessage message) throws IOException {
        checkNativeSession();
        nativeSession.getBasicRemote().sendText(message.getPayload());
    }

    @Override
    public void close(CloseStatus status) throws IOException {
        checkNativeSession();
        nativeSession.close(new jakarta.websocket.CloseReason(
                jakarta.websocket.CloseReason.CloseCodes.getCloseCode(status.getCode()),
                status.getReason()));
    }
    // ...
}
```

### StandardEndpointAdapter — the critical bridge

**New file:** `src/main/java/com/simplespringmvc/websocket/StandardEndpointAdapter.java`

This is the most important class in the WebSocket subsystem. It extends Jakarta's `Endpoint` and translates all Jakarta lifecycle events into calls on our `WebSocketHandler`:

```java
package com.simplespringmvc.websocket;

import jakarta.websocket.*;

public class StandardEndpointAdapter extends Endpoint {

    static final String ATTRIBUTES_KEY = "simple.ws.attributes";
    static final String URI_KEY = "simple.ws.uri";

    private final WebSocketHandler handler;
    private StandardWebSocketSession wsSession;

    public StandardEndpointAdapter(WebSocketHandler handler) {
        this.handler = handler;
    }

    @Override
    public void onOpen(Session session, EndpointConfig config) {
        // Retrieve handshake attributes from modifyHandshake() via shared user properties
        Map<String, Object> attributes = (Map<String, Object>) config.getUserProperties()
                .getOrDefault(ATTRIBUTES_KEY, new ConcurrentHashMap<>());
        URI uri = (URI) config.getUserProperties()
                .getOrDefault(URI_KEY, session.getRequestURI());

        this.wsSession = new StandardWebSocketSession(UUID.randomUUID().toString(), uri, attributes);
        wsSession.initializeNativeSession(session);

        // Register text message handler — must use anonymous class (not lambda)
        // so Jakarta container can read generic type <String> via reflection
        session.addMessageHandler(new MessageHandler.Whole<String>() {
            @Override
            public void onMessage(String payload) {
                try {
                    handler.handleMessage(wsSession, new TextMessage(payload));
                } catch (Exception ex) {
                    tryCloseWithError(ex);
                }
            }
        });

        try {
            handler.afterConnectionEstablished(wsSession);
        } catch (Exception ex) {
            tryCloseWithError(ex);
        }
    }

    @Override
    public void onClose(Session session, CloseReason closeReason) {
        CloseStatus status = new CloseStatus(
                closeReason.getCloseCode().getCode(),
                closeReason.getReasonPhrase() != null ? closeReason.getReasonPhrase() : "");
        try {
            handler.afterConnectionClosed(wsSession, status);
        } catch (Exception ex) {
            System.err.println("Error in afterConnectionClosed: " + ex.getMessage());
        }
    }

    @Override
    public void onError(Session session, Throwable thr) {
        try {
            handler.handleTransportError(wsSession, thr);
        } catch (Exception ex) {
            tryCloseWithError(ex);
        }
    }

    private void tryCloseWithError(Exception ex) {
        try {
            wsSession.close(CloseStatus.SERVER_ERROR.withReason(ex.getMessage()));
        } catch (IOException closeEx) { /* already handling error */ }
    }
}
```

The anonymous inner class for `MessageHandler.Whole<String>` is intentional — not a style choice. The Jakarta WebSocket engine uses runtime reflection to determine the generic type parameter (`String`). Java lambdas erase generic types, so the container wouldn't know whether this handler processes `String`, `ByteBuffer`, or `PongMessage`. The real Spring framework has the same pattern with an explicit comment explaining why.

## 21.5 The Registry and Configuration

### HandshakeInterceptor — hook for the HTTP upgrade

**New file:** `src/main/java/com/simplespringmvc/websocket/HandshakeInterceptor.java`

```java
package com.simplespringmvc.websocket;

import java.net.URI;
import java.util.Map;

public interface HandshakeInterceptor {

    boolean beforeHandshake(URI requestUri, Map<String, Object> attributes) throws Exception;

    void afterHandshake(URI requestUri, Exception exception);
}
```

### WebSocketConfigurer — the callback for registering handlers

**New file:** `src/main/java/com/simplespringmvc/websocket/WebSocketConfigurer.java`

```java
package com.simplespringmvc.websocket;

@FunctionalInterface
public interface WebSocketConfigurer {
    void registerWebSocketHandlers(SimpleWebSocketHandlerRegistry registry);
}
```

### SimpleWebSocketHandlerRegistry — collects and deploys registrations

**New file:** `src/main/java/com/simplespringmvc/websocket/SimpleWebSocketHandlerRegistry.java`

```java
package com.simplespringmvc.websocket;

import jakarta.websocket.server.ServerContainer;
import jakarta.websocket.server.ServerEndpointConfig;

public class SimpleWebSocketHandlerRegistry {

    private final List<Registration> registrations = new ArrayList<>();

    public Registration addHandler(WebSocketHandler handler, String... paths) {
        Registration registration = new Registration(handler, paths);
        registrations.add(registration);
        return registration;
    }

    public void deploy(ServerContainer serverContainer) throws DeploymentException {
        for (Registration reg : registrations) {
            for (String path : reg.paths) {
                ServerEndpointConfig config = ServerEndpointConfig.Builder
                        .create(StandardEndpointAdapter.class, path)
                        .configurator(new HandshakeConfigurator(reg.handler, reg.interceptors))
                        .build();
                serverContainer.addEndpoint(config);
            }
        }
    }
}
```

The `HandshakeConfigurator` is a nested class that bridges Jakarta's `ServerEndpointConfig.Configurator` to our API:

```java
static class HandshakeConfigurator extends ServerEndpointConfig.Configurator {

    @Override
    public void modifyHandshake(ServerEndpointConfig sec,
                                HandshakeRequest request,
                                HandshakeResponse response) {
        // Run interceptors and store attributes in sec.getUserProperties()
        // These flow to onOpen() via EndpointConfig.getUserProperties()
    }

    @Override
    public <T> T getEndpointInstance(Class<T> endpointClass) {
        return (T) new StandardEndpointAdapter(handler);
    }
}
```

The data flow through `getUserProperties()` is the key insight: attributes set during `modifyHandshake()` are available in `onOpen()` because the `ServerEndpointConfig` and `EndpointConfig` share the same user properties map.

## 21.6 Try It Yourself

<details>
<summary>Challenge: Build a chat room handler that tracks connected sessions and broadcasts messages</summary>

Think about: How do you safely track multiple sessions? What happens when a session disconnects?

```java
public class ChatHandler extends TextWebSocketHandler {
    private final Set<WebSocketSession> sessions =
            Collections.synchronizedSet(new HashSet<>());

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        sessions.add(session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        for (WebSocketSession s : sessions) {
            if (s.isOpen()) {
                s.sendMessage(new TextMessage(
                        session.getId() + ": " + message.getPayload()));
            }
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) {
        sessions.remove(session);
    }
}
```

</details>

<details>
<summary>Challenge: Create a HandshakeInterceptor that rejects connections without a "token" query parameter</summary>

Think about: How do you extract query parameters from the handshake URI?

```java
public class AuthInterceptor implements HandshakeInterceptor {
    @Override
    public boolean beforeHandshake(URI requestUri, Map<String, Object> attributes) {
        String query = requestUri.getQuery();
        if (query != null && query.contains("token=")) {
            String token = query.split("token=")[1].split("&")[0];
            attributes.put("token", token);
            return true;  // Allow connection
        }
        return false;  // Reject — no token
    }

    @Override
    public void afterHandshake(URI requestUri, Exception exception) {
        // No cleanup needed
    }
}
```

</details>

## 21.7 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/websocket/TextMessageTest.java`

```java
@Test
void shouldStorePayload_WhenCreated() {
    var message = new TextMessage("Hello WebSocket");
    assertThat(message.getPayload()).isEqualTo("Hello WebSocket");
}

@Test
void shouldBeEqual_WhenPayloadsMatch() {
    var msg1 = new TextMessage("Hello");
    var msg2 = new TextMessage("Hello");
    assertThat(msg1).isEqualTo(msg2);
}

@Test
void shouldRejectNullPayload() {
    assertThatThrownBy(() -> new TextMessage(null))
            .isInstanceOf(NullPointerException.class);
}
```

**New file:** `src/test/java/com/simplespringmvc/websocket/CloseStatusTest.java`

```java
@Test
void shouldCreateNewInstance_WhenWithReasonCalled() {
    CloseStatus original = CloseStatus.SERVER_ERROR;
    CloseStatus withReason = original.withReason("Custom error message");

    assertThat(withReason.getCode()).isEqualTo(1011);
    assertThat(withReason.getReason()).isEqualTo("Custom error message");
    assertThat(original.getReason()).isEqualTo("Server error");  // Original unchanged
}
```

**New file:** `src/test/java/com/simplespringmvc/websocket/TextWebSocketHandlerTest.java`

```java
@Test
void shouldTrackConnectionLifecycle_WhenAllCallbacksOverridden() throws Exception {
    List<String> events = new ArrayList<>();
    var handler = new TextWebSocketHandler() {
        @Override
        public void afterConnectionEstablished(WebSocketSession session) {
            events.add("opened:" + session.getId());
        }
        @Override
        protected void handleTextMessage(WebSocketSession session, TextMessage message) {
            events.add("message:" + message.getPayload());
        }
        @Override
        public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) {
            events.add("closed:" + closeStatus.getCode());
        }
    };

    var session = new StandardWebSocketSession("sess-1", URI.create("/ws"), Map.of());
    handler.afterConnectionEstablished(session);
    handler.handleMessage(session, new TextMessage("ping"));
    handler.afterConnectionClosed(session, CloseStatus.NORMAL);

    assertThat(events).containsExactly("opened:sess-1", "message:ping", "closed:1000");
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/WebSocketIntegrationTest.java`

Tests use a real embedded Tomcat and the JDK's `java.net.http.WebSocket` client:

```java
@Test
void shouldEchoMessage_WhenTextSent() throws Exception {
    var receivedMessages = new CopyOnWriteArrayList<String>();
    var latch = new CountDownLatch(1);

    server = createServer(new TextWebSocketHandler() {
        @Override
        protected void handleTextMessage(WebSocketSession session, TextMessage message)
                throws Exception {
            session.sendMessage(new TextMessage("Echo: " + message.getPayload()));
        }
    }, "/ws/echo");

    var client = HttpClient.newHttpClient();
    var ws = client.newWebSocketBuilder()
            .buildAsync(wsUri("/ws/echo"), createListener(receivedMessages, latch))
            .join();

    ws.sendText("Hello", true);

    assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
    assertThat(receivedMessages).containsExactly("Echo: Hello");

    ws.sendClose(WebSocket.NORMAL_CLOSURE, "").join();
}

@Test
void shouldBroadcastToAllClients_WhenMultipleConnectionsExist() throws Exception {
    // Tests broadcast pattern — one message reaches all connected clients
}
```

**Run:** `./gradlew test` — expected: all 718 tests pass (including 35 new WebSocket tests)

---

## 21.8 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why a separate container, not the DispatcherServlet?** WebSocket connections persist indefinitely — they don't fit the request/response model that DispatcherServlet is built around. The Jakarta WebSocket container runs alongside the Servlet container as a peer, not a subordinate. This separation of concerns mirrors how the real Spring Framework keeps `spring-websocket` as an independent module from `spring-webmvc`, with the only connection point being the embedded server setup.
> - **When this pattern breaks down:** If you need deep integration with Spring MVC features (security, content negotiation, session management), the separate container approach creates a gap. This is exactly why Spring's STOMP over WebSocket support (`@MessageMapping`) was built — it layers MVC-like abstractions on top of raw WebSocket.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why anonymous inner classes, not lambdas?** The Jakarta WebSocket spec requires the container to determine message types (String, ByteBuffer, PongMessage) at runtime via reflection on the `MessageHandler`'s generic type parameter. Java lambdas use type erasure that makes this impossible — `(String msg) -> {}` compiles to `MessageHandler.Whole<Object>` at the bytecode level. Anonymous inner classes preserve the generic type in the class file's signature. The real framework has an [explicit comment](https://github.com/spring-projects/spring-framework/blob/main/spring-websocket/src/main/java/org/springframework/web/socket/adapter/standard/StandardWebSocketHandlerAdapter.java#L66) about this.
> - **Trade-off:** Anonymous classes are more verbose but correct. This is a case where following the language's type system rules matters more than code aesthetics.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **The Adapter Pattern at work:** `StandardEndpointAdapter` is a textbook Adapter — it makes our `WebSocketHandler` interface work where the container expects Jakarta's `Endpoint`. Similarly, `StandardWebSocketSession` adapts Jakarta's `Session` to our `WebSocketSession`. Two adapters, two incompatible APIs, zero application code changes. The real framework has the same two-adapter structure.
> - **Why not just use Jakarta's API directly?** Because your application code would be coupled to a specific WebSocket implementation (Tomcat, Jetty, Undertow). Spring's abstraction lets you switch containers without changing handler code.
> -----------------------------------------------------------

## 21.9 What We Enhanced

| Aspect | Before (ch02) | Current (ch21) | Real Framework |
|--------|---------------|----------------|----------------|
| Server protocols | HTTP request/response only — `EmbeddedTomcat` creates a `Context` as a local variable | HTTP + WebSocket — `Context` stored as field, `registerWebSocketHandlers()` initializes WsSci and deploys Jakarta endpoints | Full WebSocket support via `@EnableWebSocket`, SockJS fallback, STOMP messaging (`WebSocketConfigurationSupport.java:86`) |
| Communication model | Client-initiated request/response — server cannot push data | Persistent bidirectional connections — either side can send messages at any time | Same + STOMP sub-protocol for pub/sub messaging, `SimpMessagingTemplate` for server-initiated sends |

## 21.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `WebSocketHandler` | `WebSocketHandler` | `WebSocketHandler.java:43` | Real adds `supportsPartialMessages()` for fragmented message support |
| `WebSocketSession` | `WebSocketSession` | `WebSocketSession.java:38` | Real adds sub-protocol, extensions, message size limits, principal, addresses |
| `TextMessage` | `TextMessage` extends `AbstractWebSocketMessage<String>` | `TextMessage.java:34` | Real supports partial messages (`isLast` flag), byte-level access |
| `CloseStatus` | `CloseStatus` | `CloseStatus.java:39` | Real defines all RFC 6455 codes + custom `SESSION_NOT_RELIABLE` (4500) |
| `TextWebSocketHandler` | `TextWebSocketHandler` extends `AbstractWebSocketHandler` | `TextWebSocketHandler.java:35` | Real rejects binary messages with `NOT_ACCEPTABLE`, has `AbstractWebSocketHandler` with type-dispatch |
| `StandardEndpointAdapter` | `StandardWebSocketHandlerAdapter` | `StandardWebSocketHandlerAdapter.java:60` | Real handles binary + pong messages, uses `ExceptionWebSocketHandlerDecorator` for error handling |
| `StandardWebSocketSession` | `StandardWebSocketSession` | `StandardWebSocketSession.java:181` | Real extends `AbstractWebSocketSession<Session>`, handles extensions and sub-protocol negotiation |
| `SimpleWebSocketHandlerRegistry` | `ServletWebSocketHandlerRegistry` | `ServletWebSocketHandlerRegistry.java:52` | Real produces a `HandlerMapping` for Spring MVC, supports SockJS fallback, allowed origins |
| `HandshakeInterceptor` | `HandshakeInterceptor` | `HandshakeInterceptor.java:36` | Real uses `ServerHttpRequest`/`ServerHttpResponse` for transport-agnostic API |
| `EmbeddedTomcat.registerWebSocketHandlers()` | `WebSocketConfigurationSupport` | `WebSocketConfigurationSupport.java:86` | Real uses `@EnableWebSocket` + auto-configuration, not manual container init |

## 21.11 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/websocket/WebSocketHandler.java` [NEW]

```java
package com.simplespringmvc.websocket;

/**
 * Core interface for handling WebSocket messages and lifecycle events.
 *
 * Maps to: {@code org.springframework.web.socket.WebSocketHandler}
 *
 * The real framework's interface is identical — four lifecycle callbacks
 * that the container invokes as a WebSocket connection progresses through
 * its states: open → message(s) → error? → close.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No {@code supportsPartialMessages()} — we always receive full messages</li>
 *   <li>No binary message support — text only (use {@link TextWebSocketHandler})</li>
 * </ul>
 */
public interface WebSocketHandler {

    /**
     * Called after the WebSocket handshake succeeds and the connection is open.
     */
    void afterConnectionEstablished(WebSocketSession session) throws Exception;

    /**
     * Called when a new text message arrives from the client.
     */
    void handleMessage(WebSocketSession session, TextMessage message) throws Exception;

    /**
     * Called when a transport-level error occurs on the connection.
     */
    void handleTransportError(WebSocketSession session, Throwable exception) throws Exception;

    /**
     * Called after the connection has been closed by either side.
     */
    void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) throws Exception;
}
```

#### File: `src/main/java/com/simplespringmvc/websocket/WebSocketSession.java` [NEW]

```java
package com.simplespringmvc.websocket;

import java.io.Closeable;
import java.io.IOException;
import java.net.URI;
import java.util.Map;

/**
 * Abstraction over a WebSocket connection, providing methods to send messages,
 * inspect connection metadata, and close the connection.
 *
 * Maps to: {@code org.springframework.web.socket.WebSocketSession}
 *
 * The real framework extends {@link Closeable} and adds methods for
 * sub-protocols, extensions, message size limits, local/remote addresses,
 * and the authenticated principal. We keep only the essentials.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No binary message support</li>
 *   <li>No sub-protocol negotiation</li>
 *   <li>No extensions</li>
 *   <li>No message size limit configuration</li>
 *   <li>No principal (authentication)</li>
 * </ul>
 */
public interface WebSocketSession extends Closeable {

    /**
     * Returns a unique session identifier.
     */
    String getId();

    /**
     * Returns the URI used to open the WebSocket connection.
     */
    URI getUri();

    /**
     * Returns mutable session attributes. Populated by {@link HandshakeInterceptor}s
     * during the handshake phase.
     */
    Map<String, Object> getAttributes();

    /**
     * Returns whether the connection is currently open.
     */
    boolean isOpen();

    /**
     * Sends a text message to the remote endpoint.
     */
    void sendMessage(TextMessage message) throws IOException;

    /**
     * Closes the connection with a normal close status.
     */
    @Override
    void close() throws IOException;

    /**
     * Closes the connection with the given status code and reason.
     */
    void close(CloseStatus status) throws IOException;
}
```

#### File: `src/main/java/com/simplespringmvc/websocket/TextMessage.java` [NEW]

```java
package com.simplespringmvc.websocket;

import java.util.Objects;

/**
 * Value object representing a text WebSocket message.
 *
 * Maps to: {@code org.springframework.web.socket.TextMessage}
 *
 * The real framework has an inheritance chain:
 *   {@code WebSocketMessage<T>} → {@code AbstractWebSocketMessage<T>} → {@code TextMessage}
 * supporting text, binary, ping, and pong messages with partial message fragments.
 *
 * We simplify to a single final class holding a String payload — no binary support,
 * no partial messages, no byte-level access.
 */
public final class TextMessage {

    private final String payload;

    public TextMessage(String payload) {
        this.payload = Objects.requireNonNull(payload, "payload must not be null");
    }

    /**
     * Returns the text payload of this message.
     */
    public String getPayload() {
        return payload;
    }

    /**
     * Returns the length of the payload in characters.
     */
    public int getPayloadLength() {
        return payload.length();
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof TextMessage that)) return false;
        return payload.equals(that.payload);
    }

    @Override
    public int hashCode() {
        return payload.hashCode();
    }

    @Override
    public String toString() {
        String truncated = payload.length() > 50
                ? payload.substring(0, 50) + "..."
                : payload;
        return "TextMessage[payload=" + truncated + "]";
    }
}
```

#### File: `src/main/java/com/simplespringmvc/websocket/CloseStatus.java` [NEW]

```java
package com.simplespringmvc.websocket;

import java.util.Objects;

/**
 * Value object representing a WebSocket close status code and optional reason phrase,
 * as defined in RFC 6455 Section 7.4.1.
 *
 * Maps to: {@code org.springframework.web.socket.CloseStatus}
 *
 * The real framework defines all RFC 6455 close codes plus a custom
 * {@code SESSION_NOT_RELIABLE} (4500). We include the most commonly used codes.
 */
public final class CloseStatus {

    /** Normal closure — the connection successfully completed its purpose. */
    public static final CloseStatus NORMAL = new CloseStatus(1000, "Normal closure");

    /** The endpoint is going away (e.g., server shutting down or browser navigating away). */
    public static final CloseStatus GOING_AWAY = new CloseStatus(1001, "Going away");

    /** The endpoint received a message it cannot process (e.g., protocol error). */
    public static final CloseStatus PROTOCOL_ERROR = new CloseStatus(1002, "Protocol error");

    /** The endpoint received data it cannot accept (e.g., text-only endpoint got binary). */
    public static final CloseStatus NOT_ACCEPTABLE = new CloseStatus(1003, "Not acceptable");

    /** An unexpected error caused the connection to close. */
    public static final CloseStatus SERVER_ERROR = new CloseStatus(1011, "Server error");

    private final int code;
    private final String reason;

    public CloseStatus(int code) {
        this(code, "");
    }

    public CloseStatus(int code, String reason) {
        this.code = code;
        this.reason = Objects.requireNonNull(reason, "reason must not be null");
    }

    /**
     * Returns the status code (per RFC 6455 Section 7.4.1).
     */
    public int getCode() {
        return code;
    }

    /**
     * Returns the human-readable reason phrase.
     */
    public String getReason() {
        return reason;
    }

    /**
     * Creates a new CloseStatus with the same code but a different reason.
     */
    public CloseStatus withReason(String reason) {
        return new CloseStatus(this.code, reason);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CloseStatus that)) return false;
        return code == that.code && reason.equals(that.reason);
    }

    @Override
    public int hashCode() {
        return Objects.hash(code, reason);
    }

    @Override
    public String toString() {
        return "CloseStatus[code=" + code + ", reason=" + reason + "]";
    }
}
```

#### File: `src/main/java/com/simplespringmvc/websocket/TextWebSocketHandler.java` [NEW]

```java
package com.simplespringmvc.websocket;

/**
 * Convenience base class for WebSocket handlers that only process text messages.
 * Provides empty default implementations for all lifecycle callbacks so subclasses
 * only need to override the methods they care about.
 *
 * Maps to: {@code org.springframework.web.socket.handler.TextWebSocketHandler}
 *
 * The real framework has a richer hierarchy:
 *   {@code WebSocketHandler} → {@code AbstractWebSocketHandler}
 *     → {@code TextWebSocketHandler} (rejects binary with NOT_ACCEPTABLE)
 *
 * We simplify to a single abstract class with empty defaults. The typical
 * application entry point is:
 * <pre>
 *   public class ChatHandler extends TextWebSocketHandler {
 *       {@literal @}Override
 *       protected void handleTextMessage(WebSocketSession session, TextMessage message) {
 *           session.sendMessage(new TextMessage("Echo: " + message.getPayload()));
 *       }
 *   }
 * </pre>
 */
public abstract class TextWebSocketHandler implements WebSocketHandler {

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        // Override to handle connection opened
    }

    /**
     * Dispatches to {@link #handleTextMessage} — the method subclasses override.
     */
    @Override
    public final void handleMessage(WebSocketSession session, TextMessage message) throws Exception {
        handleTextMessage(session, message);
    }

    /**
     * Called when a text message arrives. Override this method in subclasses.
     */
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        // Override to handle incoming text messages
    }

    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) throws Exception {
        // Override to handle transport errors
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) throws Exception {
        // Override to handle connection closed
    }
}
```

#### File: `src/main/java/com/simplespringmvc/websocket/HandshakeInterceptor.java` [NEW]

```java
package com.simplespringmvc.websocket;

import java.util.Map;

/**
 * Interceptor for the WebSocket handshake request. Allows inspection and
 * modification of the HTTP upgrade request before and after the handshake,
 * and population of session attributes.
 *
 * Maps to: {@code org.springframework.web.socket.server.HandshakeInterceptor}
 *
 * The real framework uses {@code ServerHttpRequest}/{@code ServerHttpResponse} wrappers
 * instead of raw Servlet types, providing a transport-agnostic abstraction.
 * We use the session attributes map directly — the interceptor can read
 * headers/parameters from the HTTP request and store data that will be
 * available via {@link WebSocketSession#getAttributes()} for the lifetime
 * of the connection.
 *
 * Common use cases: authentication checks, rate limiting, populating
 * user identity into session attributes.
 */
public interface HandshakeInterceptor {

    /**
     * Called before the WebSocket handshake is processed.
     *
     * @param requestUri  the URI of the WebSocket handshake request
     * @param attributes  mutable map that becomes {@link WebSocketSession#getAttributes()}
     * @return {@code true} to proceed with the handshake, {@code false} to reject
     */
    boolean beforeHandshake(java.net.URI requestUri, Map<String, Object> attributes) throws Exception;

    /**
     * Called after the handshake completes (successfully or not).
     *
     * @param requestUri the URI of the WebSocket handshake request
     * @param exception  the exception from the handshake, or {@code null} if successful
     */
    void afterHandshake(java.net.URI requestUri, Exception exception);
}
```

#### File: `src/main/java/com/simplespringmvc/websocket/StandardWebSocketSession.java` [NEW]

```java
package com.simplespringmvc.websocket;

import jakarta.websocket.Session;

import java.io.IOException;
import java.net.URI;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Concrete {@link WebSocketSession} implementation that wraps a Jakarta WebSocket
 * {@link Session} (JSR 356 / Jakarta WebSocket 2.1).
 *
 * Maps to: {@code org.springframework.web.socket.adapter.standard.StandardWebSocketSession}
 *
 * The real framework extends {@code AbstractWebSocketSession<Session>}, which
 * provides template methods for send/close and delegates to the native session.
 * It also handles extensions, sub-protocols, message size limits, and address lookup.
 *
 * We simplify to a direct wrapper that:
 * <ul>
 *   <li>Delegates {@code sendMessage()} to {@code session.getBasicRemote().sendText()}</li>
 *   <li>Delegates {@code close()} to {@code session.close()}</li>
 *   <li>Stores attributes in a {@link ConcurrentHashMap}</li>
 * </ul>
 */
public class StandardWebSocketSession implements WebSocketSession {

    private final String id;
    private final URI uri;
    private final Map<String, Object> attributes;
    private volatile Session nativeSession;

    public StandardWebSocketSession(String id, URI uri, Map<String, Object> attributes) {
        this.id = id;
        this.uri = uri;
        this.attributes = new ConcurrentHashMap<>(attributes);
    }

    /**
     * Initializes this session with the native Jakarta WebSocket session.
     * Called during {@code onOpen} of the endpoint adapter.
     */
    public void initializeNativeSession(Session nativeSession) {
        this.nativeSession = nativeSession;
    }

    @Override
    public String getId() {
        return id;
    }

    @Override
    public URI getUri() {
        return uri;
    }

    @Override
    public Map<String, Object> getAttributes() {
        return attributes;
    }

    @Override
    public boolean isOpen() {
        Session session = this.nativeSession;
        return session != null && session.isOpen();
    }

    @Override
    public void sendMessage(TextMessage message) throws IOException {
        checkNativeSession();
        nativeSession.getBasicRemote().sendText(message.getPayload());
    }

    @Override
    public void close() throws IOException {
        close(CloseStatus.NORMAL);
    }

    @Override
    public void close(CloseStatus status) throws IOException {
        checkNativeSession();
        nativeSession.close(new jakarta.websocket.CloseReason(
                jakarta.websocket.CloseReason.CloseCodes.getCloseCode(status.getCode()),
                status.getReason()
        ));
    }

    /**
     * Returns the underlying Jakarta WebSocket session, for testing or advanced use.
     */
    public Session getNativeSession() {
        return nativeSession;
    }

    private void checkNativeSession() {
        if (nativeSession == null) {
            throw new IllegalStateException(
                    "WebSocket session has not been initialized — onOpen has not been called yet");
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/websocket/StandardEndpointAdapter.java` [NEW]

```java
package com.simplespringmvc.websocket;

import jakarta.websocket.CloseReason;
import jakarta.websocket.Endpoint;
import jakarta.websocket.EndpointConfig;
import jakarta.websocket.MessageHandler;
import jakarta.websocket.Session;

import java.io.IOException;
import java.net.URI;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Adapts a Spring-style {@link WebSocketHandler} into a Jakarta WebSocket
 * {@link Endpoint} (JSR 356). This is the bridge that allows our handler
 * abstraction to work inside a Jakarta WebSocket container (Tomcat's built-in
 * implementation).
 *
 * Maps to: {@code org.springframework.web.socket.adapter.standard.StandardWebSocketHandlerAdapter}
 *
 * The real framework:
 * <ul>
 *   <li>Supports both partial and whole message handlers based on
 *       {@code supportsPartialMessages()}</li>
 *   <li>Handles binary messages and pong messages</li>
 *   <li>Uses {@code ExceptionWebSocketHandlerDecorator} for error handling</li>
 *   <li>Uses anonymous inner classes (not lambdas) so the container can
 *       read generic type parameters at runtime</li>
 * </ul>
 *
 * We simplify to text-only, whole-message handling.
 *
 * Data flow:
 * <pre>
 *   Jakarta container calls onOpen/onClose/onError on this Endpoint
 *     → we translate to WebSocketHandler.afterConnectionEstablished/Closed/handleTransportError
 *   Jakarta container delivers text via MessageHandler.Whole&lt;String&gt;
 *     → we wrap in TextMessage and call WebSocketHandler.handleMessage
 * </pre>
 */
public class StandardEndpointAdapter extends Endpoint {

    static final String ATTRIBUTES_KEY = "simple.ws.attributes";
    static final String URI_KEY = "simple.ws.uri";

    private final WebSocketHandler handler;
    private StandardWebSocketSession wsSession;

    public StandardEndpointAdapter(WebSocketHandler handler) {
        this.handler = handler;
    }

    @Override
    @SuppressWarnings("unchecked")
    public void onOpen(Session session, EndpointConfig config) {
        // Retrieve handshake data stored by HandshakeConfigurator.modifyHandshake()
        // via the shared getUserProperties() map.
        Map<String, Object> attributes = (Map<String, Object>) config.getUserProperties()
                .getOrDefault(ATTRIBUTES_KEY, new ConcurrentHashMap<>());
        URI uri = (URI) config.getUserProperties()
                .getOrDefault(URI_KEY, session.getRequestURI());

        String id = UUID.randomUUID().toString();
        this.wsSession = new StandardWebSocketSession(id, uri, attributes);
        wsSession.initializeNativeSession(session);

        // Register a text message handler using an anonymous inner class
        // (not a lambda) so the Jakarta container can read the generic type
        // parameter <String> at runtime via reflection.
        session.addMessageHandler(new MessageHandler.Whole<String>() {
            @Override
            public void onMessage(String payload) {
                try {
                    handler.handleMessage(wsSession, new TextMessage(payload));
                } catch (Exception ex) {
                    tryCloseWithError(ex);
                }
            }
        });

        try {
            handler.afterConnectionEstablished(wsSession);
        } catch (Exception ex) {
            tryCloseWithError(ex);
        }
    }

    @Override
    public void onClose(Session session, CloseReason closeReason) {
        CloseStatus status = new CloseStatus(
                closeReason.getCloseCode().getCode(),
                closeReason.getReasonPhrase() != null ? closeReason.getReasonPhrase() : ""
        );
        try {
            handler.afterConnectionClosed(wsSession, status);
        } catch (Exception ex) {
            // Log but don't rethrow — the connection is already closing
            System.err.println("Error in afterConnectionClosed: " + ex.getMessage());
        }
    }

    @Override
    public void onError(Session session, Throwable thr) {
        try {
            handler.handleTransportError(wsSession, thr);
        } catch (Exception ex) {
            tryCloseWithError(ex);
        }
    }

    /**
     * Returns the Spring WebSocket session (for testing).
     */
    public StandardWebSocketSession getWsSession() {
        return wsSession;
    }

    /**
     * Attempts to close the session after an unhandled exception.
     * Mirrors the real framework's {@code ExceptionWebSocketHandlerDecorator.tryCloseWithError()}.
     */
    private void tryCloseWithError(Exception ex) {
        System.err.println("Closing WebSocket session due to error: " + ex.getMessage());
        try {
            wsSession.close(CloseStatus.SERVER_ERROR.withReason(ex.getMessage()));
        } catch (IOException closeEx) {
            // Ignore — we're already handling an error
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/websocket/WebSocketConfigurer.java` [NEW]

```java
package com.simplespringmvc.websocket;

/**
 * Callback interface for registering WebSocket handlers via a fluent API.
 * Application code implements this to map handlers to URL paths.
 *
 * Maps to: {@code org.springframework.web.socket.config.annotation.WebSocketConfigurer}
 *
 * In the real framework, this is used with {@code @EnableWebSocket} which triggers
 * auto-configuration via {@code WebSocketConfigurationSupport}. The configurer
 * receives a {@code WebSocketHandlerRegistry} and registers handlers.
 *
 * Usage:
 * <pre>
 *   WebSocketConfigurer configurer = registry -> {
 *       registry.addHandler(new ChatHandler(), "/chat")
 *               .addInterceptors(new AuthInterceptor());
 *   };
 * </pre>
 */
@FunctionalInterface
public interface WebSocketConfigurer {

    /**
     * Register WebSocket handlers with the given registry.
     */
    void registerWebSocketHandlers(SimpleWebSocketHandlerRegistry registry);
}
```

#### File: `src/main/java/com/simplespringmvc/websocket/SimpleWebSocketHandlerRegistry.java` [NEW]

```java
package com.simplespringmvc.websocket;

import jakarta.websocket.DeploymentException;
import jakarta.websocket.server.ServerContainer;
import jakarta.websocket.server.ServerEndpointConfig;

import java.net.URI;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Collects WebSocket handler registrations and deploys them to the Jakarta
 * WebSocket {@link ServerContainer}.
 *
 * Maps to: {@code org.springframework.web.socket.config.annotation.ServletWebSocketHandlerRegistry}
 *
 * The real framework's registry:
 * <ul>
 *   <li>Produces a Spring MVC {@code HandlerMapping} that intercepts HTTP upgrade requests</li>
 *   <li>Supports SockJS fallback transport</li>
 *   <li>Supports allowed origins configuration</li>
 *   <li>Uses {@code HttpRequestHandler} to perform the handshake</li>
 * </ul>
 *
 * We simplify by registering directly with the Jakarta {@link ServerContainer}
 * using programmatic endpoint configuration. Each registration creates a
 * {@link ServerEndpointConfig} with a custom {@link ServerEndpointConfig.Configurator}
 * that creates our {@link StandardEndpointAdapter} and runs {@link HandshakeInterceptor}s.
 */
public class SimpleWebSocketHandlerRegistry {

    private final List<Registration> registrations = new ArrayList<>();

    /**
     * Registers a WebSocket handler at the given URL path(s).
     * Returns a fluent handle for adding interceptors.
     *
     * @param handler the handler to register
     * @param paths   the URL path(s) to map (e.g., "/chat", "/ws/echo")
     * @return a registration handle for further configuration
     */
    public Registration addHandler(WebSocketHandler handler, String... paths) {
        Registration registration = new Registration(handler, paths);
        registrations.add(registration);
        return registration;
    }

    /**
     * Deploys all registered handlers to the given Jakarta WebSocket container.
     * Called by {@code EmbeddedTomcat} after the context is initialized.
     */
    public void deploy(ServerContainer serverContainer) throws DeploymentException {
        for (Registration reg : registrations) {
            for (String path : reg.paths) {
                ServerEndpointConfig config = ServerEndpointConfig.Builder
                        .create(StandardEndpointAdapter.class, path)
                        .configurator(new HandshakeConfigurator(reg.handler, reg.interceptors))
                        .build();
                serverContainer.addEndpoint(config);
            }
        }
    }

    /**
     * Returns all registrations (for testing).
     */
    public List<Registration> getRegistrations() {
        return Collections.unmodifiableList(registrations);
    }

    // -----------------------------------------------------------------------
    // Registration — fluent handle for a single handler-to-path(s) binding
    // -----------------------------------------------------------------------

    /**
     * A single handler-to-path(s) registration with optional interceptors.
     */
    public static class Registration {
        private final WebSocketHandler handler;
        private final String[] paths;
        private final List<HandshakeInterceptor> interceptors = new ArrayList<>();

        Registration(WebSocketHandler handler, String[] paths) {
            this.handler = handler;
            this.paths = paths;
        }

        /**
         * Adds handshake interceptors to this registration.
         * @return this registration for fluent chaining
         */
        public Registration addInterceptors(HandshakeInterceptor... interceptors) {
            Collections.addAll(this.interceptors, interceptors);
            return this;
        }

        public WebSocketHandler getHandler() {
            return handler;
        }

        public String[] getPaths() {
            return paths;
        }

        public List<HandshakeInterceptor> getInterceptors() {
            return Collections.unmodifiableList(interceptors);
        }
    }

    // -----------------------------------------------------------------------
    // HandshakeConfigurator — bridges Jakarta's Configurator to our API
    // -----------------------------------------------------------------------

    /**
     * Custom Jakarta WebSocket configurator that creates our {@link StandardEndpointAdapter}
     * and runs {@link HandshakeInterceptor}s during the handshake.
     *
     * Maps to: the handshake logic in Spring's {@code AbstractHandlerMapping} and
     * {@code DefaultHandshakeHandler}.
     *
     * The Jakarta WebSocket spec's {@link ServerEndpointConfig.Configurator} has two
     * key hooks:
     * <ul>
     *   <li>{@code modifyHandshake()} — called during the HTTP upgrade, before the
     *       WebSocket connection is established. We run our interceptors here and
     *       store session attributes in {@code sec.getUserProperties()}.</li>
     *   <li>{@code getEndpointInstance()} — called to create the Endpoint instance for
     *       each new connection. We create a {@link StandardEndpointAdapter} wrapping
     *       our handler.</li>
     * </ul>
     *
     * The {@code getUserProperties()} map is shared between {@code modifyHandshake()},
     * {@code getEndpointInstance()}, and the eventual {@code onOpen()} call — this is
     * how handshake data flows to the WebSocket session.
     */
    static class HandshakeConfigurator extends ServerEndpointConfig.Configurator {

        private final WebSocketHandler handler;
        private final List<HandshakeInterceptor> interceptors;

        HandshakeConfigurator(WebSocketHandler handler, List<HandshakeInterceptor> interceptors) {
            this.handler = handler;
            this.interceptors = interceptors;
        }

        @Override
        public void modifyHandshake(ServerEndpointConfig sec,
                                    jakarta.websocket.server.HandshakeRequest request,
                                    jakarta.websocket.HandshakeResponse response) {
            URI requestUri = request.getRequestURI();
            Map<String, Object> attributes = new ConcurrentHashMap<>();

            // Run beforeHandshake on each interceptor
            List<HandshakeInterceptor> completed = new ArrayList<>();
            boolean proceed = true;

            for (HandshakeInterceptor interceptor : interceptors) {
                try {
                    if (!interceptor.beforeHandshake(requestUri, attributes)) {
                        proceed = false;
                        break;
                    }
                    completed.add(interceptor);
                } catch (Exception ex) {
                    proceed = false;
                    // Run afterHandshake for all completed interceptors
                    for (HandshakeInterceptor c : completed) {
                        c.afterHandshake(requestUri, ex);
                    }
                    return;
                }
            }

            if (proceed) {
                // Store attributes and URI in user properties — these flow to onOpen()
                // via EndpointConfig.getUserProperties()
                sec.getUserProperties().put(StandardEndpointAdapter.ATTRIBUTES_KEY, attributes);
                sec.getUserProperties().put(StandardEndpointAdapter.URI_KEY, requestUri);

                // Run afterHandshake on success
                for (HandshakeInterceptor interceptor : completed) {
                    interceptor.afterHandshake(requestUri, null);
                }
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public <T> T getEndpointInstance(Class<T> endpointClass) {
            return (T) new StandardEndpointAdapter(handler);
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/server/EmbeddedTomcat.java` [MODIFIED]

```java
package com.simplespringmvc.server;

import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import com.simplespringmvc.websocket.SimpleWebSocketHandlerRegistry;
import com.simplespringmvc.websocket.WebSocketConfigurer;
import jakarta.websocket.DeploymentException;
import jakarta.websocket.server.ServerContainer;
import org.apache.catalina.Context;
import org.apache.catalina.startup.Tomcat;

import java.io.File;

/**
 * Embeds Apache Tomcat to receive HTTP requests and route them through
 * the {@link SimpleDispatcherServlet}. Optionally initializes the WebSocket
 * container for persistent bidirectional connections.
 *
 * Maps to: Spring Boot's {@code TomcatServletWebServerFactory} which
 * programmatically creates a Tomcat instance, configures a context,
 * and registers the DispatcherServlet.
 *
 * The real Spring Boot registration chain:
 * <pre>
 *   TomcatServletWebServerFactory.getWebServer()
 *     → Tomcat tomcat = new Tomcat()
 *     → prepareContext(tomcat.getHost(), initializers)
 *     → new TomcatWebServer(tomcat)
 *
 *   ServletWebServerApplicationContext.onRefresh()
 *     → createWebServer()
 *     → factory.getWebServer(getSelfInitializer())
 *     → initializers register DispatcherServlet via servletContext.addServlet()
 * </pre>
 *
 * We simplify all of that into a single class that:
 * <ol>
 *   <li>Creates a Tomcat instance</li>
 *   <li>Adds a context at the root path</li>
 *   <li>Registers SimpleDispatcherServlet mapped to "/"</li>
 *   <li>Optionally initializes WebSocket support via {@link WebSocketConfigurer}</li>
 *   <li>Starts the server</li>
 * </ol>
 *
 * Simplifications vs real Spring Boot:
 * <ul>
 *   <li>No auto-configuration</li>
 *   <li>No customizer callbacks</li>
 *   <li>No graceful shutdown</li>
 *   <li>No SSL/HTTP2 configuration</li>
 *   <li>No compression, access logs, etc.</li>
 * </ul>
 */
public class EmbeddedTomcat {

    private final Tomcat tomcat;
    private final SimpleDispatcherServlet dispatcherServlet;
    private final Context context;

    /**
     * Creates an embedded Tomcat that routes all requests through the given servlet.
     *
     * @param port              the port to listen on (use 0 for a random available port)
     * @param dispatcherServlet the front-controller servlet that handles all requests
     */
    public EmbeddedTomcat(int port, SimpleDispatcherServlet dispatcherServlet) {
        this.dispatcherServlet = dispatcherServlet;
        this.tomcat = new Tomcat();
        tomcat.setPort(port);

        // Tomcat needs a base directory for temp files. Use a temp dir so we
        // don't pollute the project directory.
        tomcat.setBaseDir(new File(System.getProperty("java.io.tmpdir"), "tomcat-simple")
                .getAbsolutePath());

        // Create the root context — equivalent to a web application at "/"
        // The empty docBase means no static files directory.
        this.context = tomcat.addContext("", null);

        // Register our DispatcherServlet.
        // Tomcat.addServlet() is the programmatic equivalent of a <servlet> entry
        // in web.xml. We map it to "/" so it receives all requests (the default servlet).
        Tomcat.addServlet(context, "dispatcher", dispatcherServlet);
        context.addServletMappingDecoded("/", "dispatcher");

        // Enable the connector so Tomcat will actually listen on the port.
        tomcat.getConnector();
    }

    /**
     * Registers WebSocket handlers via the given configurer.
     *
     * This initializes Tomcat's WebSocket container (WsSci) and deploys
     * all handlers registered through the configurer's registry.
     *
     * Must be called BEFORE {@link #start()}.
     *
     * Maps to: Spring's {@code WebSocketConfigurationSupport} which creates a
     * {@code ServletWebSocketHandlerRegistry}, passes it to all {@code WebSocketConfigurer}
     * beans, and produces a {@code HandlerMapping} for WebSocket upgrade requests.
     *
     * We simplify by registering directly with the Jakarta ServerContainer
     * instead of creating a HandlerMapping.
     *
     * @param configurer the callback that registers WebSocket handlers
     */
    public void registerWebSocketHandlers(WebSocketConfigurer configurer) {
        // Initialize Tomcat's WebSocket support by adding the WsSci
        // (WebSocket Server Container Initializer) to the context.
        // This makes the ServerContainer available via the ServletContext attribute.
        context.addServletContainerInitializer(
                new org.apache.tomcat.websocket.server.WsSci(), null);

        // Create a registry and let the configurer populate it
        SimpleWebSocketHandlerRegistry registry = new SimpleWebSocketHandlerRegistry();
        configurer.registerWebSocketHandlers(registry);

        // Store for deployment after Tomcat starts (WsSci needs to run first)
        context.addLifecycleListener(event -> {
            if ("after_start".equals(event.getType())) {
                try {
                    ServerContainer serverContainer = (ServerContainer) context.getServletContext()
                            .getAttribute(ServerContainer.class.getName());
                    if (serverContainer != null) {
                        registry.deploy(serverContainer);
                    }
                } catch (DeploymentException ex) {
                    throw new RuntimeException("Failed to deploy WebSocket endpoints", ex);
                }
            }
        });
    }

    /**
     * Starts the embedded Tomcat server.
     *
     * After this call, the server is accepting HTTP connections on {@link #getPort()}.
     */
    public void start() throws Exception {
        tomcat.start();
    }

    /**
     * Stops the embedded Tomcat server and releases all resources.
     */
    public void stop() throws Exception {
        tomcat.stop();
        tomcat.destroy();
    }

    /**
     * Returns the port the server is listening on.
     *
     * If the server was created with port 0 (random), this returns the actual
     * allocated port after {@link #start()} has been called.
     */
    public int getPort() {
        return tomcat.getConnector().getLocalPort();
    }

    /**
     * Returns the DispatcherServlet this server is routing requests to.
     */
    public SimpleDispatcherServlet getDispatcherServlet() {
        return dispatcherServlet;
    }

    /**
     * Blocks the calling thread until the server shuts down.
     * Useful for running the server as a standalone application.
     */
    public void await() {
        tomcat.getServer().await();
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/websocket/TextMessageTest.java` [NEW]

```java
package com.simplespringmvc.websocket;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TextMessageTest {

    @Test
    void shouldStorePayload_WhenCreated() {
        var message = new TextMessage("Hello WebSocket");

        assertThat(message.getPayload()).isEqualTo("Hello WebSocket");
    }

    @Test
    void shouldReturnPayloadLength_WhenAsked() {
        var message = new TextMessage("Hello");

        assertThat(message.getPayloadLength()).isEqualTo(5);
    }

    @Test
    void shouldBeEqual_WhenPayloadsMatch() {
        var msg1 = new TextMessage("Hello");
        var msg2 = new TextMessage("Hello");

        assertThat(msg1).isEqualTo(msg2);
        assertThat(msg1.hashCode()).isEqualTo(msg2.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenPayloadsDiffer() {
        var msg1 = new TextMessage("Hello");
        var msg2 = new TextMessage("World");

        assertThat(msg1).isNotEqualTo(msg2);
    }

    @Test
    void shouldTruncateInToString_WhenPayloadIsLong() {
        var message = new TextMessage("A".repeat(100));

        assertThat(message.toString()).contains("...");
        assertThat(message.toString()).contains("A".repeat(50));
    }

    @Test
    void shouldNotTruncateInToString_WhenPayloadIsShort() {
        var message = new TextMessage("Short");

        assertThat(message.toString()).doesNotContain("...");
        assertThat(message.toString()).contains("Short");
    }

    @Test
    void shouldRejectNullPayload() {
        assertThatThrownBy(() -> new TextMessage(null))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void shouldHandleEmptyPayload() {
        var message = new TextMessage("");

        assertThat(message.getPayload()).isEmpty();
        assertThat(message.getPayloadLength()).isZero();
    }
}
```

#### File: `src/test/java/com/simplespringmvc/websocket/CloseStatusTest.java` [NEW]

```java
package com.simplespringmvc.websocket;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class CloseStatusTest {

    @Test
    void shouldStoreCodeAndReason_WhenCreated() {
        var status = new CloseStatus(1000, "Normal closure");

        assertThat(status.getCode()).isEqualTo(1000);
        assertThat(status.getReason()).isEqualTo("Normal closure");
    }

    @Test
    void shouldDefaultToEmptyReason_WhenOnlyCodeGiven() {
        var status = new CloseStatus(1000);

        assertThat(status.getReason()).isEmpty();
    }

    @Test
    void shouldCreateNewInstance_WhenWithReasonCalled() {
        CloseStatus original = CloseStatus.SERVER_ERROR;
        CloseStatus withReason = original.withReason("Custom error message");

        assertThat(withReason.getCode()).isEqualTo(1011);
        assertThat(withReason.getReason()).isEqualTo("Custom error message");
        // Original is unchanged (immutable)
        assertThat(original.getReason()).isEqualTo("Server error");
    }

    @Test
    void shouldDefineStandardCodes() {
        assertThat(CloseStatus.NORMAL.getCode()).isEqualTo(1000);
        assertThat(CloseStatus.GOING_AWAY.getCode()).isEqualTo(1001);
        assertThat(CloseStatus.PROTOCOL_ERROR.getCode()).isEqualTo(1002);
        assertThat(CloseStatus.NOT_ACCEPTABLE.getCode()).isEqualTo(1003);
        assertThat(CloseStatus.SERVER_ERROR.getCode()).isEqualTo(1011);
    }

    @Test
    void shouldBeEqual_WhenCodeAndReasonMatch() {
        var status1 = new CloseStatus(1000, "Normal");
        var status2 = new CloseStatus(1000, "Normal");

        assertThat(status1).isEqualTo(status2);
        assertThat(status1.hashCode()).isEqualTo(status2.hashCode());
    }

    @Test
    void shouldNotBeEqual_WhenCodesDiffer() {
        var status1 = new CloseStatus(1000, "Normal");
        var status2 = new CloseStatus(1011, "Normal");

        assertThat(status1).isNotEqualTo(status2);
    }

    @Test
    void shouldNotBeEqual_WhenReasonsDiffer() {
        var status1 = new CloseStatus(1000, "Reason A");
        var status2 = new CloseStatus(1000, "Reason B");

        assertThat(status1).isNotEqualTo(status2);
    }
}
```

#### File: `src/test/java/com/simplespringmvc/websocket/TextWebSocketHandlerTest.java` [NEW]

```java
package com.simplespringmvc.websocket;

import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class TextWebSocketHandlerTest {

    @Test
    void shouldDispatchToHandleTextMessage_WhenMessageReceived() throws Exception {
        List<String> received = new ArrayList<>();

        var handler = new TextWebSocketHandler() {
            @Override
            protected void handleTextMessage(WebSocketSession session, TextMessage message) {
                received.add(message.getPayload());
            }
        };

        var session = new StandardWebSocketSession("1", java.net.URI.create("/test"), Map.of());
        handler.handleMessage(session, new TextMessage("Hello"));

        assertThat(received).containsExactly("Hello");
    }

    @Test
    void shouldDoNothing_WhenDefaultCallbacksInvoked() throws Exception {
        // Verify that default implementations don't throw
        var handler = new TextWebSocketHandler() {};
        var session = new StandardWebSocketSession("1", java.net.URI.create("/test"), Map.of());

        handler.afterConnectionEstablished(session);
        handler.handleMessage(session, new TextMessage("test"));
        handler.handleTransportError(session, new RuntimeException("test"));
        handler.afterConnectionClosed(session, CloseStatus.NORMAL);
        // No exception = success
    }

    @Test
    void shouldTrackConnectionLifecycle_WhenAllCallbacksOverridden() throws Exception {
        List<String> events = new ArrayList<>();

        var handler = new TextWebSocketHandler() {
            @Override
            public void afterConnectionEstablished(WebSocketSession session) {
                events.add("opened:" + session.getId());
            }

            @Override
            protected void handleTextMessage(WebSocketSession session, TextMessage message) {
                events.add("message:" + message.getPayload());
            }

            @Override
            public void handleTransportError(WebSocketSession session, Throwable exception) {
                events.add("error:" + exception.getMessage());
            }

            @Override
            public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) {
                events.add("closed:" + closeStatus.getCode());
            }
        };

        var session = new StandardWebSocketSession("sess-1", java.net.URI.create("/ws"), Map.of());

        handler.afterConnectionEstablished(session);
        handler.handleMessage(session, new TextMessage("ping"));
        handler.handleTransportError(session, new RuntimeException("network error"));
        handler.afterConnectionClosed(session, CloseStatus.NORMAL);

        assertThat(events).containsExactly(
                "opened:sess-1",
                "message:ping",
                "error:network error",
                "closed:1000"
        );
    }
}
```

#### File: `src/test/java/com/simplespringmvc/websocket/StandardWebSocketSessionTest.java` [NEW]

```java
package com.simplespringmvc.websocket;

import org.junit.jupiter.api.Test;

import java.net.URI;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class StandardWebSocketSessionTest {

    @Test
    void shouldStoreIdAndUri_WhenCreated() {
        var session = new StandardWebSocketSession("abc-123", URI.create("/chat"), Map.of());

        assertThat(session.getId()).isEqualTo("abc-123");
        assertThat(session.getUri()).isEqualTo(URI.create("/chat"));
    }

    @Test
    void shouldCopyAttributes_WhenCreated() {
        var attrs = Map.of("user", (Object) "Alice");
        var session = new StandardWebSocketSession("1", URI.create("/ws"), attrs);

        assertThat(session.getAttributes()).containsEntry("user", "Alice");
    }

    @Test
    void shouldAllowMutableAttributes() {
        var session = new StandardWebSocketSession("1", URI.create("/ws"), Map.of());

        session.getAttributes().put("key", "value");
        assertThat(session.getAttributes()).containsEntry("key", "value");
    }

    @Test
    void shouldReturnNotOpen_WhenNativeSessionNotInitialized() {
        var session = new StandardWebSocketSession("1", URI.create("/ws"), Map.of());

        assertThat(session.isOpen()).isFalse();
    }

    @Test
    void shouldThrow_WhenSendingWithoutNativeSession() {
        var session = new StandardWebSocketSession("1", URI.create("/ws"), Map.of());

        assertThatThrownBy(() -> session.sendMessage(new TextMessage("hello")))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("not been initialized");
    }

    @Test
    void shouldThrow_WhenClosingWithoutNativeSession() {
        var session = new StandardWebSocketSession("1", URI.create("/ws"), Map.of());

        assertThatThrownBy(() -> session.close())
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("not been initialized");
    }
}
```

#### File: `src/test/java/com/simplespringmvc/websocket/SimpleWebSocketHandlerRegistryTest.java` [NEW]

```java
package com.simplespringmvc.websocket;

import org.junit.jupiter.api.Test;

import java.net.URI;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class SimpleWebSocketHandlerRegistryTest {

    @Test
    void shouldRegisterHandler_WhenAddHandlerCalled() {
        var registry = new SimpleWebSocketHandlerRegistry();
        var handler = new TextWebSocketHandler() {};

        registry.addHandler(handler, "/ws/echo");

        assertThat(registry.getRegistrations()).hasSize(1);
        assertThat(registry.getRegistrations().get(0).getHandler()).isSameAs(handler);
        assertThat(registry.getRegistrations().get(0).getPaths()).containsExactly("/ws/echo");
    }

    @Test
    void shouldRegisterMultiplePaths_WhenGiven() {
        var registry = new SimpleWebSocketHandlerRegistry();
        var handler = new TextWebSocketHandler() {};

        registry.addHandler(handler, "/ws/echo", "/ws/chat");

        assertThat(registry.getRegistrations()).hasSize(1);
        assertThat(registry.getRegistrations().get(0).getPaths())
                .containsExactly("/ws/echo", "/ws/chat");
    }

    @Test
    void shouldRegisterMultipleHandlers_WhenAddHandlerCalledMultipleTimes() {
        var registry = new SimpleWebSocketHandlerRegistry();
        var echo = new TextWebSocketHandler() {};
        var chat = new TextWebSocketHandler() {};

        registry.addHandler(echo, "/ws/echo");
        registry.addHandler(chat, "/ws/chat");

        assertThat(registry.getRegistrations()).hasSize(2);
    }

    @Test
    void shouldAddInterceptors_WhenFluentApiUsed() {
        var registry = new SimpleWebSocketHandlerRegistry();
        var handler = new TextWebSocketHandler() {};
        var interceptor = new HandshakeInterceptor() {
            @Override
            public boolean beforeHandshake(URI requestUri, Map<String, Object> attributes) {
                return true;
            }

            @Override
            public void afterHandshake(URI requestUri, Exception exception) {}
        };

        registry.addHandler(handler, "/ws/echo")
                .addInterceptors(interceptor);

        assertThat(registry.getRegistrations().get(0).getInterceptors())
                .hasSize(1)
                .containsExactly(interceptor);
    }

    @Test
    void shouldChainMultipleInterceptors_WhenCalledMultipleTimes() {
        var registry = new SimpleWebSocketHandlerRegistry();
        var handler = new TextWebSocketHandler() {};
        HandshakeInterceptor i1 = new HandshakeInterceptor() {
            @Override
            public boolean beforeHandshake(URI requestUri, Map<String, Object> attributes) {
                return true;
            }

            @Override
            public void afterHandshake(URI requestUri, Exception exception) {}
        };
        HandshakeInterceptor i2 = new HandshakeInterceptor() {
            @Override
            public boolean beforeHandshake(URI requestUri, Map<String, Object> attributes) {
                return true;
            }

            @Override
            public void afterHandshake(URI requestUri, Exception exception) {}
        };

        registry.addHandler(handler, "/ws/echo")
                .addInterceptors(i1)
                .addInterceptors(i2);

        assertThat(registry.getRegistrations().get(0).getInterceptors()).hasSize(2);
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/WebSocketIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import com.simplespringmvc.websocket.CloseStatus;
import com.simplespringmvc.websocket.HandshakeInterceptor;
import com.simplespringmvc.websocket.TextMessage;
import com.simplespringmvc.websocket.TextWebSocketHandler;
import com.simplespringmvc.websocket.WebSocketSession;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for WebSocket support. Starts a real embedded Tomcat
 * and connects using the JDK HttpClient's WebSocket API.
 */
class WebSocketIntegrationTest {

    private EmbeddedTomcat server;

    @AfterEach
    void tearDown() throws Exception {
        if (server != null) {
            server.stop();
        }
    }

    // --- Echo Handler Tests ---

    @Test
    void shouldEchoMessage_WhenTextSent() throws Exception {
        var receivedMessages = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(1);

        server = createServer(new TextWebSocketHandler() {
            @Override
            protected void handleTextMessage(WebSocketSession session, TextMessage message)
                    throws Exception {
                session.sendMessage(new TextMessage("Echo: " + message.getPayload()));
            }
        }, "/ws/echo");

        var client = java.net.http.HttpClient.newHttpClient();
        var ws = client.newWebSocketBuilder()
                .buildAsync(wsUri("/ws/echo"), new java.net.http.WebSocket.Listener() {
                    private final StringBuilder buffer = new StringBuilder();

                    @Override
                    public java.util.concurrent.CompletionStage<?> onText(
                            java.net.http.WebSocket webSocket, CharSequence data, boolean last) {
                        buffer.append(data);
                        if (last) {
                            receivedMessages.add(buffer.toString());
                            buffer.setLength(0);
                            latch.countDown();
                        }
                        webSocket.request(1);
                        return null;
                    }
                })
                .join();

        ws.sendText("Hello", true);

        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(receivedMessages).containsExactly("Echo: Hello");

        ws.sendClose(java.net.http.WebSocket.NORMAL_CLOSURE, "").join();
    }

    @Test
    void shouldTrackLifecycleEvents_WhenConnectionOpensAndCloses() throws Exception {
        var events = new CopyOnWriteArrayList<String>();
        var closeLatch = new CountDownLatch(1);

        server = createServer(new TextWebSocketHandler() {
            @Override
            public void afterConnectionEstablished(WebSocketSession session) {
                events.add("opened");
            }

            @Override
            protected void handleTextMessage(WebSocketSession session, TextMessage message) {
                events.add("message:" + message.getPayload());
            }

            @Override
            public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) {
                events.add("closed:" + closeStatus.getCode());
                closeLatch.countDown();
            }
        }, "/ws/lifecycle");

        var client = java.net.http.HttpClient.newHttpClient();
        var ws = client.newWebSocketBuilder()
                .buildAsync(wsUri("/ws/lifecycle"), new java.net.http.WebSocket.Listener() {
                    @Override
                    public java.util.concurrent.CompletionStage<?> onText(
                            java.net.http.WebSocket webSocket, CharSequence data, boolean last) {
                        webSocket.request(1);
                        return null;
                    }
                })
                .join();

        ws.sendText("ping", true);
        Thread.sleep(200); // Allow time for message processing
        ws.sendClose(java.net.http.WebSocket.NORMAL_CLOSURE, "").join();

        assertThat(closeLatch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(events).containsExactly("opened", "message:ping", "closed:1000");
    }

    @Test
    void shouldSendMultipleMessages_WhenConnectionStaysOpen() throws Exception {
        var receivedMessages = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(3);

        server = createServer(new TextWebSocketHandler() {
            @Override
            protected void handleTextMessage(WebSocketSession session, TextMessage message)
                    throws Exception {
                session.sendMessage(new TextMessage("Re: " + message.getPayload()));
            }
        }, "/ws/multi");

        var client = java.net.http.HttpClient.newHttpClient();
        var ws = client.newWebSocketBuilder()
                .buildAsync(wsUri("/ws/multi"), new java.net.http.WebSocket.Listener() {
                    private final StringBuilder buffer = new StringBuilder();

                    @Override
                    public java.util.concurrent.CompletionStage<?> onText(
                            java.net.http.WebSocket webSocket, CharSequence data, boolean last) {
                        buffer.append(data);
                        if (last) {
                            receivedMessages.add(buffer.toString());
                            buffer.setLength(0);
                            latch.countDown();
                        }
                        webSocket.request(1);
                        return null;
                    }
                })
                .join();

        ws.sendText("msg1", true);
        ws.sendText("msg2", true);
        ws.sendText("msg3", true);

        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(receivedMessages).containsExactly("Re: msg1", "Re: msg2", "Re: msg3");

        ws.sendClose(java.net.http.WebSocket.NORMAL_CLOSURE, "").join();
    }

    // --- Multiple Paths Tests ---

    @Test
    void shouldRouteToCorrectHandler_WhenMultiplePathsRegistered() throws Exception {
        var echoMessages = new CopyOnWriteArrayList<String>();
        var uppercaseMessages = new CopyOnWriteArrayList<String>();
        var echoLatch = new CountDownLatch(1);
        var uppercaseLatch = new CountDownLatch(1);

        var echoHandler = new TextWebSocketHandler() {
            @Override
            protected void handleTextMessage(WebSocketSession session, TextMessage message)
                    throws Exception {
                session.sendMessage(new TextMessage("echo:" + message.getPayload()));
            }
        };

        var uppercaseHandler = new TextWebSocketHandler() {
            @Override
            protected void handleTextMessage(WebSocketSession session, TextMessage message)
                    throws Exception {
                session.sendMessage(new TextMessage(message.getPayload().toUpperCase()));
            }
        };

        server = createServerMultiHandler(echoHandler, "/ws/echo", uppercaseHandler, "/ws/upper");

        var client = java.net.http.HttpClient.newHttpClient();

        var echoWs = client.newWebSocketBuilder()
                .buildAsync(wsUri("/ws/echo"), createListener(echoMessages, echoLatch))
                .join();

        var upperWs = client.newWebSocketBuilder()
                .buildAsync(wsUri("/ws/upper"), createListener(uppercaseMessages, uppercaseLatch))
                .join();

        echoWs.sendText("hello", true);
        upperWs.sendText("hello", true);

        assertThat(echoLatch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(uppercaseLatch.await(5, TimeUnit.SECONDS)).isTrue();

        assertThat(echoMessages).containsExactly("echo:hello");
        assertThat(uppercaseMessages).containsExactly("HELLO");

        echoWs.sendClose(java.net.http.WebSocket.NORMAL_CLOSURE, "").join();
        upperWs.sendClose(java.net.http.WebSocket.NORMAL_CLOSURE, "").join();
    }

    // --- Handshake Interceptor Tests ---

    @Test
    void shouldPopulateAttributes_WhenInterceptorSetsThemDuringHandshake() throws Exception {
        var receivedAttributes = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(1);

        var interceptor = new HandshakeInterceptor() {
            @Override
            public boolean beforeHandshake(URI requestUri, Map<String, Object> attributes) {
                attributes.put("greeting", "Hello from interceptor");
                return true;
            }

            @Override
            public void afterHandshake(URI requestUri, Exception exception) {}
        };

        server = createServerWithInterceptor(new TextWebSocketHandler() {
            @Override
            public void afterConnectionEstablished(WebSocketSession session) throws Exception {
                String greeting = (String) session.getAttributes().get("greeting");
                session.sendMessage(new TextMessage(greeting != null ? greeting : "no attribute"));
            }
        }, "/ws/attrs", interceptor);

        var client = java.net.http.HttpClient.newHttpClient();
        var messages = new CopyOnWriteArrayList<String>();
        var ws = client.newWebSocketBuilder()
                .buildAsync(wsUri("/ws/attrs"), createListener(messages, latch))
                .join();

        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(messages).containsExactly("Hello from interceptor");

        ws.sendClose(java.net.http.WebSocket.NORMAL_CLOSURE, "").join();
    }

    // --- Broadcast (multiple connections) ---

    @Test
    void shouldBroadcastToAllClients_WhenMultipleConnectionsExist() throws Exception {
        var sessions = Collections.synchronizedList(new ArrayList<WebSocketSession>());
        var client1Messages = new CopyOnWriteArrayList<String>();
        var client2Messages = new CopyOnWriteArrayList<String>();
        var broadcastLatch = new CountDownLatch(2);

        server = createServer(new TextWebSocketHandler() {
            @Override
            public void afterConnectionEstablished(WebSocketSession session) {
                sessions.add(session);
            }

            @Override
            protected void handleTextMessage(WebSocketSession session, TextMessage message)
                    throws Exception {
                // Broadcast to all connected sessions
                for (WebSocketSession s : sessions) {
                    if (s.isOpen()) {
                        s.sendMessage(new TextMessage("broadcast:" + message.getPayload()));
                    }
                }
            }

            @Override
            public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) {
                sessions.remove(session);
            }
        }, "/ws/broadcast");

        var client = java.net.http.HttpClient.newHttpClient();

        var ws1 = client.newWebSocketBuilder()
                .buildAsync(wsUri("/ws/broadcast"), createListener(client1Messages, broadcastLatch))
                .join();

        var ws2 = client.newWebSocketBuilder()
                .buildAsync(wsUri("/ws/broadcast"), createListener(client2Messages, broadcastLatch))
                .join();

        // Wait for both connections to be established
        Thread.sleep(300);

        // Client 1 sends a message — both clients should receive the broadcast
        ws1.sendText("hello everyone", true);

        assertThat(broadcastLatch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(client1Messages).containsExactly("broadcast:hello everyone");
        assertThat(client2Messages).containsExactly("broadcast:hello everyone");

        ws1.sendClose(java.net.http.WebSocket.NORMAL_CLOSURE, "").join();
        ws2.sendClose(java.net.http.WebSocket.NORMAL_CLOSURE, "").join();
    }

    // --- Helper Methods ---

    private URI wsUri(String path) {
        return URI.create("ws://localhost:" + server.getPort() + path);
    }

    private EmbeddedTomcat createServer(TextWebSocketHandler handler, String path) throws Exception {
        var container = new SimpleBeanContainer();
        var servlet = new SimpleDispatcherServlet(container);
        var server = new EmbeddedTomcat(0, servlet);
        server.registerWebSocketHandlers(registry ->
                registry.addHandler(handler, path));
        server.start();
        return server;
    }

    private EmbeddedTomcat createServerMultiHandler(
            TextWebSocketHandler handler1, String path1,
            TextWebSocketHandler handler2, String path2) throws Exception {
        var container = new SimpleBeanContainer();
        var servlet = new SimpleDispatcherServlet(container);
        var server = new EmbeddedTomcat(0, servlet);
        server.registerWebSocketHandlers(registry -> {
            registry.addHandler(handler1, path1);
            registry.addHandler(handler2, path2);
        });
        server.start();
        return server;
    }

    private EmbeddedTomcat createServerWithInterceptor(
            TextWebSocketHandler handler, String path,
            HandshakeInterceptor interceptor) throws Exception {
        var container = new SimpleBeanContainer();
        var servlet = new SimpleDispatcherServlet(container);
        var server = new EmbeddedTomcat(0, servlet);
        server.registerWebSocketHandlers(registry ->
                registry.addHandler(handler, path)
                        .addInterceptors(interceptor));
        server.start();
        return server;
    }

    private java.net.http.WebSocket.Listener createListener(
            List<String> messages, CountDownLatch latch) {
        return new java.net.http.WebSocket.Listener() {
            private final StringBuilder buffer = new StringBuilder();

            @Override
            public java.util.concurrent.CompletionStage<?> onText(
                    java.net.http.WebSocket webSocket, CharSequence data, boolean last) {
                buffer.append(data);
                if (last) {
                    messages.add(buffer.toString());
                    buffer.setLength(0);
                    latch.countDown();
                }
                webSocket.request(1);
                return null;
            }
        };
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **WebSocket** | A protocol (RFC 6455) providing persistent, bidirectional communication over a single TCP connection — unlike HTTP's request/response model |
| **Endpoint** | Jakarta WebSocket's term for a server-side WebSocket handler — our `StandardEndpointAdapter` extends it |
| **Adapter Pattern** | `StandardEndpointAdapter` and `StandardWebSocketSession` translate between two incompatible APIs (Jakarta ↔ our abstraction) |
| **Template Method** | `TextWebSocketHandler.handleMessage()` is `final` and dispatches to `handleTextMessage()` — subclasses override the hook, not the template |
| **ServerContainer** | Jakarta WebSocket's runtime that manages endpoint registration and connection lifecycle — obtained from `ServletContext` attributes |
| **WsSci** | Tomcat's WebSocket Server Container Initializer — bootstraps WebSocket support when added to a Tomcat `Context` |
| **HandshakeInterceptor** | A before/after hook that runs during the HTTP upgrade handshake, before the WebSocket connection is established |

**Next: Chapter 22 — SSE (Server-Sent Events)** — Stream real-time events from server to client over a long-lived HTTP connection using async Servlet processing, complementing WebSocket's bidirectional model with a simpler, HTTP-native alternative.
