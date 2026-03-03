# Chapter 3: Template Method — Eliminating Boilerplate

## What You'll Learn
- How the **Template Method** pattern captures invariant algorithms with customizable steps
- How Spring's `JdbcTemplate` eliminates 90% of JDBC boilerplate
- How `TransactionTemplate` applies the same pattern to transactions
- The **Callback** variant — using functional interfaces instead of subclassing

---

## 3.1 The Problem: Boilerplate Repetition

Every JDBC operation follows the same skeleton:

```
1. Get connection from DataSource
2. Create Statement/PreparedStatement
3. Execute SQL                        ◄── THIS CHANGES
4. Process results                    ◄── THIS CHANGES
5. Handle exceptions
6. Close ResultSet, Statement, Connection (in reverse order!)
```

Steps 1, 2, 5, and 6 are **identical** every time. Only steps 3 and 4 vary. Writing them repeatedly is error-prone — missing a `finally` block causes connection leaks.

```java
// Without Template Method: 20 lines of boilerplate for 2 lines of logic
Connection conn = null;
PreparedStatement ps = null;
ResultSet rs = null;
try {
    conn = dataSource.getConnection();
    ps = conn.prepareStatement("SELECT name FROM users WHERE id = ?");
    ps.setLong(1, userId);           // ◄── only these 3 lines matter
    rs = ps.executeQuery();           // ◄──
    return rs.next() ? rs.getString("name") : null;  // ◄──
} catch (SQLException e) {
    throw new RuntimeException(e);
} finally {
    if (rs != null) try { rs.close(); } catch (SQLException ignored) {}
    if (ps != null) try { ps.close(); } catch (SQLException ignored) {}
    if (conn != null) try { conn.close(); } catch (SQLException ignored) {}
}
```

## 3.2 Template Method: Fix the Skeleton, Vary the Steps

The pattern defines an algorithm skeleton in a method, deferring specific steps to callbacks:

```
┌─────────────────────────────────────────────┐
│         Template Method (fixed)              │
│                                             │
│   1. Acquire resource        ← FIXED        │
│   2. Configure resource      ← FIXED        │
│   3. ──► callback.execute() ◄── VARIABLE    │
│   4. Handle errors           ← FIXED        │
│   5. Release resource        ← FIXED        │
│                                             │
│   Only step 3 changes per use case          │
└─────────────────────────────────────────────┘
```

> ★ **Insight** ─────────────────────────────────────
> - Classic Template Method uses **inheritance** (abstract class with abstract steps). Spring's variant uses **callbacks** (functional interfaces passed as parameters). This is sometimes called the **Template Callback** or **Strategy in Template** pattern.
> - Callbacks are superior here because: (1) you don't need a new subclass per query, (2) lambdas make it concise, (3) multiple callbacks can be composed. The original GoF Template Method via inheritance creates class explosion.
> - **When to use Template Method:** When you have an invariant algorithm with 1-2 steps that vary. If MORE than half the steps vary, use Strategy instead — the algorithm itself differs, not just individual steps.
> ─────────────────────────────────────────────────────

## 3.3 Building SimpleJdbcTemplate

### Step 1: Define the Callback Interfaces

```java
// === StatementCallback.java ===
import java.sql.Statement;
import java.sql.SQLException;

@FunctionalInterface
public interface StatementCallback<T> {
    T doInStatement(Statement stmt) throws SQLException;
}
```

```java
// === ResultSetExtractor.java ===
import java.sql.ResultSet;
import java.sql.SQLException;

@FunctionalInterface
public interface ResultSetExtractor<T> {
    T extractData(ResultSet rs) throws SQLException;
}
```

```java
// === RowMapper.java ===
import java.sql.ResultSet;
import java.sql.SQLException;

@FunctionalInterface
public interface RowMapper<T> {
    T mapRow(ResultSet rs, int rowNum) throws SQLException;
}
```

### Step 2: The Template

