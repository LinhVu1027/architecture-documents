# Chapter 4: Conditional Annotations — When to Create Beans?

> An auto-configuration class that always runs is dangerous. Conditional annotations add the "if" to "create this bean if..."

---

## The Three Core Conditions

### 1. `@ConditionalOnClass` — "Is the library on the classpath?"

```java
// ConditionalOnClass.java:62-66
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)          // ← points to the implementation
public @interface ConditionalOnClass {
    Class<?>[] value() default {};             // safe: parsed by ASM, not class loading
    String[] name() default {};                // for classes that might not exist
}
```

**Source:** `core/spring-boot-autoconfigure/.../condition/ConditionalOnClass.java:65`

**Use case:** Your auto-configuration should only activate when the user has the ZaloBot client library on their classpath.

```java
@AutoConfiguration
@ConditionalOnClass(ZaloBotClient.class)      // only if zalobot-client JAR is present
public class ZaloBotClientAutoConfiguration { ... }
```

**Implementation:** `OnClassCondition` at `OnClassCondition.java:45-46`
- `@Order(Ordered.HIGHEST_PRECEDENCE)` — checked first (cheapest check)
- Extends `FilteringSpringBootCondition` — also works as an early filter in Phase 4

### 2. `@ConditionalOnMissingBean` — "Has the user already defined this?"

```java
// ConditionalOnMissingBean.java:66-70
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)           // ← points to the implementation
public @interface ConditionalOnMissingBean {
    Class<?>[] value() default {};            // bean type to check
    String[] type() default {};               // by class name
    String[] name() default {};               // by bean name
    Class<? extends Annotation>[] annotation() default {};
    SearchStrategy search() default SearchStrategy.ALL;
    // ...
}
```

**Source:** `core/spring-boot-autoconfigure/.../condition/ConditionalOnMissingBean.java:69`

**Use case:** If the user defines their own `ZaloBotClient` bean, your auto-configured one should back away.

```java
@Bean
@ConditionalOnMissingBean                     // if no ZaloBotClient bean exists yet
ZaloBotClient zaloBotClient(ZaloBotClient.Builder builder) {
    return builder.build();
}
```

When `@ConditionalOnMissingBean` is on a `@Bean` method with no explicit `value`, it defaults to the **return type** of the method. So `@ConditionalOnMissingBean` on the above method means "if no `ZaloBotClient` bean exists."

**Implementation:** `OnBeanCondition` at `OnBeanCondition.java:88`
- `@Order(Ordered.LOWEST_PRECEDENCE)` — checked last (most expensive: scans bean factory)

### 3. `@ConditionalOnProperty` — "Is the property set?"

```java
// ConditionalOnProperty.java:94-99
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.METHOD })
@Documented
@Conditional(OnPropertyCondition.class)       // ← points to the implementation
@Repeatable(ConditionalOnProperties.class)
public @interface ConditionalOnProperty {
    String[] value() default {};              // property name(s)
    String prefix() default "";               // prefix (auto-adds ".")
    String[] name() default {};               // alias for value
    String havingValue() default "";           // expected value ("" = not "false")
    boolean matchIfMissing() default false;    // match when property is absent?
}
```

**Source:** `core/spring-boot-autoconfigure/.../condition/ConditionalOnProperty.java:97`

**Use case:** The bot token is required — don't activate auto-configuration if it's not set.

```java
@AutoConfiguration
@ConditionalOnProperty(name = "zalobot.bot-token")    // must be present
public class ZaloBotClientAutoConfiguration { ... }
```

**Implementation:** `OnPropertyCondition` at `OnPropertyCondition.java:52`
- `@Order(Ordered.HIGHEST_PRECEDENCE + 40)` — checked after class conditions, before bean conditions

---

## The Condition Hierarchy — Template Method Pattern

All three conditions share a common base class:

```java
// SpringBootCondition.java:39-44,115
public abstract class SpringBootCondition implements Condition {

    @Override
    public final boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String classOrMethodName = getClassOrMethodName(metadata);
        try {
            ConditionOutcome outcome = getMatchOutcome(context, metadata);    // ← abstract
            logOutcome(classOrMethodName, outcome);             // ← logging
            recordEvaluation(context, classOrMethodName, outcome);  // ← debug report
            return outcome.isMatch();
        }
        catch (NoClassDefFoundError ex) { ... }
        catch (RuntimeException ex) { ... }
    }

    // Subclasses implement this:
    public abstract ConditionOutcome getMatchOutcome(
            ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

**Source:** `core/spring-boot-autoconfigure/.../condition/SpringBootCondition.java:44,115`

This is the **Template Method** pattern:
- `matches()` is `final` — it handles logging, error handling, and condition reporting
- Subclasses (`OnClassCondition`, `OnBeanCondition`, `OnPropertyCondition`) only implement `getMatchOutcome()` — the pure matching logic

```
SpringBootCondition.matches()        ← FINAL: logging + error handling + reporting
        │
        ▼ calls
