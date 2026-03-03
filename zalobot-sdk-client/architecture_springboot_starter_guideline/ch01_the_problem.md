# Chapter 1: The Problem — Manual Configuration Hell

> Before starters exist, every Spring project wires everything by hand. Let's see how painful that is.

---

## The Scenario

You want to use the ZaloBot SDK in a Spring Boot application. Without a starter, you write a `@Configuration` class manually:

```java
@Configuration
public class ZaloBotConfig {

    @Bean
    public ZaloBotClient zaloBotClient() {
        ZaloBotUrl url = new ZaloBotUrl("https", "bot-api.zaloplatforms.com", 443);
        return new DefaultZaloBotClientBuilder()    // ← but this is package-private!
                .botToken("my-secret-token")        // ← hardcoded!
                .ZaloBotUrl(url)
                .build();
    }

    @Bean
    public ContainerProperties containerProperties(UpdateListener listener) {
        ContainerProperties props = new ContainerProperties();
        props.setPollTimeout(Duration.ofSeconds(30));    // ← hardcoded!
        props.setPollInterval(Duration.ofSeconds(0));    // ← hardcoded!
        props.setShutdownTimeout(Duration.ofSeconds(10));
        props.setBackOffInterval(Duration.ofSeconds(1));
        props.setMaxBackOffInterval(Duration.ofSeconds(30));
        props.setUpdateListener(listener);
        return props;
    }

    @Bean
    public ZaloUpdateListenerContainer listenerContainer(
            ZaloBotClient client, ContainerProperties props) {
        return new ZaloUpdateListenerContainer(client, props);
    }

    // How do we start/stop the container with the Spring lifecycle?
    // We'd need a SmartLifecycle adapter too...
}
```

---

## Five Problems

### Problem 1: Hardcoded Values

The bot token, URL, timeouts — all embedded directly in Java code. To change the poll timeout from 30s to 60s, you must:
1. Edit the Java file
2. Recompile
3. Redeploy

**What we want:** `application.yml` properties that change behavior without recompilation.

```yaml
# What we WANT to write:
zalobot:
  bot-token: ${ZALO_BOT_TOKEN}
  listener:
    poll-timeout: 60s
```

### Problem 2: No Back-Away

If a user wants to customize just the `ZaloBotClient` (e.g., with a custom `JsonMapper`), they have to rewrite the **entire** configuration class. There's no way to say "use my client but auto-configure everything else."

**What we want:** `@ConditionalOnMissingBean` — if the user defines their own `ZaloBotClient`, the auto-configured one backs away. But the listener container still auto-configures using the user's client.

### Problem 3: No Conditional Activation

The configuration always runs. Even if:
- The bot token isn't set (app will crash at runtime)
- The `zalobot-listener` module isn't on the classpath (compilation error)
- The user doesn't want the listener (no way to disable it)

**What we want:** `@ConditionalOnClass`, `@ConditionalOnProperty` — configuration only activates when the right classes are present AND the right properties are set.

### Problem 4: Copy-Paste Across Projects

Every project that uses ZaloBot needs this same 40-line configuration. If the SDK API changes (e.g., a new required parameter), every project must be updated independently.

**What we want:** A single `spring-boot-starter-zalobot` dependency that brings in all the right configuration. Update the starter, and all projects benefit.

### Problem 5: No IDE Autocomplete for Properties

Without `@ConfigurationProperties`, IDEs like IntelliJ don't know what properties exist. Users have to read the SDK Javadoc to discover configuration options.

**What we want:** `@ConfigurationProperties("zalobot")` with a well-structured `ZaloBotProperties` class, so IDEs provide autocomplete for `zalobot.listener.poll-timeout`, `zalobot.client.host`, etc.

---

## The Solution: Spring Boot Starters

Spring Boot's auto-configuration mechanism solves all five problems:

| Problem | Solution | Spring Boot Mechanism |
|---|---|---|
| Hardcoded values | Externalized properties | `@ConfigurationProperties` (Chapter 2) |
| No back-away | Conditional bean creation | `@ConditionalOnMissingBean` (Chapter 4) |
| No conditional activation | Class/property guards | `@ConditionalOnClass`, `@ConditionalOnProperty` (Chapter 4) |
| Copy-paste across projects | Dependency = configuration | Starter + `.imports` discovery (Chapters 5, 9) |
| No IDE autocomplete | Typed property classes | `@ConfigurationProperties` + annotation processor (Chapter 2) |

---

## What's Next?

In Chapter 2, we start solving Problem 1 by learning how `@ConfigurationProperties` binds YAML properties to Java POJOs — turning `zalobot.listener.poll-timeout=60s` into `Duration.ofSeconds(60)` automatically.

```
Chapter 1  ←── YOU ARE HERE: "Why do we need starters?"
Chapter 2  ──► "Externalize config with @ConfigurationProperties"
```
