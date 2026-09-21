# Day 2: REST API Design Fundamentals

## What problem does REST solve?

Once you understand basic HTTP (Day 1), the next question is: how should you *organize* a set of endpoints so they're predictable and easy for other developers (or your future self) to use? REST (Representational State Transfer) is a set of conventions for designing HTTP APIs around **resources**, so that anyone familiar with REST conventions can guess how your API works without reading extensive docs.

## The core idea: everything is a resource

A "resource" is a noun — a thing your system manages: a user, an order, a product. REST says: **name your URLs after resources (nouns), and use HTTP methods to say what you want to do to them (verbs)**.

```
GET    /orders         → list orders
GET    /orders/42      → get order 42
POST   /orders         → create a new order
PUT    /orders/42      → replace order 42 entirely
PATCH  /orders/42      → partially update order 42
DELETE /orders/42      → delete order 42
```

Notice the URL never contains a verb like `/getOrder` or `/createOrder` — the HTTP method already *is* the verb. This is the single most common REST mistake to watch for, in your own APIs and in interviews: **verbs in the URL path are a sign the design isn't RESTful.**

## How this works in practice: nesting resources

Resources often relate to each other. A common pattern is nesting:

```
GET /users/7/orders        → all orders belonging to user 7
GET /users/7/orders/42     → order 42, belonging to user 7
```

For filtering, sorting, and pagination — things that aren't really a *different resource*, just a different view of the same one — use query parameters instead of new paths:

```
GET /orders?status=shipped&sort=date&page=2
```

## Statelessness, revisited

Just like plain HTTP (Day 1), a RESTful API must be stateless: each request carries everything needed to process it (auth token, any filters), and the server doesn't rely on remembering previous requests from that client. This is what lets you scale a REST API horizontally — any server instance, behind any load balancer, can serve any request.

## Java angle: a small REST controller

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @GetMapping
    public List<Order> listOrders(@RequestParam(required = false) String status) {
        return orderService.findAll(status);
    }

    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(@PathVariable Long id) {
        return orderService.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody Order order) {
        Order created = orderService.create(order);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
        orderService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

Notice: `createOrder` returns `201 Created` (not `200 OK`) — REST convention says creation should signal that a new resource now exists. `deleteOrder` returns `204 No Content` — success, but there's nothing meaningful to send back.

## Trade-offs and common pitfalls

- **Verbs in URLs** (`/getUser`, `/createOrder`) — breaks the resource-oriented convention; the method (`GET`, `POST`) should already convey the action.
- **Wrong status codes** — returning `200 OK` for a creation (should be `201`), or `200 OK` with an error message in the body instead of an actual `4xx`/`5xx` — makes the API harder to use programmatically.
- **Over-nesting resources** — `/users/7/orders/42/items/3/reviews/9` becomes unwieldy; usually 2 levels of nesting is a practical limit.
- **Ignoring idempotency expectations** — `PUT` and `DELETE` are expected to be idempotent (calling them multiple times has the same effect as calling once); if your `PUT` implementation behaves differently on repeat calls (e.g., appending instead of replacing), it violates the convention and will surprise clients that retry on network failure.

## Interviewer follow-ups

**"How would you version this API?"** — Two common approaches: put the version in the URL (`/v1/orders`) — simple and visible, but "pollutes" the URL; or put it in a header (`Accept: application/vnd.myapi.v1+json`) — cleaner URLs, but less discoverable. URL versioning is far more common in practice because of its simplicity.

**"What makes an API 'RESTful' rather than just 'an HTTP API'?"** — Being RESTful specifically means: resource-oriented URLs, statelessness, using HTTP methods and status codes correctly, and a uniform interface (same conventions everywhere in the API). Plenty of APIs use HTTP without honoring these conventions (e.g., RPC-style APIs with a single `POST /doAction` endpoint) — they're HTTP APIs, just not REST APIs.

## Retention Check

1. Why shouldn't a REST URL contain a verb like `/createOrder`?
2. What status code should a successful `POST` that creates a resource return?
3. What status code fits a successful `DELETE` with no body to return?
4. How do query parameters differ from path segments in REST URL design?
5. Why are `PUT` and `DELETE` expected to be idempotent?
6. What are the two common ways to version a REST API, and what's the trade-off between them?
7. Why is over-nesting resource URLs (many levels deep) considered a design smell?
8. What makes an HTTP-based API specifically "RESTful," as opposed to just "an API that uses HTTP"?

**Key points to self-check against:**
- The HTTP method already conveys the action; a verb in the path duplicates and contradicts that.
- `201 Created`.
- `204 No Content`.
- Path segments identify *which* resource; query parameters describe filtering/sorting/pagination of that resource, not a different resource.
- So clients can safely retry them after a network failure without unintended side effects (like double-creating something).
- URL versioning (`/v1/...`, simple, visible) vs header versioning (cleaner URLs, less discoverable).
- It becomes hard to construct, read, and maintain; usually 1-2 levels of nesting is the practical limit.
- Resource-oriented URLs, statelessness, correct use of HTTP methods/status codes, and a uniform interface throughout.
