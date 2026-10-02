# Day 11: Monoliths vs Microservices

## The question

How do you split up a large application? As **one deployable unit**, or as **many small independent services**? This is one of the most common system design discussions, and the honest answer is "it depends." Interviewers want to hear the trade-offs, not a slogan.

## Monolith

All features live in **one codebase, built and deployed as one application**, usually sharing one database.

```
┌────────────────────────────────┐
│          One application        │
│  Users | Orders | Payments | ...│
└───────────────┬────────────────┘
                │
          One database
```

A monolith can still be well organized inside: separate packages, layers, and modules. "Monolith" describes how it's **deployed**, not that the code is messy.

**Pros**
- **Simple to build, test, and deploy.** One app, one pipeline, one thing to run locally.
- **Fast internal calls.** Features call each other as plain method calls, in the same process.
- **Easy transactions.** One database means one ACID transaction (Day 5) can cover orders and payments together.
- **Easy debugging.** One log, one stack trace.

**Cons**
- **Everything deploys together.** A one-line fix in Payments means redeploying the whole app.
- **Scales as a whole.** If only search is under heavy load, you still have to scale every feature with it.
- **One failure can take it all down.** A memory leak in one module can crash the entire app.
- **Harder for many teams.** As the codebase and team grow, people step on each other's changes.

## Microservices

The application is split into **small, independent services**, each owning one business capability, deployed separately, and ideally with **its own database**. Services talk over the network (HTTP/REST from Day 2, or messaging).

```
 Users svc ──► Users DB
 Orders svc ──► Orders DB
 Payments svc ──► Payments DB
      ▲   ▲   ▲
      └───┴───┴── API Gateway ◄── Clients
```

**Pros**
- **Independent deployment.** Payments can ship 10 times a day without touching Orders.
- **Independent scaling.** Scale only the busy service (Day 7: put more instances of *just that service* behind a load balancer).
- **Fault isolation.** If Recommendations crashes, checkout can keep working, if the system is designed for it.
- **Team autonomy.** Each team owns a service end to end, and can even pick different technology.

**Cons**
- **Network calls replace method calls.** They're slower, and they can fail, time out, or arrive twice. Every call needs timeouts and retry handling.
- **No easy cross-service transactions.** Orders and Payments have separate databases, so one ACID transaction can't span both. You need patterns like **sagas** (a later topic) and must accept **eventual consistency**.
- **Operational complexity.** Many deployments, service discovery, centralized logging, distributed tracing, monitoring.
- **Harder debugging.** One user request may hop through 6 services. You need a **correlation ID** passed along to trace it.

## The "distributed monolith" trap

The worst of both worlds: services that are deployed separately, but are so tightly coupled that they must be changed and deployed **together**, or that all share one database. You pay all the network and operational costs of microservices, and get none of the independence.

## How to choose

| Situation | Usually better |
|---|---|
| Small team, new product, requirements still changing | Monolith |
| Many teams needing to ship independently | Microservices |
| Parts of the system with very different scaling needs | Microservices (at least for those parts) |
| Strong need for transactions across features | Monolith (or keep those features in one service) |

A widely given piece of advice is **"start with a monolith, extract services later"** once boundaries are clear. Splitting too early often draws the service boundaries in the wrong places, and moving a boundary between services is much harder than moving code between packages.

A popular middle ground is the **modular monolith**: one deployment, but with strict module boundaries inside, so extracting a service later is easier.

## Java angle

Spring Boot works for both. A monolith is one Spring Boot app with many packages. In microservices, each service is its own Spring Boot app, often with:
- **Spring Cloud Gateway** as the API gateway, a single entry point for clients
- an HTTP client like **`RestClient`** or **OpenFeign** for service-to-service calls
- **Resilience4j** for timeouts, retries, and circuit breakers

## Common pitfalls

- **Choosing microservices because they're fashionable.** They solve organizational and scaling problems. If you don't have those problems, you just pay the cost.
- **Sharing one database across services.** It couples them tightly. A schema change in one breaks the others.
- **Ignoring network failure.** Calling another service without a timeout can hang your service when the other one is slow.
- **Services that are too small.** A "service" for every class creates endless network chatter.

## Interviewer follow-ups

**"When would you choose a monolith over microservices?"** Small team, early product, evolving requirements, or a strong need for transactions across features. A monolith is simpler to build, test, deploy, and debug.

**"How do you keep data consistent across microservices without a shared transaction?"** Accept eventual consistency, and use patterns like sagas: a sequence of local transactions, each with a compensating action that undoes it if a later step fails.

## Retention Check

1. What defines a monolith: how the code is organized, or how it's deployed?
2. Name two advantages of a monolith.
3. Name two advantages of microservices.
4. Why is a transaction across Orders and Payments harder in microservices?
5. What new failure modes appear when method calls become network calls?
6. What is a distributed monolith?
7. Why is "start with a monolith" common advice?
8. What is a correlation ID for?

**Key points to self-check against:**
- How it's deployed: one unit; the code inside can still be well modularized.
- Any two of: simple to build/test/deploy, fast in-process calls, easy ACID transactions, easy debugging.
- Any two of: independent deployment, independent scaling, fault isolation, team autonomy.
- Each service has its own database, so one ACID transaction can't span both.
- Calls can be slow, time out, fail, or be delivered twice.
- Services deployed separately but so coupled they must change together, or that share a database.
- Early on, boundaries are unclear, and moving a boundary between services is much harder than inside one codebase.
- Tracing a single request as it passes through many services.
