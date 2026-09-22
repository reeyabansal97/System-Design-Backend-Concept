# Day 3: SQL vs NoSQL Databases — The Basics

(Note: skipped ahead past "HTTP methods/status codes/statelessness" since Days 1-2 already covered that ground thoroughly while explaining HTTP and REST — moving to fresh territory instead.)

## What problem are we solving?

Every backend application needs to persist data somewhere durable — user accounts, orders, product catalogs. The database you choose shapes almost everything downstream: how you model data, how you scale, and what kinds of bugs are even possible. The first major fork in the road is: **relational (SQL) vs non-relational (NoSQL).**

## SQL (relational) databases, in plain terms

Data lives in **tables** — think of a spreadsheet: fixed columns (with defined types), and rows of actual data. Tables relate to each other through **foreign keys** — e.g., an `orders` table might have a `user_id` column pointing back to a row in the `users` table.

```sql
SELECT orders.id, orders.total, users.name
FROM orders
JOIN users ON orders.user_id = users.id
WHERE orders.status = 'shipped';
```

This `JOIN` is the defining feature of relational databases — you can combine data from multiple tables in a single query, and the database engine figures out how to do it efficiently (often using indexes, from your earlier reading if you've covered that topic).

SQL databases (PostgreSQL, MySQL, Oracle, SQL Server) generally guarantee **ACID transactions** — if you've encountered that term, it means a set of operations either all succeed together or all fail together, with strong consistency guarantees.

## NoSQL databases, in plain terms

"NoSQL" isn't one thing — it's an umbrella term for several different, non-relational data models:

| Type | Example | Data looks like |
|---|---|---|
| Document | MongoDB | JSON-like documents, each can have a different shape |
| Key-value | Redis | Simple `key → value` lookups, extremely fast |
| Wide-column | Cassandra | Rows with flexible, sparse columns, built for massive scale |
| Graph | Neo4j | Nodes and edges, optimized for relationship-heavy queries |

The common thread: **no fixed schema enforced by the database**, and generally **no cross-table joins** — you either store related data together in one document, or you do the "joining" yourself in application code.

## The core trade-off

SQL trades flexibility for **structure and strong consistency** — good when your data has clear relationships and you need to guarantee correctness (e.g., financial transactions where a payment and an inventory decrement must both happen or neither does).

NoSQL trades some of that structure for **flexibility and horizontal scalability** — good when your data doesn't have a fixed shape (different users might have wildly different profile fields), or when you need to scale writes across many machines more easily than traditional relational databases typically allow.

## A common misconception

**"NoSQL means you don't need to design your schema."** This is backwards — with NoSQL, you actually need to think *harder* upfront about your data access patterns, because you usually can't cheaply join data after the fact the way SQL lets you. In document databases especially, a common technique is deliberately duplicating data across documents (denormalization) so that a single read gets everything you need, without a join.

## Java angle

For SQL, the common stack is **JDBC** at the lowest level, or **Spring Data JPA** for a higher-level, entity-based approach:

```java
@Entity
public class Order {
    @Id
    private Long id;
    private BigDecimal total;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;
}

public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByStatus(String status);
}
```

For a NoSQL document store, **Spring Data MongoDB** looks structurally similar but stores whole documents instead of normalized rows:

```java
@Document(collection = "orders")
public class Order {
    @Id
    private String id;
    private BigDecimal total;
    private User user;   // the whole user object embedded directly, not a foreign key reference
}
```

Notice: in the MongoDB version, `user` is embedded directly inside the order document — no join needed to read an order with its user's details, because the data was deliberately duplicated at write time.

## Trade-offs and common pitfalls

- **Choosing NoSQL just because it sounds "modern" or "scalable"** without actually having a scaling or flexibility problem SQL couldn't solve — adds real complexity (denormalization, harder ad-hoc queries) for no real benefit.
- **Forcing deeply relational data into a document store** — if your data has many interconnected relationships that change independently, forcing it into embedded documents leads to painful, error-prone duplicate-data updates.
- **Assuming NoSQL is always "eventually consistent" and SQL is always "strongly consistent"** — this used to be a fairly reliable rule of thumb, but many modern databases blur this line (e.g., some NoSQL stores now offer strong consistency options); don't treat it as an absolute.

## Interviewer follow-ups

**"When would you choose NoSQL over SQL for a new project?"** — When the data doesn't have a fixed, predictable schema (e.g., user-generated content with varying fields), when you need to scale writes horizontally across many servers more easily than traditional relational setups allow, or when your access pattern is dominated by simple key-based lookups rather than complex relational queries.

**"What is denormalization, and why is it common in NoSQL design?"** — Denormalization means intentionally duplicating data (instead of referencing it once and joining) so that a single read returns everything needed without combining multiple records. It's common in NoSQL because joins are often expensive or unavailable, so trading storage space and write complexity for read simplicity is usually worth it.

## Retention Check

1. What is the defining structural difference between how SQL and NoSQL databases organize data?
2. What does a `JOIN` let you do in a relational database?
3. Name the four common categories of NoSQL databases mentioned, and one example of each.
4. Why is "NoSQL means no schema design needed" a misconception?
5. What is denormalization, and why is it more common in NoSQL than SQL?
6. What kind of application would benefit most from strong ACID guarantees?
7. In the Spring Data MongoDB example, why is the whole `User` object embedded directly in the `Order` document?
8. What's a realistic risk of forcing deeply relational data into a document-store model?

**Key points to self-check against:**
- SQL: fixed-schema tables with relationships via foreign keys. NoSQL: various non-relational models (document, key-value, wide-column, graph), generally without joins.
- Combine related data from multiple tables in a single query.
- Document (MongoDB), key-value (Redis), wide-column (Cassandra), graph (Neo4j).
- Because without joins, you must design your data access patterns carefully upfront — you can't cheaply combine data after the fact the way SQL allows.
- Deliberately duplicating data across records/documents so a single read gets everything needed, without a join; more common in NoSQL because joins are expensive or unavailable there.
- Something requiring strong correctness guarantees across multiple operations, like financial transactions (payment + inventory update must both succeed or both fail).
- So reading an order returns the user's details in one read, with no join required — the data was duplicated at write time for read efficiency.
- Painful, error-prone duplicate-data updates when the same underlying entity is embedded in many places and needs to change consistently everywhere.
