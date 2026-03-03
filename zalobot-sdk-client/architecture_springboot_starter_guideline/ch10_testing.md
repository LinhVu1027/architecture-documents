# Chapter 10: Testing and Verification

> Proving it works: unit tests with `ApplicationContextRunner`, integration tests, and the `--debug` flag.

---

## 1. Unit Tests with `ApplicationContextRunner`

Spring Boot provides `ApplicationContextRunner` — a lightweight way to test auto-configuration without starting a full application. It creates a real application context in memory, runs your assertions, and tears it down.

### Test 1: Token Present — Client Bean Created

```java
package dev.linhvu.zalobot.autoconfigure;

import dev.linhvu.zalobot.client.ZaloBotClient;
import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.AutoConfigurations;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import static org.assertj.core.api.Assertions.assertThat;

class ZaloBotClientAutoConfigurationTests {

    private final ApplicationContextRunner contextRunner = new ApplicationContextRunner()
            .withConfiguration(AutoConfigurations.of(ZaloBotClientAutoConfiguration.class));

    @Test
    void whenTokenIsPresent_thenClientBeanCreated() {
        this.contextRunner
                .withPropertyValues("zalobot.bot-token=test-token")
                .run(context -> {
                    assertThat(context).hasSingleBean(ZaloBotClient.class);
                    assertThat(context).hasSingleBean(ZaloBotClient.Builder.class);
                });
    }
}
```

**What's happening:**
1. `ApplicationContextRunner` creates a minimal application context
2. `withConfiguration(AutoConfigurations.of(...))` registers our auto-configuration
3. `withPropertyValues(...)` sets properties as if they came from `application.yml`
4. `.run(context -> ...)` creates the context, runs assertions, and closes it

### Test 2: Token Missing — No Beans Created

```java
@Test
void whenTokenIsMissing_thenNoBeanCreated() {
    this.contextRunner
            // no "zalobot.bot-token" property
            .run(context -> {
                assertThat(context).doesNotHaveBean(ZaloBotClient.class);
                assertThat(context).doesNotHaveBean(ZaloBotClient.Builder.class);
            });
}
```

This validates that `@ConditionalOnProperty(name = "zalobot.bot-token")` works correctly — when the token is absent, the entire auto-configuration is skipped.

### Test 3: User Override — Auto-Config Backs Away

```java
@Test
void whenUserDefinesOwnClient_thenAutoConfigBacksAway() {
    ZaloBotClient customClient = ZaloBotClient.botToken("custom-token");

    this.contextRunner
            .withPropertyValues("zalobot.bot-token=test-token")
            .withBean(ZaloBotClient.class, () -> customClient)    // user-defined bean
            .run(context -> {
                assertThat(context).hasSingleBean(ZaloBotClient.class);
                assertThat(context.getBean(ZaloBotClient.class)).isSameAs(customClient);
            });
}
```

This validates `@ConditionalOnMissingBean` — the user's bean takes precedence, and the auto-configured bean is not created.

### Test 4: Custom Properties Bound Correctly

```java
@Test
void whenCustomPropertiesSet_thenBoundCorrectly() {
    this.contextRunner
            .withPropertyValues(
                    "zalobot.bot-token=test-token",
                    "zalobot.client.host=custom-api.example.com",
                    "zalobot.client.port=8443",
                    "zalobot.listener.poll-timeout=60s",
                    "zalobot.listener.concurrency=3")
            .run(context -> {
                ZaloBotProperties props = context.getBean(ZaloBotProperties.class);
                assertThat(props.getBotToken()).isEqualTo("test-token");
                assertThat(props.getClient().getHost()).isEqualTo("custom-api.example.com");
                assertThat(props.getClient().getPort()).isEqualTo(8443);
                assertThat(props.getListener().getPollTimeout())
                        .isEqualTo(java.time.Duration.ofSeconds(60));
                assertThat(props.getListener().getConcurrency()).isEqualTo(3);
            });
}
```

---

## 2. Listener Auto-Configuration Tests

