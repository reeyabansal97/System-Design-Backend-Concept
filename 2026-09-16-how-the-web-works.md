# Day 1: How the Web Works — Client-Server & HTTP Basics

## What problem are we solving?

Every backend job eventually comes down to this: some program (a browser, mobile app, or another service) needs to ask your server for something, or tell it to do something. Understanding *exactly* how that conversation happens is the foundation everything else in backend engineering builds on.

## Client-server model, in plain terms

- **Client** — the thing that *asks* for something (a browser, a mobile app, another backend service calling yours).
- **Server** — the thing that *listens* and responds (your backend application).

The client always initiates. Your server sits there, listening on a port (commonly 80 for HTTP, 443 for HTTPS), waiting for requests to come in.

## What actually travels over the network: HTTP

HTTP (HyperText Transfer Protocol) is the agreed-upon *format* for these client-server conversations. A request looks roughly like this:

```
GET /users/42 HTTP/1.1
Host: api.example.com
Authorization: Bearer abc123
```

And the server sends back a response:

```
HTTP/1.1 200 OK
Content-Type: application/json

{"id": 42, "name": "Reeya"}
```

Breaking this down:
- **Method** (`GET`) — what kind of action the client wants.
- **Path** (`/users/42`) — which resource it's asking about.
- **Headers** (`Authorization`, `Content-Type`) — metadata about the request/response (who's asking, what format the data is in, etc.).
- **Body** — the actual data being sent (present in requests like `POST`, and in most responses).
- **Status code** (`200 OK`) — did it work, and if not, what kind of problem was it?

## The main HTTP methods

| Method | Meaning | Example |
|---|---|---|
| GET | Read data, no side effects | Fetch a user's profile |
| POST | Create something new | Create a new order |
| PUT | Replace an entire resource | Replace a user's full profile |
| PATCH | Partially update a resource | Update just a user's email |
| DELETE | Remove a resource | Delete an order |

## Common status codes, grouped by what they mean

- **2xx (Success)** — `200 OK`, `201 Created`
- **3xx (Redirection)** — `301 Moved Permanently`
- **4xx (Client's fault)** — `400 Bad Request` (malformed input), `401 Unauthorized` (not logged in), `403 Forbidden` (logged in, but not allowed), `404 Not Found`
- **5xx (Server's fault)** — `500 Internal Server Error`

Getting this right matters in real jobs: a `404` versus `500` tells whoever's debugging (or an automated alert system) *where* to look for the problem — client-side mistake, or a bug in your service.

## What "stateless" means, and why it matters

HTTP is **stateless** — the server doesn't remember anything about the client between requests, by default. Every request must carry everything the server needs to handle it (like an auth token). This is a deliberate design choice: it means any server instance can handle any request, which is what makes it easy to run multiple copies of your backend behind a load balancer — none of them need to "remember" which client they talked to last.

## A simple Java angle

In a Spring Boot controller, this basic conversation looks like:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        if (user == null) {
            return ResponseEntity.status(404).build();
        }
        return ResponseEntity.ok(user); // sends 200 + JSON body
    }
}
```

`@GetMapping` ties this method to `GET /users/{id}`. Spring handles turning the HTTP request into a Java method call, and turning your return value back into an HTTP response — but under the hood, it's still just the request/response exchange described above.

## Common pitfalls

- Using `GET` for an action that changes data (like deleting something) — breaks the expectation that `GET` has no side effects, and can cause accidental deletions (e.g., a web crawler following links).
- Returning `200 OK` even when something failed, instead of an appropriate error status — makes debugging and client error-handling much harder.
- Forgetting that "stateless" means you can't rely on server memory between requests — session data needs to be passed (token) or stored somewhere shared (database, cache), not kept in a single server's memory.

## Retention Check

1. In the client-server model, which side initiates the conversation?
2. What's the difference between `PUT` and `PATCH`?
3. What does a `404` status code mean, versus a `500`?
4. Why is HTTP described as "stateless," and what does that require of every request?
5. Why shouldn't `GET` requests have side effects like deleting data?
6. What's the difference between a 4xx and a 5xx status code, conceptually?
7. What port does HTTPS typically use?
8. Why does statelessness make it easier to run multiple server instances behind a load balancer?

**Key points to self-check against:**
- The client always initiates; the server listens and responds.
- `PUT` replaces the whole resource; `PATCH` updates part of it.
- `404` = resource not found (client asked for something that doesn't exist); `500` = something broke on the server's side.
- The server keeps no memory of past requests, so each request must carry all the context it needs (e.g., an auth token).
- Because `GET` is expected to be safe/idempotent — automated tools (crawlers, prefetchers) may call it without the user intending an action.
- 4xx = the client did something wrong (bad input, not authorized); 5xx = the server itself failed.
- 443.
- Any server instance can handle any request since none of them need to remember prior client state.
