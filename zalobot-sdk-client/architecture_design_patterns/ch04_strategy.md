# Chapter 4: Strategy — Pluggable Algorithms

## What You'll Learn
- How the **Strategy** pattern decouples algorithm selection from algorithm execution
- How Spring's `ConversionService` uses strategies for type conversion
- How `PlatformTransactionManager` lets you swap transaction implementations
- The difference between Strategy and Template Method

---

## 4.1 The Problem: Algorithm Lock-In

In Chapter 3, our `SimpleTransactionTemplate` directly manages JDBC connections. What if the application uses JPA? Or distributed transactions (JTA)? We'd need separate template classes for each:

```java
// This doesn't scale!
JdbcTransactionTemplate    → manages java.sql.Connection
JpaTransactionTemplate     → manages javax.persistence.EntityManager
JtaTransactionTemplate     → manages javax.transaction.UserTransaction
```

The **transaction lifecycle** (begin → execute → commit/rollback) is the same. Only the **mechanism** changes.

## 4.2 Strategy Pattern: Swap the Algorithm

Strategy encapsulates interchangeable algorithms behind an interface:

```
┌─────────────────────────────────────────┐
│           Client (Template)              │
│                                         │
│  uses ──► TransactionManager (Strategy) │
│            │                            │
│      ┌─────┼──────────────┐             │
│      │     │              │             │
│    JDBC   JPA            JTA            │
│   Manager Manager       Manager         │
│                                         │
│  Client doesn't know which one it uses! │
└─────────────────────────────────────────┘
```

> ★ **Insight** ─────────────────────────────────────
> - **Strategy vs Template Method:** Template Method defines an algorithm with variable STEPS. Strategy defines a family of interchangeable ALGORITHMS. They often work together — the Template calls a Strategy to execute one of its steps.
> - In Spring, `TransactionTemplate` (Template Method) uses `PlatformTransactionManager` (Strategy). The template defines WHEN to begin/commit/rollback. The strategy defines HOW.
> - **When to use Strategy:** When you have multiple algorithms that solve the same problem differently, and you want to select one at runtime. Examples: sorting algorithms, authentication methods, serialization formats.
> ─────────────────────────────────────────────────────

## 4.3 Building SimpleTransactionManager (Strategy)

### Step 1: The Strategy Interface

```java
// === SimpleTransactionManager.java ===
/**
 * Strategy interface for transaction management.
 * Different implementations handle JDBC, JPA, JTA, etc.
 */
public interface SimpleTransactionManager {
    SimpleTransactionStatus begin();
    void commit(SimpleTransactionStatus status);
    void rollback(SimpleTransactionStatus status);
}
```

```java
// === SimpleTransactionStatus.java ===
/**
 * Represents the state of an active transaction.
 */
public class SimpleTransactionStatus {
    private final Object transaction;  // The underlying transaction resource
    private boolean rollbackOnly = false;

    public SimpleTransactionStatus(Object transaction) {
        this.transaction = transaction;
    }

    @SuppressWarnings("unchecked")
    public <T> T getTransaction() { return (T) transaction; }

    public void setRollbackOnly() { this.rollbackOnly = true; }
    public boolean isRollbackOnly() { return rollbackOnly; }
}
```

### Step 2: Concrete Strategies

```java
// === JdbcTransactionManager.java ===
import java.sql.Connection;
import java.sql.SQLException;

/**
 * Strategy implementation for JDBC transactions.
 */
public class JdbcTransactionManager implements SimpleTransactionManager {

    private final SimpleDataSource dataSource;

    public JdbcTransactionManager(SimpleDataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public SimpleTransactionStatus begin() {
        try {
            Connection conn = dataSource.getConnection();
            conn.setAutoCommit(false);
            return new SimpleTransactionStatus(conn);
        } catch (SQLException e) {
            throw new DataAccessException("Cannot begin transaction", e);
        }
    }

    @Override
    public void commit(SimpleTransactionStatus status) {
        Connection conn = status.getTransaction();
        try {
            if (status.isRollbackOnly()) {
                conn.rollback();  // Honor rollback-only flag
            } else {
                conn.commit();
            }
        } catch (SQLException e) {
            throw new DataAccessException("Cannot commit", e);
        } finally {
            closeQuietly(conn);
        }
    }

    @Override
    public void rollback(SimpleTransactionStatus status) {
        Connection conn = status.getTransaction();
        try {
            conn.rollback();
        } catch (SQLException e) {
            throw new DataAccessException("Cannot rollback", e);
        } finally {
            closeQuietly(conn);
        }
    }

    private void closeQuietly(AutoCloseable c) {
        try { if (c != null) c.close(); } catch (Exception ignored) {}
    }
}
```