```java
class ZaloBotListenerAutoConfigurationTests {

    private final ApplicationContextRunner contextRunner = new ApplicationContextRunner()
            .withConfiguration(AutoConfigurations.of(
                    ZaloBotClientAutoConfiguration.class,
                    ZaloBotListenerAutoConfiguration.class));

    @Test
    void whenListenerBeanPresent_thenContainerCreated() {
        this.contextRunner
                .withPropertyValues("zalobot.bot-token=test-token")
                .withBean(UpdateListener.class, () -> update -> {})    // user's listener
                .run(context -> {
                    assertThat(context).hasSingleBean(UpdateListenerContainer.class);
                    assertThat(context).hasSingleBean(ZaloBotListenerContainerLifecycle.class);
                    assertThat(context).hasSingleBean(ContainerProperties.class);
                });
    }

    @Test
    void whenNoListenerBean_thenNoContainerCreated() {
        this.contextRunner
                .withPropertyValues("zalobot.bot-token=test-token")
                // no UpdateListener bean
                .run(context -> {
                    assertThat(context).doesNotHaveBean(UpdateListenerContainer.class);
                    assertThat(context).doesNotHaveBean(ZaloBotListenerContainerLifecycle.class);
                });
    }

    @Test
    void whenListenerDisabled_thenNoContainerCreated() {
        this.contextRunner
                .withPropertyValues(
                        "zalobot.bot-token=test-token",
                        "zalobot.listener.enabled=false")
                .withBean(UpdateListener.class, () -> update -> {})
                .run(context -> {
                    assertThat(context).doesNotHaveBean(UpdateListenerContainer.class);
                });
    }

    @Test
    void whenConcurrencyGreaterThanOne_thenConcurrentContainer() {
        this.contextRunner
                .withPropertyValues(
                        "zalobot.bot-token=test-token",
                        "zalobot.listener.concurrency=3")
                .withBean(UpdateListener.class, () -> update -> {})
                .run(context -> {
                    assertThat(context).hasSingleBean(UpdateListenerContainer.class);
                    assertThat(context.getBean(UpdateListenerContainer.class))
                            .isInstanceOf(ConcurrentUpdateListenerContainer.class);
                });
    }
}
```

---

## 3. Integration Test: Minimal Application

A full integration test starts a real Spring Boot application:

```java
package dev.linhvu.zalobot.autoconfigure;

import dev.linhvu.zalobot.client.ZaloBotClient;
import dev.linhvu.zalobot.listener.UpdateListener;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Bean;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(properties = "zalobot.bot-token=integration-test-token")
class ZaloBotStarterIntegrationTests {

    @Autowired
    private ZaloBotClient client;

    @Autowired
    private ZaloBotProperties properties;

    @Test
    void contextLoads() {
        assertThat(this.client).isNotNull();
        assertThat(this.properties.getBotToken()).isEqualTo("integration-test-token");
    }

    @SpringBootApplication
    static class TestApplication {

        @Bean
        UpdateListener updateListener() {
            return update -> {
                // test listener — does nothing
            };
        }
    }
}
```

---

## 4. The `--debug` Flag: Condition Evaluation Report

Running your application with `--debug` (or setting `debug=true` in `application.properties`) activates the `ConditionEvaluationReport`. This prints a detailed report showing every condition that was evaluated:

```bash
java -jar my-bot-app.jar --debug
```

**Example output for ZaloBot auto-configuration:**

```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:
-----------------

   ZaloBotClientAutoConfiguration matched:
      - @ConditionalOnClass found required class
        'dev.linhvu.zalobot.client.ZaloBotClient' (OnClassCondition)
      - @ConditionalOnProperty (zalobot.bot-token) matched (OnPropertyCondition)

   ZaloBotClientAutoConfiguration#zaloBotClient matched:
      - @ConditionalOnMissingBean (types: dev.linhvu.zalobot.client.ZaloBotClient;
        SearchStrategy: all) did not find any beans (OnBeanCondition)

   ZaloBotListenerAutoConfiguration matched:
      - @ConditionalOnClass found required class
        'dev.linhvu.zalobot.listener.UpdateListenerContainer' (OnClassCondition)
      - @ConditionalOnBean (types: dev.linhvu.zalobot.client.ZaloBotClient;
        SearchStrategy: all) found bean 'zaloBotClient' (OnBeanCondition)
      - @ConditionalOnProperty (zalobot.listener.enabled=true) matched (OnPropertyCondition)


Negative matches:
-----------------

   ZaloBotListenerAutoConfiguration#zaloBotListenerContainer:
      - @ConditionalOnBean (types: dev.linhvu.zalobot.listener.UpdateListener;
        SearchStrategy: all) did not find any beans (OnBeanCondition)
```

