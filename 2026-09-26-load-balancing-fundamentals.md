# Day 7: Load Balancing Fundamentals

## What problem does it solve?

One server can only handle so many requests. When traffic grows, the usual fix is to run **several copies** of your application on several servers (horizontal scaling). But then there's a new question: when a request arrives, which server should handle it?

A **load balancer** sits in front of your servers, receives every incoming request, and forwards each one to a server. Clients only ever talk to the load balancer. They don't know or care how many servers are behind it.

```
               ┌──► Server A
Client ──► LB ─┼──► Server B
               └──► Server C
```

## What a load balancer gives you

1. **Spreads the load.** No single server gets overwhelmed while others sit idle.
2. **Availability.** If Server B crashes, the load balancer stops sending it traffic and routes to A and C. Users don't notice.
3. **Easy scaling.** Traffic spike? Add Server D behind the load balancer. Clients change nothing.

## How it knows a server is dead: health checks

The load balancer regularly pings each server (for example, an HTTP `GET /health` every few seconds). If a server fails several checks in a row, it's marked unhealthy and removed from rotation. When it starts passing again, it's added back. Spring Boot's Actuator exposes exactly this kind of endpoint (`/actuator/health`).

## Common algorithms for choosing a server

| Algorithm | How it works | Good for |
|---|---|---|
| **Round robin** | A, B, C, A, B, C, ... in order | Servers that are similar, requests that are similar |
| **Weighted round robin** | Bigger servers get proportionally more turns | Servers with different capacities |
| **Least connections** | Send to the server with the fewest active requests | Requests that take very different amounts of time |
| **IP hash** | Hash the client's IP to pick a server | Keeping the same client on the same server |

Round robin is the simplest and a sensible default. Least connections matters when some requests take 10 ms and others take 10 seconds: round robin might pile several slow requests onto one server, while least connections notices that server is busy.

## The "sticky session" problem

Recall Day 1: HTTP is stateless, so any server should be able to handle any request. But some applications store user session data (like "who's logged in") **in the memory of one server**. If the user's next request goes to a different server, that session is missing and they get logged out.

Two fixes:
- **Sticky sessions:** the load balancer pins each user to one server (e.g. by cookie or IP hash). This works, but it weakens load balancing, and if that server dies, those sessions are lost anyway.
- **Stateless servers (preferred):** keep session data out of server memory. Store it in a shared place (like Redis, from Day 6) or in the client's token (like a JWT). Then any server can handle any request, and load balancing works cleanly.

This is why "keep your servers stateless" comes up constantly in system design interviews: it's what makes horizontal scaling and load balancing simple.

## Layer 4 vs Layer 7 (brief)

- **Layer 4 (transport level):** routes based on IP address and port only. It doesn't look inside the request. Very fast.
- **Layer 7 (application level):** reads the HTTP request (URL, headers, cookies) and can route on it, e.g. send `/api/*` to API servers and `/images/*` to image servers. Smarter, slightly more overhead.

Examples you'll hear: **Nginx** and **HAProxy** (software you run), **AWS Elastic Load Balancer** (managed: ALB is Layer 7, NLB is Layer 4).

## Common pitfalls

- **The load balancer as a single point of failure.** If there's only one load balancer and it dies, everything is down. Production setups run load balancers in redundant pairs, or use a managed service that handles this.
- **Storing session state in server memory.** Breaks as soon as requests land on a different server. Keep servers stateless.
- **Health checks that lie.** A `/health` endpoint that always returns "OK" even when the database is unreachable keeps a broken server in rotation. Health checks should verify the dependencies the server actually needs.

## Interviewer follow-ups

**"What happens to in-flight requests when a server fails its health check?"** Requests already being processed on that server may fail. The load balancer only stops sending *new* requests to it. Clients (or the load balancer) typically retry, which is safe only if the operation is idempotent (Day 2: `GET`, `PUT`, `DELETE` should be).

**"Round robin or least connections?"** Round robin when requests are roughly uniform in cost. Least connections when request durations vary a lot, because it adapts to how busy each server actually is.

## Retention Check

1. What problem does a load balancer solve?
2. How does a load balancer detect and route around a dead server?
3. Describe round robin, weighted round robin, and least connections.
4. When is least connections better than round robin?
5. What is the sticky session problem, and why does it happen?
6. Why are stateless servers preferred over sticky sessions?
7. What's the difference between Layer 4 and Layer 7 load balancing?
8. How can the load balancer itself become a single point of failure, and how is that avoided?

**Key points to self-check against:**
- Distributing requests across multiple servers so no single server is overwhelmed, while adding availability and easy scaling.
- Periodic health checks; servers that fail repeatedly are removed from rotation until they pass again.
- In order; in order but bigger servers get more turns; to the server with the fewest active requests.
- When request durations vary a lot.
- Session data stored in one server's memory is missing when the user's next request goes to a different server.
- Any server can handle any request, so load balancing stays even and a server failure doesn't lose sessions.
- Layer 4 routes on IP/port only; Layer 7 reads HTTP details like URL and headers.
- If there's only one and it fails, all traffic stops; run redundant load balancers or use a managed service.
