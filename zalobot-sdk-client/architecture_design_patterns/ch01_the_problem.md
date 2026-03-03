# Chapter 1: The Problem — Why Do We Need Design Patterns in a Framework?

## What You'll Learn
- Why naive application code becomes unmaintainable at scale
- The specific problems that Spring's design patterns solve
- How tightly-coupled code fails when requirements change
- The cost of boilerplate and the value of abstractions

---

## 1.1 A Naive Application Without Patterns

Imagine building a web application that manages users. You start simple:

```java
// Main.java — Everything hardcoded, no patterns
public class Main {
    public static void main(String[] args) throws Exception {
        // Problem 1: Hard-coded dependencies
        DataSource ds = new HikariDataSource();
        ((HikariDataSource) ds).setJdbcUrl("jdbc:h2:mem:test");

        // Problem 2: Boilerplate everywhere
        Connection conn = ds.getConnection();
        try {
            PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
            ps.setLong(1, 42L);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                String name = rs.getString("name");
                System.out.println("Found: " + name);
            }
            rs.close();
            ps.close();
        } catch (SQLException e) {
            // Problem 3: Poor error handling
            e.printStackTrace();
        } finally {
            conn.close();
        }

        // Problem 4: Cross-cutting concerns mixed with logic
        long start = System.currentTimeMillis();
        // ... business logic here ...
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("Took " + elapsed + "ms");  // timing everywhere!

        // Problem 5: No way to swap implementations
        // What if we want to switch from H2 to PostgreSQL?
        // What if we want to add caching?
        // What if we want transactions?
    }
}
```

Every method in this application suffers from the same problems:

| Problem | Impact |
|---------|--------|
| Hard-coded `new` everywhere | Cannot test, cannot swap implementations |
| JDBC boilerplate repeated | 15 lines to execute 1 query; error-prone resource leaks |
| Cross-cutting concerns scattered | Timing, logging, security mixed into every method |
| No lifecycle management | Who creates objects? Who destroys them? Who manages connections? |
| No event system | Components cannot communicate without direct references |

## 1.2 What Happens When Requirements Change

Your product manager says: "Add transaction support, caching, and audit logging to every database operation."

Without patterns, you must modify **every single method**:

```java
// BEFORE: Simple query
public User findUser(long id) {
    Connection conn = ds.getConnection();
    // ... query ...
}

// AFTER: Adding transactions + caching + audit (without patterns)
public User findUser(long id) {
    // Check cache first
    User cached = cache.get("user:" + id);
    if (cached != null) return cached;

    // Start transaction
    Connection conn = ds.getConnection();
    conn.setAutoCommit(false);
    try {
        // ... query ...

        // Audit log
        auditLogger.log("findUser", id, currentUser());

        conn.commit();

        // Update cache
        cache.put("user:" + id, user);
        return user;
    } catch (Exception e) {
        conn.rollback();
        throw e;
    } finally {
        conn.close();
    }
}
```

**Every** method now has 30+ lines of ceremony around 3 lines of actual logic. This is the **Boilerplate Explosion** problem.

## 1.3 The Five Forces That Demand Patterns

Spring Framework addresses five fundamental forces through design patterns:

```
    ┌─────────────────────────────────────────────────────┐
    │                                                     │
    │  ① CREATION        "Who creates objects?"            │
    │     → Factory, Singleton, Prototype, Builder         │
    │                                                     │
    │  ② STRUCTURE       "How are objects composed?"       │
    │     → Proxy, Decorator, Adapter, Facade, Composite  │
    │                                                     │
    │  ③ BEHAVIOR        "How do objects interact?"        │
    │     → Template Method, Strategy, Observer,           │
    │       Chain of Responsibility, Command, Visitor      │
    │                                                     │
    │  ④ LIFECYCLE       "When do objects live and die?"   │
    │     → Aware, BeanPostProcessor, InitializingBean    │
    │                                                     │
    │  ⑤ CROSS-CUTTING   "How to add concerns globally?"  │
    │     → AOP Proxy, Interceptor Chain                  │
    │                                                     │
    └─────────────────────────────────────────────────────┘
```

## 1.4 Our Journey: Building a Mini-Spring

Over the next chapters, we'll build a simplified Spring Framework that demonstrates every major pattern. Each chapter introduces ONE pattern, showing:
1. The **concrete problem** it solves
2. A **simplified implementation** (compilable Java)
3. The **mapping** to real Spring source code

Here's our roadmap:

```
Ch02: Factory + Singleton    → SimpleContainer (bean creation + caching)
Ch03: Template Method        → SimpleJdbcTemplate (eliminate boilerplate)
Ch04: Strategy               → Pluggable converters + transaction managers
Ch05: Proxy                  → SimpleAopProxy (transparent cross-cutting)
Ch06: Observer               → SimpleEventBus (decoupled communication)
Ch07: Decorator              → SimpleBeanPostProcessor (bean enhancement chain)
Ch08: Adapter + Chain        → SimpleDispatcher (request processing pipeline)
Ch09: Builder + Composite    → SimpleBeanDefinitionBuilder + composites
Ch10: Facade + Aware         → SimpleApplicationContext (unified API)
Ch11: Visitor + Command      → BeanDefinition traversal + callbacks
Ch12: Integration            → All patterns working together
```

---

## 1.5 Connection to Real Spring

The problems we identified map directly to Spring modules:

| Problem | Spring Solution | Module |
|---------|----------------|--------|
| Hard-coded dependencies | IoC Container | `spring-beans` |
| JDBC boilerplate | `JdbcTemplate` | `spring-jdbc` |
| Transaction ceremony | `@Transactional` + AOP | `spring-tx`, `spring-aop` |
| Cross-cutting concerns | AOP proxies | `spring-aop` |
| Component communication | Application events | `spring-context` |
| Request routing | DispatcherServlet | `spring-webmvc` |

> ★ **Insight** ─────────────────────────────────────
> - Design patterns are not academic exercises — they emerged as solutions to **real pain** in large codebases. Spring Framework's genius is applying them systematically so that application developers never need to write boilerplate.
> - The key principle: **separate what changes from what stays the same**. The JDBC connection lifecycle never changes (get connection → execute → close). The SQL query always changes. Template Method captures this separation.
> ─────────────────────────────────────────────────────

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Boilerplate Explosion** | Adding concerns multiplies code in every method |
| **Tight Coupling** | `new` creates direct dependency on concrete classes |
| **Cross-Cutting Scatter** | Logging, timing, security mixed into business logic |
| **Five Forces** | Creation, Structure, Behavior, Lifecycle, Cross-Cutting |
| **Pattern Catalog** | 17+ patterns organized by force they address |

**Next: Chapter 2** — We build `SimpleContainer` using the **Factory** and **Singleton** patterns to solve the object creation problem.
