# Day 8: Authentication vs Authorization

(Skipped "Horizontal vs Vertical Scaling" as a standalone day: Day 7 on load balancing already covered why and how you scale horizontally. Quick recap: vertical = a bigger machine, simple but capped and a single point of failure; horizontal = more machines behind a load balancer, which needs stateless servers.)

## The two questions every secure request must answer

- **Authentication (AuthN): "Who are you?"** Proving identity: username + password, a fingerprint, a login with Google.
- **Authorization (AuthZ): "What are you allowed to do?"** Checking permissions once identity is known.

Airport analogy: showing your passport at security is authentication. Your boarding pass letting you into *one specific gate and seat* is authorization. Passing security doesn't let you board any plane you like.

**Authentication always comes first.** You can't decide what someone is allowed to do until you know who they are.

## The status codes (connects to Day 1)

- **401 Unauthorized:** actually means *unauthenticated*. "I don't know who you are." Missing or invalid credentials. (The name is a historical misnomer.)
- **403 Forbidden:** "I know who you are, and you're not allowed to do this."

Mixing these up is a common interview and code-review catch.

## How authentication works across stateless requests

HTTP is stateless (Day 1), so the server doesn't remember that you logged in a minute ago. Every request must carry proof of identity. Two main approaches:

### 1. Session-based

1. User logs in with a username and password.
2. Server verifies them, creates a **session** (stored on the server side, e.g. in Redis), and sends back a **session ID** in a cookie.
3. Every later request sends the cookie; the server looks up the session ID to find the user.

Easy to revoke (delete the session), but the server must store and look up sessions. With several servers, that store must be shared (Day 7's sticky-session problem).

### 2. Token-based (JWT)

1. User logs in.
2. Server creates a **JWT (JSON Web Token)** containing claims like user ID and roles, **signs** it with a secret key, and sends it back.
3. Every later request sends it in a header: `Authorization: Bearer <token>`.
4. Any server verifies the **signature**. If it's valid, it trusts the claims inside, with no database or session lookup needed.

A JWT has three parts separated by dots: `header.payload.signature`.

**Critical point:** the payload is only **Base64-encoded, not encrypted**. Anyone holding the token can decode and read it. The signature only proves nobody **tampered** with it. So never put secrets (passwords, card numbers) inside a JWT.

| | Sessions | JWT |
|---|---|---|
| State stored on server | Yes (session store) | No (self-contained token) |
| Scaling across servers | Needs a shared session store | Any server can verify |
| Revoking a login | Easy: delete the session | Hard: the token is valid until it expires |
| Typical fix for revocation | n/a | Short expiry + refresh tokens |

## Authorization: two common models

- **Role-based (RBAC):** users have roles (`ADMIN`, `USER`), and roles grant permissions. Simple and very common.
- **Permission/attribute-based:** finer-grained rules, like "a user can edit an order only if they own it." Often layered on top of roles.

A classic bug: checking the role but **not ownership**. User 5 is a valid `USER`, so they're allowed to call `GET /orders/{id}`. But if the code never checks that order 42 belongs to user 5, they can read anyone's orders by changing the ID. This is called **IDOR (Insecure Direct Object Reference)**, and it's one of the most common real-world API security flaws.

## Java angle: Spring Security

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @GetMapping("/{id}")
    @PreAuthorize("hasRole('USER')")                 // authorization: role check
    public Order getOrder(@PathVariable Long id, Authentication auth) {
        Order order = orderService.findById(id);
        if (!order.getOwnerId().equals(auth.getName())) {   // authorization: ownership check
            throw new AccessDeniedException("Not your order");   // → 403
        }
        return order;
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")                // only admins can delete
    public void deleteOrder(@PathVariable Long id) {
        orderService.delete(id);
    }
}
```

Spring Security handles authentication in its filter chain *before* your controller runs. Requests with bad credentials are rejected with 401 and never reach your code. `@PreAuthorize` then enforces authorization. (Note: `@PreAuthorize` requires method security to be enabled, e.g. `@EnableMethodSecurity`.)

**Always store passwords hashed** with a slow, salted algorithm like **bcrypt** (Spring's `BCryptPasswordEncoder`). Never store plain text, and never use a fast hash like MD5 or SHA-256 alone for passwords.

## Common pitfalls

- **Returning 401 when you mean 403**, or vice versa.
- **Putting sensitive data in a JWT payload.** It's readable by anyone who has the token.
- **Checking roles but not ownership (IDOR).** A valid user reads other users' data by changing an ID.
- **Long-lived JWTs with no way to revoke them.** A stolen token stays usable until it expires. Keep access tokens short-lived.
- **Doing authorization only in the frontend.** Hiding the "Delete" button is not security. The backend must enforce every rule, because anyone can call your API directly.

## Interviewer follow-ups

**"Sessions or JWT?"** Sessions when you need easy revocation and you already have shared storage. JWT when you want stateless verification across many services. Many systems use short-lived JWTs with refresh tokens to get some of both.

**"Is a JWT encrypted?"** No, a standard signed JWT is only encoded. The signature guarantees integrity (it hasn't been modified), not confidentiality.

## Retention Check

1. What question does authentication answer, and what question does authorization answer?
2. Which one always happens first, and why?
3. When should an API return 401 versus 403?
4. How does session-based authentication identify the user on later requests?
5. How does a server verify a JWT without a database lookup?
6. Why must you never put a password inside a JWT payload?
7. What is IDOR, and how do you prevent it?
8. Why isn't hiding a button in the frontend enough for authorization?

**Key points to self-check against:**
- AuthN: who are you? AuthZ: what are you allowed to do?
- Authentication, because permissions can't be checked until identity is known.
- 401: missing or invalid credentials (unknown user). 403: known user without permission.
- The client sends a session ID cookie; the server looks it up in its session store.
- It checks the token's signature with the secret key; if valid, it trusts the claims inside.
- The payload is only Base64-encoded, so anyone with the token can read it.
- Accessing another user's data by changing an ID in the request; always check that the resource belongs to the requesting user.
- Anyone can call the API directly, so the backend must enforce every permission.
