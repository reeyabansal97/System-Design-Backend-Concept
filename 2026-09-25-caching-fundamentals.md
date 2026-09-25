# Day 6: Caching Fundamentals

## What problem does it solve?

Some data is expensive to get: a slow database query, a call to another service, a heavy computation. If the same data is requested again and again, fetching it fresh every time wastes time and puts load on the source.

A **cache** is a fast storage layer that keeps copies of frequently-used data so future requests can be served quickly, without going back to the slow source.

Analogy: instead of walking to the library every time you need a fact, you keep the books you use most on your desk.

## Why it's fast

Caches usually live in memory (RAM), and memory is vastly faster than disk or a network round trip. Reading a value from an in-memory cache typically takes well under a millisecond. The same data from a database query, especially one involving joins or disk reads, often takes many milliseconds.

## Key vocabulary

- **Cache hit:** the requested data is in the cache. Fast path.
- **Cache miss:** it isn't. You fetch from the source, and usually store it in the cache for next time.
- **Hit ratio:** hits ÷ total requests. Higher means the cache is doing its job.
- **TTL (time to live):** how long an entry stays valid before it expires and gets refetched.
- **Eviction:** removing entries when the cache is full. The most common policy is **LRU (Least Recently Used)**: throw out whatever hasn't been accessed for the longest time.

## The most common pattern: cache-aside

The application manages the cache itself:

```
1. Request comes in for user 42
2. Check cache for "user:42"
   - Hit  → return cached value
   - Miss → query database, store result in cache, return it
```

This is also called **lazy loading**: data only enters the cache when someone actually asks for it.

## Where caches live

| Location | Example | Notes |
|---|---|---|
| Inside your application | A `HashMap`, Caffeine | Fastest; each server instance has its own copy |
| Separate cache server | Redis, Memcached | Shared by all instances of your app |
| Browser / CDN | HTTP cache headers, CDN edge servers | Serves users without even reaching your servers |

A local, in-process cache is fastest but gets tricky with multiple servers: instance A and instance B each hold their own copy, and those copies can disagree. A shared cache like Redis avoids that, at the cost of a network hop.

## Java angle: Spring's `@Cacheable`

```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElseThrow();   // only runs on a cache miss
    }

    @CacheEvict(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);   // removes the stale cached copy
    }
}
```

The first call to `getProduct(5)` hits the database and stores the result. Later calls with `5` return the cached copy without running the method body. `@CacheEvict` removes the entry when the product changes, so nobody gets the old version.

(Same proxy rule as `@Transactional` from Day 5: calling `getProduct` from another method in the same class skips the cache.)

## The hard part: stale data

The moment you cache something, you have two copies of the truth: the source and the cache. If the source changes and the cache doesn't, users see **stale data**. Keeping them in sync is called **cache invalidation**, and it's famously one of the hardest problems in computing. Common approaches:

- **TTL:** accept that data can be stale for up to N seconds, then it refreshes automatically.
- **Evict on write:** remove the cached entry whenever the underlying data changes (like `@CacheEvict` above).

Pick based on how much staleness your use case can tolerate. A product description stale for 5 minutes is usually fine. A bank balance stale for 5 minutes is not.

## Common pitfalls

- **Caching data that changes constantly.** If it's invalidated faster than it's read, the cache just adds overhead.
- **No TTL and no eviction.** The cache grows until it runs out of memory.
- **Caching per-user data under a shared key.** User A sees user B's data. Always include the right identifiers in the cache key.
- **Treating the cache as the source of truth.** Caches can lose data at any time (restarts, evictions). The real data must always live somewhere durable.

## Interviewer follow-ups

**"What's the difference between a local cache and a distributed cache?"** A local cache lives inside each application instance: fastest, but each instance has its own copy, and copies can drift apart. A distributed cache (Redis) is shared by all instances: consistent across servers, but every read costs a network hop.

**"How do you decide what to cache?"** Data that's read often, changes rarely, and is expensive to compute or fetch. Look for high read-to-write ratios.

## Retention Check

1. What is a cache, and why is reading from it faster than reading from a database?
2. Define cache hit, cache miss, and hit ratio.
3. Walk through the cache-aside pattern for a request.
4. What does LRU eviction remove first?
5. What problem does a shared cache like Redis solve compared with a local in-process cache?
6. What is stale data, and name two ways to limit it.
7. Why would caching a bank balance with a 5-minute TTL be a bad idea?
8. Why must the cache never be treated as the source of truth?

**Key points to self-check against:**
- A fast storage layer holding copies of frequently-used data; it usually lives in memory, which is far faster than disk or network.
- Hit: data found in cache. Miss: not found. Hit ratio: hits ÷ total requests.
- Check cache → on hit return it; on miss fetch from source, store in cache, return.
- The entry that was accessed least recently.
- All app instances see the same cached copy, instead of each instance holding its own possibly-different copy.
- The cache holding an outdated copy after the source changes; limit with TTLs or by evicting on write.
- Users could see a wrong balance for minutes, which is unacceptable for financial data.
- Caches can lose entries at any time (restarts, eviction); the durable source must hold the real data.