```java
// === SimpleJdbcTemplate.java ===
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class SimpleJdbcTemplate {

    private final SimpleDataSource dataSource;

    public SimpleJdbcTemplate(SimpleDataSource dataSource) {
        this.dataSource = dataSource;
    }

    // ══════════════════════════════════════════════════════════
    // TEMPLATE METHOD: The invariant algorithm
    // ══════════════════════════════════════════════════════════
    public <T> T execute(StatementCallback<T> callback) {
        Connection conn = null;
        Statement stmt = null;
        try {
            // Step 1: Acquire resource (FIXED)
            conn = dataSource.getConnection();

            // Step 2: Create statement (FIXED)
            stmt = conn.createStatement();

            // Step 3: Execute callback (VARIABLE) ◄── the only part that changes
            T result = callback.doInStatement(stmt);

            return result;
        } catch (SQLException ex) {
            // Step 4: Translate exception (FIXED)
            throw new DataAccessException("SQL error: " + ex.getMessage(), ex);
        } finally {
            // Step 5: Release resources (FIXED)
            closeQuietly(stmt);
            closeQuietly(conn);
        }
    }

    // ══════════════════════════════════════════════════════════
    // Convenience methods built on top of the template
    // ══════════════════════════════════════════════════════════

    /**
     * Query with a ResultSetExtractor — full control over ResultSet processing.
     */
    public <T> T query(String sql, ResultSetExtractor<T> extractor) {
        return execute(stmt -> {
            ResultSet rs = null;
            try {
                rs = stmt.executeQuery(sql);
                return extractor.extractData(rs);
            } finally {
                closeQuietly(rs);
            }
        });
    }

    /**
     * Query with a RowMapper — maps each row to an object.
     * This builds on query(String, ResultSetExtractor) by wrapping
     * the RowMapper in a ResultSetExtractor.
     */
    public <T> List<T> query(String sql, RowMapper<T> rowMapper) {
        return query(sql, rs -> {
            List<T> results = new ArrayList<>();
            int rowNum = 0;
            while (rs.next()) {
                results.add(rowMapper.mapRow(rs, rowNum++));
            }
            return results;
        });
    }

    /**
     * Execute an update (INSERT, UPDATE, DELETE).
     */
    public int update(String sql) {
        return execute(stmt -> stmt.executeUpdate(sql));
    }

    // ── Resource cleanup helpers ──
    private void closeQuietly(AutoCloseable resource) {
        if (resource != null) {
            try { resource.close(); } catch (Exception ignored) {}
        }
    }
}
```

```java
// === DataAccessException.java ===
public class DataAccessException extends RuntimeException {
    public DataAccessException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### Step 3: TransactionTemplate — Same Pattern, Different Domain

```java
// === TransactionCallback.java ===
@FunctionalInterface
public interface TransactionCallback<T> {
    T doInTransaction();
}
```

```java
// === SimpleTransactionTemplate.java ===
import java.sql.Connection;
import java.sql.SQLException;

public class SimpleTransactionTemplate {

    private final SimpleDataSource dataSource;

    public SimpleTransactionTemplate(SimpleDataSource dataSource) {
        this.dataSource = dataSource;
    }

    /**
     * Template Method for transactions:
     * 1. Begin transaction      (FIXED)
     * 2. Execute business logic (VARIABLE — the callback)
     * 3. Commit                 (FIXED)
     * 4. Rollback on error      (FIXED)
     */
    public <T> T execute(TransactionCallback<T> action) {
        Connection conn = null;
        try {
            conn = dataSource.getConnection();
            conn.setAutoCommit(false);          // 1. Begin

            T result = action.doInTransaction(); // 2. Execute (VARIABLE)

            conn.commit();                       // 3. Commit
            return result;
        } catch (Exception ex) {
            rollbackQuietly(conn);               // 4. Rollback
            throw new DataAccessException("Transaction failed: " + ex.getMessage(), ex);
        } finally {
            restoreAutoCommit(conn);
            closeQuietly(conn);
        }
    }

    private void rollbackQuietly(Connection conn) {
        if (conn != null) {
            try { conn.rollback(); } catch (SQLException ignored) {}
        }
    }

    private void restoreAutoCommit(Connection conn) {
        if (conn != null) {
            try { conn.setAutoCommit(true); } catch (SQLException ignored) {}
        }
    }

