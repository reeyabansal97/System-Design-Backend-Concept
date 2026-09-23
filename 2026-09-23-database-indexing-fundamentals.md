# Day 4: Database Indexing Fundamentals

## What problem does it solve?

Without an index, finding a row in a table means the database has to check every single row — a **full table scan**, O(n) in the number of rows. For a table with a few hundred rows, that's fine. For a table with 50 million rows, that's a query that takes seconds instead of milliseconds. An index is a separate, sorted data structure the database maintains alongside your table, specifically so it can find matching rows without scanning everything.

## How it works mechanically

Think of the analogy from a physical book: instead of reading every page to find "backend," you check the index at the back, which is alphabetically sorted, jump straight to the right entry, and it tells you exactly which page to go to.

Most relational databases implement indexes using a **B-tree** (or a close variant) — a balanced, sorted tree structure. Because it's sorted and balanced, looking up a value means comparing against a handful of nodes and following the right branch each time, similar in spirit to binary search, rather than checking every row.

```sql
CREATE INDEX idx_users_email ON users(email);
```

After this, a query like `SELECT * FROM users WHERE email = 'x@example.com'` no longer scans the whole `users` table — it walks the B-tree for `email`, finds the matching entry, and that entry points directly to the matching row(s).

## Complexity

| Operation | Without index | With index (B-tree) |
|---|---|---|
| Exact-match lookup | O(n) | O(log n) |
| Range query (`WHERE age > 30`) | O(n) | O(log n + k), where k = matching rows |
| Insert/update/delete | O(1) amortized (just append/modify) | O(log n) — the index itself must also be updated |

That last row matters: **indexes aren't free.** Every write to an indexed column also requires updating the index's B-tree structure, which is why indexing every column "just in case" is a real mistake, not just an obscure optimization detail.

## A composite index, and why column order matters

You can index multiple columns together:

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

This is efficient for queries filtering on `user_id` alone, or `user_id` AND `status` together — but **not** efficient for filtering on `status` alone. The B-tree is sorted first by `user_id`, then by `status` within each `user_id` group — so without a known `user_id`, the `status` values are scattered throughout the tree with no useful ordering to exploit. This is a very common interview question: "if you have a composite index on `(a, b)`, does a query filtering only on `b` benefit from it?" — generally no.

## Java angle

Indexing decisions happen at the database schema level, but you'll interact with them constantly through JPA/Hibernate annotations or migration scripts:

```java
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_users_email", columnList = "email")
})
public class User {
    @Id
    private Long id;

    @Column(unique = true)
    private String email;
}
```

Note: `@Column(unique = true)` actually creates its own index implicitly (databases need an index to efficiently enforce uniqueness), so you don't need to declare both a unique constraint *and* a separate index on the same column.

## Trade-offs and common pitfalls

- **Indexing every column "to be safe"** — every additional index slows down every `INSERT`/`UPDATE`/`DELETE` on that table, and consumes extra storage. Index the columns you actually query/filter/sort on, not everything.
- **Wrong column order in a composite index** — as shown above, putting the less-selective or less-commonly-filtered column first wastes the index's usefulness for many queries.
- **Assuming an index always gets used** — the query planner may choose a full table scan anyway if, say, the query would return a large fraction of the table (an index lookup plus fetching most rows can be slower than just scanning sequentially). This is why tools like `EXPLAIN` (in SQL) exist — to show you what the database actually decided to do.
- **Forgetting that indexes on low-cardinality columns are often useless** — e.g., a `boolean is_active` column only has two possible values, so an index there rarely helps much; the database ends up needing to check a large fraction of the index entries anyway.

## Interviewer follow-ups

**"Why not just index every column?"** — Every index adds write overhead (every insert/update must also update the index) and storage overhead, with no benefit for columns you never filter/sort/join on. Indexing is a deliberate trade-off between read speed and write speed/storage, not a free win.

**"What's the difference between a clustered and non-clustered index?"** (if it comes up) — A clustered index determines the actual physical order rows are stored on disk (a table can have at most one), while a non-clustered index is a separate structure that points back to the row's location — you can have several non-clustered indexes per table.

## Retention Check

1. Why does a full table scan cost O(n), and what data structure does an index typically use to avoid that?
2. Why does a lookup using a B-tree index cost O(log n) instead of O(n)?
3. Do inserts get slower or faster on a heavily-indexed table, and why?
4. For a composite index on `(user_id, status)`, would a query filtering only by `status` benefit from it? Why or why not?
5. Why might the query planner choose a full table scan even when an index exists on the filtered column?
6. Why is indexing a low-cardinality column (like a boolean) often not very useful?
7. What does `@Column(unique = true)` implicitly create, in terms of indexing?
8. What's the fundamental trade-off you're making every time you add an index?

**Key points to self-check against:**
- Because every row must be checked one by one with no shortcuts; a B-tree (sorted, balanced tree) is the typical structure used to avoid this.
- Because the sorted, balanced structure lets you eliminate large portions of the search space at each step, similar to binary search, instead of checking every entry.
- Slower — every write must also update the index's B-tree structure, not just the underlying row.
- No (generally) — the index is sorted first by `user_id`, so without filtering on `user_id`, the `status` values have no useful grouping/ordering the index can exploit.
- If the query would return a large fraction of the table anyway, a sequential scan can be faster than jumping around via an index lookup for each matching row.
- Because with only a couple of distinct values, the index can't narrow the search down much — a large fraction of rows share the same value.
- A separate index on that column (needed for the database to efficiently enforce uniqueness).
- Faster reads (via indexed lookups) in exchange for slower writes and extra storage — you index what you actually query, not everything.