```java
// === InMemoryTransactionManager.java ===
import java.util.ArrayList;
import java.util.List;

/**
 * A simple in-memory "transaction manager" for testing.
 * Demonstrates that Strategy lets you swap the entire implementation.
 */
public class InMemoryTransactionManager implements SimpleTransactionManager {

    private final List<String> log = new ArrayList<>();

    @Override
    public SimpleTransactionStatus begin() {
        log.add("BEGIN");
        return new SimpleTransactionStatus("in-memory-tx-" + System.nanoTime());
    }

    @Override
    public void commit(SimpleTransactionStatus status) {
        log.add("COMMIT");
    }

    @Override
    public void rollback(SimpleTransactionStatus status) {
        log.add("ROLLBACK");
    }

    public List<String> getLog() { return log; }
}
```

### Step 3: Update TransactionTemplate to Use Strategy

```java
// === SimpleTransactionTemplate.java (updated) ===
/**
 * Template Method + Strategy:
 * - Template defines WHEN to begin/commit/rollback
 * - Strategy (SimpleTransactionManager) defines HOW
 */
public class SimpleTransactionTemplate {

    private final SimpleTransactionManager transactionManager;  // ◄── Strategy!

    public SimpleTransactionTemplate(SimpleTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public <T> T execute(TransactionCallback<T> action) {
        // 1. Delegate "begin" to strategy
        SimpleTransactionStatus status = transactionManager.begin();
        T result;
        try {
            // 2. Execute user's business logic
            result = action.doInTransaction();
        } catch (RuntimeException | Error ex) {
            // 3. Delegate "rollback" to strategy
            transactionManager.rollback(status);
            throw ex;
        } catch (Throwable ex) {
            transactionManager.rollback(status);
            throw new DataAccessException("Unexpected exception in transaction", ex);
        }
        // 4. Delegate "commit" to strategy
        transactionManager.commit(status);
        return result;
    }
}
```

## 4.4 Building SimpleConversionService (Strategy Registry)

Spring's `ConversionService` takes Strategy further — it maintains a **registry of strategies**:

```java
// === SimpleConverter.java ===
/**
 * Strategy interface for type conversion.
 * Each converter handles one source→target type pair.
 */
@FunctionalInterface
public interface SimpleConverter<S, T> {
    T convert(S source);
}
```