    private void closeQuietly(AutoCloseable resource) {
        if (resource != null) {
            try { resource.close(); } catch (Exception ignored) {}
        }
    }
}
```

> ★ **Insight** ─────────────────────────────────────
> - Notice how `JdbcTemplate` and `TransactionTemplate` both follow the same meta-pattern: acquire → execute callback → handle errors → release. This is the Template Method applied to **resource management**.
> - In real Spring, `TransactionTemplate` delegates to `PlatformTransactionManager` (a Strategy!) rather than managing JDBC connections directly. This means the same template works for JDBC, JPA, JTA, and any other transaction technology.
> - The `DataAccessException` hierarchy converts checked `SQLException` to unchecked exceptions — this is the **Exception Translation** pattern, which Spring uses to decouple application code from specific database vendors.
> ─────────────────────────────────────────────────────

## 3.4 Usage — Before and After

```java
// === TemplateMethodDemo.java ===
public class TemplateMethodDemo {
    public static void main(String[] args) {
        SimpleDataSource ds = new SimpleDataSource("jdbc:h2:mem:test");
        SimpleJdbcTemplate jdbc = new SimpleJdbcTemplate(ds);
        SimpleTransactionTemplate tx = new SimpleTransactionTemplate(ds);

        // BEFORE (Chapter 1): 20 lines of boilerplate
        // AFTER: 1 line!
        List<String> names = jdbc.query(
            "SELECT name FROM users",
            (rs, rowNum) -> rs.getString("name")  // Only the varying part!
        );

        // Transaction with template
        tx.execute(() -> {
            jdbc.update("INSERT INTO users (name) VALUES ('Alice')");
            jdbc.update("INSERT INTO users (name) VALUES ('Bob')");
            return null;  // commit happens automatically
        });
        // rollback happens automatically on exception

        System.out.println("Users: " + names);
    }
}
```

The boilerplate went from **20 lines to 1 line**. That's the power of Template Method.

## 3.5 Connection to Real Spring

| Simplified | Real Spring Source | Line |
|-----------|-------------------|------|
| `SimpleJdbcTemplate.execute(StatementCallback)` | `JdbcTemplate.execute(StatementCallback, boolean)` | `:404-440` |
| `SimpleJdbcTemplate.query(sql, ResultSetExtractor)` | `JdbcTemplate.query(String, ResultSetExtractor)` | `:465-492` |
| `SimpleJdbcTemplate.query(sql, RowMapper)` | `JdbcTemplate.query(String, RowMapper)` | `:500-502` |
| `SimpleTransactionTemplate.execute()` | `TransactionTemplate.execute(TransactionCallback)` | `:127-152` |
| `StatementCallback<T>` | `StatementCallback<T>` | Spring's is identical |
| `ResultSetExtractor<T>` | `ResultSetExtractor<T>` | Spring's is identical |
| `RowMapper<T>` | `RowMapper<T>` | Spring's is identical |
| `DataAccessException` | `DataAccessException` hierarchy | 30+ subclasses in real Spring |

**What real Spring adds:**
- `PreparedStatementCallback` for parameterized queries
- `NamedParameterJdbcTemplate` wrapping `JdbcTemplate` with named parameters
- `SQLExceptionTranslator` for vendor-specific exception translation
- `queryForStream()` using `ResultSetSpliterator` for lazy streaming (`:505-524`)
- `CallbackPreferringPlatformTransactionManager` optimization (`:129`)

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Template Method** | Define algorithm skeleton; let callbacks fill in the variable steps |
| **Callback variant** | Use functional interfaces instead of subclassing — more flexible |
| **Resource management template** | Acquire → execute → handle errors → release |
| **Exception Translation** | Convert checked exceptions to unchecked `DataAccessException` hierarchy |
| **Layered convenience methods** | `query(sql, RowMapper)` builds on `query(sql, ResultSetExtractor)` builds on `execute(StatementCallback)` |

**Next: Chapter 4** — We build pluggable **Strategy** implementations for type conversion and transaction management.