getMatchOutcome()                    ← ABSTRACT: each condition implements this
        │
        ├── OnClassCondition:   "Is the class on the classpath?"
        ├── OnBeanCondition:    "Does a bean of this type exist?"
        └── OnPropertyCondition: "Is this property set to this value?"
```

---

## The Dual-Role Optimization: `FilteringSpringBootCondition`

```java
// FilteringSpringBootCondition.java:42-43
abstract class FilteringSpringBootCondition extends SpringBootCondition
        implements AutoConfigurationImportFilter, BeanFactoryAware, BeanClassLoaderAware {

    @Override
    public boolean[] match(@Nullable String[] autoConfigurationClasses,
            AutoConfigurationMetadata autoConfigurationMetadata) {
        // Fast batch check using pre-computed metadata
        // Does NOT load the auto-configuration classes
    }

    protected abstract @Nullable ConditionOutcome[] getOutcomes(
            @Nullable String[] autoConfigurationClasses,
            AutoConfigurationMetadata autoConfigurationMetadata);
}
```

**Source:** `core/spring-boot-autoconfigure/.../condition/FilteringSpringBootCondition.java:42-43,52-69`

`OnClassCondition` and `OnBeanCondition` extend this class, meaning they can work in **two modes**:

| Mode | When | How | Speed |
|---|---|---|---|
| **Early filter** (Phase 4) | Before classes are loaded | Uses annotation metadata cache | Fast — batch check, no class loading |
| **Full condition** (Phase 6) | After classes are loaded | Full `Condition.matches()` evaluation | Slower — per-class and per-`@Bean` |

This dual role is why auto-configuration startup is fast: ~100 candidates are eliminated in Phase 4 without ever loading their classes.

---

## Evolving Chapter 3's Class: Adding Conditionals

Let's add all three conditions to our auto-configuration class:

```java
package dev.linhvu.zalobot.autoconfigure;

import dev.linhvu.zalobot.client.ZaloBotClient;
import dev.linhvu.zalobot.client.ZaloBotUrl;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Scope;

@AutoConfiguration
@ConditionalOnClass(ZaloBotClient.class)                      // ← NEW: class guard
@ConditionalOnProperty(name = "zalobot.bot-token")            // ← NEW: property guard
@EnableConfigurationProperties(ZaloBotProperties.class)
public class ZaloBotClientAutoConfiguration {

    private final ZaloBotProperties properties;

    ZaloBotClientAutoConfiguration(ZaloBotProperties properties) {
        this.properties = properties;
    }

    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)           // ← NEW: fresh builder each time
    @ConditionalOnMissingBean                                 // ← NEW: back-away
    ZaloBotClient.Builder zaloBotClientBuilder(
            ObjectProvider<ZaloBotClientCustomizer> customizerProvider) {
        ZaloBotProperties.Client clientProps = this.properties.getClient();
        ZaloBotUrl url = new ZaloBotUrl(
                clientProps.getScheme(), clientProps.getHost(), clientProps.getPort());

        ZaloBotClient.Builder builder = ZaloBotClient.builder()
                .botToken(this.properties.getBotToken())
                .ZaloBotUrl(url);

        customizerProvider.orderedStream()
                .forEach(customizer -> customizer.customize(builder));
        return builder;
    }

    @Bean
    @ConditionalOnMissingBean                                 // ← NEW: back-away
    ZaloBotClient zaloBotClient(ZaloBotClient.Builder builder) {
        return builder.build();
    }
}
```

### What Each Conditional Does

| Annotation | Target | Effect |
|---|---|---|
| `@ConditionalOnClass(ZaloBotClient.class)` | Class | Entire configuration skipped if `zalobot-client` JAR not on classpath |
| `@ConditionalOnProperty(name = "zalobot.bot-token")` | Class | Entire configuration skipped if bot token not set |
| `@ConditionalOnMissingBean` on `zaloBotClientBuilder()` | Bean | Backs away if user defines their own `ZaloBotClient.Builder` bean |
| `@ConditionalOnMissingBean` on `zaloBotClient()` | Bean | Backs away if user defines their own `ZaloBotClient` bean |

### Evaluation Order

Because `@ConditionalOnClass` has `@Order(HIGHEST_PRECEDENCE)` and `@ConditionalOnMissingBean` has `@Order(LOWEST_PRECEDENCE)`:

```
1. @ConditionalOnClass(ZaloBotClient.class)      ← cheapest check first
2. @ConditionalOnProperty("zalobot.bot-token")    ← then property check
3. @ConditionalOnMissingBean (per @Bean method)   ← most expensive last
```

If any class-level condition fails, all `@Bean` methods are skipped — no wasted work.

---

## What's Next?

We have auto-configuration with conditions, but how does Spring Boot **find** our auto-configuration class? It's not component-scanned. In Chapter 5, we learn about the `.imports` discovery mechanism.

```
Chapter 1  ──► "Why do we need starters?"
Chapter 2  ──► "Externalize config"
Chapter 3  ──► "Who creates beans?"
Chapter 4  ←── YOU ARE HERE: "When to create? Conditional annotations"
Chapter 5  ──► "How does Spring find us? Discovery mechanism"
```