This tells you:
- Which conditions matched (positive) — these auto-configurations were applied
- Which conditions didn't match (negative) — these were skipped, with the reason

In the example above, the listener container was NOT created because no `UpdateListener` bean was found — the user forgot to define one.

---

## 5. End-User Experience

### `application.yml`

```yaml
zalobot:
  bot-token: ${ZALO_BOT_TOKEN}
  listener:
    concurrency: 2
    poll-timeout: 30s
```

### `MyBotApp.java`

```java
@SpringBootApplication
public class MyBotApp {

    public static void main(String[] args) {
        SpringApplication.run(MyBotApp.class, args);
    }

    @Bean
    UpdateListener updateListener(ZaloBotClient client) {
        return update -> {
            // Echo bot: sends back whatever it receives
            client.sendMessage()
                    .body(new SendMessage(
                            update.message().chat().id(),
                            "Echo: " + update.message().text()))
                    .retrieve()
                    .call(SendMessageResult.class);
        };
    }
}
```

### What Happens at Startup

1. Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` files
2. Finds `ZaloBotClientAutoConfiguration` and `ZaloBotListenerAutoConfiguration`
3. Phase 4: `OnClassCondition` confirms `ZaloBotClient.class` and `UpdateListenerContainer.class` are present
4. Phase 6: `@ConditionalOnProperty("zalobot.bot-token")` matches (token is set)
5. `ZaloBotProperties` is created and bound: `botToken="..."`, `listener.concurrency=2`, `listener.pollTimeout=30s`
6. `ZaloBotClient.Builder` is created (prototype scope) with properties from YAML
7. `ZaloBotClient` is created from the builder
8. `ZaloBotListenerAutoConfiguration` runs (after client):
   - `ContainerProperties` bean created, wired with `updateListener` bean
   - `ConcurrentUpdateListenerContainer` created (concurrency=2)
   - `ZaloBotListenerContainerLifecycle` created
9. Spring context finishes starting → `SmartLifecycle.start()` called
10. Listener container starts polling the Zalo API with 2 concurrent consumers

**Zero configuration code written by the user.** Just a dependency, a token, and an `UpdateListener` bean.

---

## Verification Checklist

```
[ ] mvn clean install — all modules compile
[ ] ApplicationContextRunner test: token present → client bean exists
[ ] ApplicationContextRunner test: token missing → no beans
[ ] ApplicationContextRunner test: user override → auto-config backs away
[ ] ApplicationContextRunner test: listener bean → container created
[ ] ApplicationContextRunner test: no listener bean → no container
[ ] ApplicationContextRunner test: listener disabled → no container
[ ] ApplicationContextRunner test: concurrency > 1 → ConcurrentContainer
[ ] Integration test: @SpringBootTest with token → context loads
[ ] --debug flag: condition report shows expected matches
```

---

## Complete Chapter Progression

```
Ch1  Manual Config Hell         → "Why do we need starters?"
Ch2  @ConfigurationProperties   → "Externalize config"
Ch3  @AutoConfiguration         → "Who creates beans?"
Ch4  Conditional Annotations    → "When to create?"
Ch5  Discovery (.imports)       → "How does Spring find us?"
Ch6  Full Pipeline              → "Complete loading picture"
Ch7  ZaloBot Properties         → "Build it: properties"
Ch8  ZaloBot Auto-Configuration → "Build it: auto-config"
Ch9  ZaloBot Starter Module     → "Build it: POM/dependencies"
Ch10 Testing                    → "Verify it all works" ← YOU ARE HERE
```

You now have a complete understanding of how Spring Boot starters work — from the annotation mechanism down to the source code — and a fully specified custom starter for the ZaloBot SDK.
