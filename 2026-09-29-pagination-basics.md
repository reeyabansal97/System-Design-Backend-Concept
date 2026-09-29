# Day 9: Pagination Basics

(Skipped "API Versioning" as a standalone day: Day 2's REST read already covered it, with URL versioning like `/v1/orders` versus header versioning.)

## What problem does it solve?

`GET /orders` sounds harmless, until the table has 10 million rows. Returning all of them would:
- take a long time to query,
- use huge amounts of server memory to build the response,
- send a massive payload over the network,
- and swamp the client, which could only display about 20 rows at once anyway.

**Pagination** returns data in small chunks (**pages**), and gives the client a way to request the next chunk.

## Approach 1: Offset pagination

The client says "skip this many rows, then give me this many":

```
GET /orders?page=3&size=20
```

```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 40;
```

(page 3, with pages starting at 1 → skip the first 40 rows → return rows 41-60)

**Pros:** simple, and lets users jump straight to "page 50."

**Cons:**
1. **Gets slow deep into the data.** `OFFSET 1000000` doesn't magically jump ahead. The database still walks past a million rows just to throw them away. Page 1 is fast; page 50,000 is slow.
2. **Rows can shift under you.** If a new order is inserted while a user is paging, every row slides down by one. The user can see the same row twice on consecutive pages, or skip one entirely.

## Approach 2: Cursor (keyset) pagination

Instead of "skip N rows," the client says "give me rows **after** the last one I saw":

```
GET /orders?size=20&after=2026-09-28T10:15:00Z
```

```sql
SELECT * FROM orders
WHERE created_at < '2026-09-28T10:15:00Z'
ORDER BY created_at DESC
LIMIT 20;
```

The response includes a **cursor**: a marker for where the next page starts, usually based on the last row's sort value. The client passes it back for the next page.

**Pros:**
1. **Consistently fast.** With an index on `created_at` (Day 4), the database jumps straight to the starting point, whether it's the first page or the millionth.
2. **Stable.** New inserts don't shift your position, so no duplicates or skipped rows.

**Cons:** no "jump to page 50." You can only move forward (or backward) from where you are. That's fine for infinite scroll and feeds, but less so for numbered page links.

## The tie-breaker problem

What if two orders have exactly the same `created_at`? A cursor of just that timestamp could skip or repeat rows. The fix is to make the sort order **unique**: sort by `(created_at, id)`, and use both in the cursor:

```sql
WHERE (created_at, id) < ('2026-09-28T10:15:00Z', 9876)
ORDER BY created_at DESC, id DESC
```

The row-value comparison `(a, b) < (x, y)` is supported by PostgreSQL and MySQL. On databases without it, spell it out as `created_at < x OR (created_at = x AND id < y)`.

## Which to use

| | Offset | Cursor |
|---|---|---|
| Speed on deep pages | Degrades | Stays fast |
| Stable when data changes | No | Yes |
| Jump to any page number | Yes | No |
| Good for | Admin tables, small or static datasets | Feeds, infinite scroll, large or fast-changing data |

## Java angle: Spring Data

Offset pagination is built in:

```java
@GetMapping("/orders")
public Page<Order> getOrders(@RequestParam int page, @RequestParam int size) {
    Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
    return orderRepository.findAll(pageable);
}
```

`PageRequest.of` is **zero-based**: page `0` is the first page. `Page<Order>` also includes the total count, which Spring gets by running a separate `COUNT(*)` query. On huge tables, that count query can be expensive on its own. If you don't need the total, return a `Slice<Order>` instead, which only tells you whether there's a next page.

Cursor pagination is written as a normal query method:

```java
List<Order> findByCreatedAtBeforeOrderByCreatedAtDesc(Instant cursor, Pageable pageable);
```

(Pass `PageRequest.of(0, size)` to limit it to one page.)

## Common pitfalls

- **No maximum page size.** A client asking for `size=1000000` defeats the point. Always cap it on the server.
- **Paginating without a stable sort.** Without `ORDER BY`, the database can return rows in any order, so pages can overlap or skip rows.
- **Deep offset pagination on large tables.** It works in testing with 100 rows and crawls in production with 10 million.
- **Assuming the total count is free.** `COUNT(*)` on a big table can cost more than fetching the page itself.

## Interviewer follow-ups

**"Why is `OFFSET 1000000` slow?"** The database still has to read and discard the first million rows before returning anything. The cost grows with how deep you go.

**"How would you paginate a social media feed?"** With cursor pagination: new posts are constantly being added (offset would show duplicates), users scroll forward instead of jumping to page numbers, and it stays fast however far they scroll.

## Retention Check

1. What goes wrong if an API returns every row at once?
2. How does offset pagination translate to SQL?
3. Why does offset pagination get slower on deeper pages?
4. How can new inserts cause duplicates or skipped rows with offset pagination?
5. What does a cursor represent, and what does the client do with it?
6. Why is cursor pagination consistently fast?
7. Why add `id` as a tie-breaker in the sort?
8. When would you still choose offset over cursor pagination?

**Key points to self-check against:**
- Slow queries, heavy server memory use, huge payloads, and more data than the client can show.
- `LIMIT size OFFSET (page - 1) * size`, with pages counted from 1.
- The database reads and discards every skipped row, so cost grows with the offset.
- A new row shifts everything down, so the next page repeats a row the user already saw, or skips one.
- A marker for the last row seen; the client sends it back to fetch the next page.
- It uses an index to jump straight to the starting point instead of skipping rows.
- Rows with equal sort values could otherwise be skipped or repeated; a unique sort order makes the cursor exact.
- When users need to jump to specific page numbers, and the dataset is small or rarely changes.