```java
// === SimpleConversionService.java ===
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Registry of conversion strategies.
 * Demonstrates Strategy pattern with runtime registration.
 */
public class SimpleConversionService {

    // Key: "sourceType->targetType", Value: converter strategy
    private final Map<String, SimpleConverter<?, ?>> converters = new ConcurrentHashMap<>();

    /**
     * Register a conversion strategy for a specific type pair.
     */
    public <S, T> void addConverter(Class<S> sourceType, Class<T> targetType,
                                     SimpleConverter<S, T> converter) {
        String key = makeKey(sourceType, targetType);
        converters.put(key, converter);
    }

    /**
     * Check if a conversion is possible.
     */
    public boolean canConvert(Class<?> sourceType, Class<?> targetType) {
        return sourceType.equals(targetType)
                || converters.containsKey(makeKey(sourceType, targetType));
    }

    /**
     * Convert a value using the registered strategy.
     */
    @SuppressWarnings("unchecked")
    public <T> T convert(Object source, Class<T> targetType) {
        if (source == null) return null;

        Class<?> sourceType = source.getClass();

        // No conversion needed
        if (targetType.isInstance(source)) {
            return targetType.cast(source);
        }

        // Find the strategy
        String key = makeKey(sourceType, targetType);
        SimpleConverter<Object, T> converter = (SimpleConverter<Object, T>) converters.get(key);

        if (converter == null) {
            throw new RuntimeException(
                "No converter found for " + sourceType.getSimpleName()
                    + " → " + targetType.getSimpleName());
        }

        return converter.convert(source);
    }

    private String makeKey(Class<?> source, Class<?> target) {
        return source.getName() + "->" + target.getName();
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - A **Strategy Registry** (like `ConversionService`) is a common extension of the Strategy pattern. Instead of injecting ONE strategy, you maintain a MAP of strategies keyed by some discriminator (here, the source→target type pair).
> - Real Spring's `GenericConversionService` uses a sophisticated lookup with type hierarchy searching — if no converter for `Integer→String` exists, it checks `Number→String`, then `Object→String`. This is the **most specific match** principle.
> - **When NOT to use Strategy Registry:** If you only ever have 2-3 strategies that are selected at startup. A simple `if/else` or constructor injection is clearer than a registry for small sets.
> ─────────────────────────────────────────────────────

## 4.5 Usage

```java
// === StrategyDemo.java ===
public class StrategyDemo {
    public static void main(String[] args) {
        // ── Transaction Manager Strategy ──
        // Swap InMemoryTransactionManager for JdbcTransactionManager in production
        InMemoryTransactionManager txManager = new InMemoryTransactionManager();
        SimpleTransactionTemplate txTemplate = new SimpleTransactionTemplate(txManager);

        String result = txTemplate.execute(() -> {
            return "business logic result";
        });
        System.out.println("Result: " + result);
        System.out.println("TX Log: " + txManager.getLog());
        // Output: TX Log: [BEGIN, COMMIT]

        // ── Conversion Service Strategy Registry ──
        SimpleConversionService conversionService = new SimpleConversionService();

        // Register strategies
        conversionService.addConverter(String.class, Integer.class, Integer::parseInt);
        conversionService.addConverter(String.class, Boolean.class, Boolean::parseBoolean);
        conversionService.addConverter(Integer.class, String.class, String::valueOf);

        // Use strategies
        Integer num = conversionService.convert("42", Integer.class);
        Boolean flag = conversionService.convert("true", Boolean.class);
        String str = conversionService.convert(42, String.class);

        System.out.println("Converted: " + num + ", " + flag + ", " + str);
        // Output: Converted: 42, true, 42
    }
}
```

## 4.6 Connection to Real Spring

| Simplified | Real Spring Source | Line |
|-----------|-------------------|------|
| `SimpleTransactionManager` | `PlatformTransactionManager` | `:47-120` |
| `SimpleTransactionStatus` | `TransactionStatus` | Interface in `spring-tx` |
| `JdbcTransactionManager` | `DataSourceTransactionManager` | `spring-jdbc` |
| `SimpleConverter<S,T>` | `Converter<S,T>` | `:46` |
| `SimpleConversionService` | `GenericConversionService` | `:77-243` |
| `addConverter()` | `GenericConversionService.addConverter()` | `:85-95` |
| `convert()` | `GenericConversionService.convert()` | `:169-185` |

**What real Spring adds:**
- `GenericConverter` for complex multi-type conversions (`:57-66`)
- `ConverterFactory` for creating family of converters (e.g., `String→Enum<T>`)
- `ConvertiblePair` for type-safe converter registration (`:72-119`)
- Converter cache for performance (`:79`)
- Type hierarchy searching in converter lookup

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Strategy Pattern** | Encapsulate interchangeable algorithms behind an interface |
| **Strategy Registry** | Map of strategies keyed by discriminator; dynamic selection |
| **Template + Strategy** | Template defines when; Strategy defines how |
| **ConversionService** | Registry of type conversion strategies |
| **PlatformTransactionManager** | Strategy for transaction lifecycle (begin/commit/rollback) |

**Next: Chapter 5** — We build `SimpleAopProxy` using the **Proxy** pattern for transparent cross-cutting concerns.
