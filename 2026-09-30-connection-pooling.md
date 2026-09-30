# Day 10: Connection Pooling

## What problem does it solve?

Before your app can run a single SQL query, it needs a **connection** to the database. Opening one is surprisingly expensive:
- a network handshake with the database server,
- often TLS encryption setup,
- authenticating the username and password,
- and the database allocating memory for that session.

This can take anywhere from a few milliseconds to much more, sometimes longer than the query itself. If every request opened a fresh connection and then closed it, most of your time would go into setting up and tearing down connections.

A **connection pool** opens a set of connections **once**, keeps them open, and **lends** them out. A request borrows a connection, runs its queries, and **returns** it to the pool for the next request to reuse.

Analogy: a car-rental counter. Instead of building a new car for every customer and scrapping it afterward, there's a fleet. You borrow one and bring it back.

## How it works

```
request → borrow connection from pool → run queries → return connection to pool
```

The pool manages:
- **Minimum idle connections:** kept open and ready, even when traffic is quiet.
- **Maximum pool size:** a hard cap on how many connections can exist at once.
- **Connection timeout:** how long a request waits for a free connection before giving up with an error.
- **Validation:** checking that a connection is still alive before handing it out (databases and firewalls sometimes close idle connections).

## Why there's a maximum

It's tempting to think "more connections = more throughput." It doesn't work that way:
- **The database has its own limit.** PostgreSQL, for example, has a configured `max_connections`, and each connection uses memory on the database server.
- **You probably run several app instances.** With 10 instances and a pool of 50 each, that's 500 connections hitting one database (Day 7: horizontal scaling multiplies this).
- **Past a point, more connections make things slower.** The database has limited CPU cores and disk. Hundreds of queries competing at once spend time switching between each other instead of finishing.

A small pool is often faster than a huge one. HikariCP's own documentation recommends starting from a small pool and measuring, rather than defaulting to large numbers.

## Java angle: HikariCP

Spring Boot uses **HikariCP** as its default connection pool. You rarely touch it directly, but you configure it:

```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
```

(Those `connection-timeout` numbers are in milliseconds, so this example is 30 seconds. Treat all three values as examples to tune, not recommendations.)

When you use Spring Data JPA or `JdbcTemplate`, borrowing and returning happens behind the scenes. With `@Transactional` (Day 5), the connection is held for the **whole transaction**, from start to commit. That's one more reason to keep transactions short.

If you use raw JDBC, always return the connection with try-with-resources:

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?")) {
    ps.setLong(1, userId);
    ResultSet rs = ps.executeQuery();
    // ...
}   // conn.close() here returns it to the pool; it does NOT close the real connection
```

With a pool, `close()` doesn't actually close the network connection. It hands the connection back to the pool.

## The classic failure: connection leaks

If code borrows a connection and **never returns it** (for example, an exception is thrown before `close()`, and there's no try-with-resources), that connection is gone for good. Leak enough of them and the pool runs dry. Every new request then waits for the connection timeout and fails, even though the database is perfectly healthy.

The symptom in logs usually looks like "Connection is not available, request timed out after 30000ms." HikariCP has a `leakDetectionThreshold` setting that logs a warning when a connection has been borrowed for suspiciously long.

## Common pitfalls

- **Pool too large.** Overloads the database, especially multiplied across many app instances.
- **Connection leaks.** Always use try-with-resources with raw JDBC.
- **Holding a connection during slow work.** Calling an external API inside a `@Transactional` method holds a pooled connection the whole time. Other requests queue up behind it.
- **Treating pool exhaustion as a database problem.** Often the database is fine, and the app just isn't returning connections fast enough.

## Interviewer follow-ups

**"Why use a connection pool?"** Opening database connections is expensive (network handshake, authentication, server-side setup). A pool opens them once and reuses them, cutting latency and limiting how many connections hit the database.

**"Requests are timing out waiting for a connection, but the database looks healthy. What do you check?"** Leaks (connections never returned), long transactions holding connections, slow queries holding them, and whether the pool size fits the load. Enable leak detection to find where connections are borrowed and not returned.

## Retention Check

1. Why is opening a new database connection expensive?
2. What does a connection pool do instead?
3. Why does the pool have a maximum size?
4. How can horizontal scaling multiply the number of connections the database sees?
5. What does `conn.close()` do when you're using a pool?
6. What is a connection leak, and what's the symptom?
7. How does `@Transactional` affect how long a connection is held?
8. Why might a smaller pool perform better than a very large one?

**Key points to self-check against:**
- Network handshake, possibly TLS, authentication, and server-side session setup, often slower than the query itself.
- Opens connections once, lends them out, and takes them back for reuse.
- The database has connection limits and finite CPU/memory; too many connections slow everything down.
- Each app instance has its own pool, so instances × pool size = total connections.
- Returns the connection to the pool instead of closing it.
- A borrowed connection that's never returned; eventually the pool is empty and requests time out.
- The connection is held from the start of the transaction until commit or rollback.
- Fewer competing queries means less switching overhead on the database's limited cores and disk.
