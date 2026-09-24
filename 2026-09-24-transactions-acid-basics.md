# Day 5: Database Transactions & ACID Basics

## What problem does it solve?

Picture a bank transfer: move ₹500 from account A to account B. That's two separate writes:

1. Subtract 500 from A
2. Add 500 to B

Now imagine the server crashes between step 1 and step 2. The ₹500 has left A but never arrived in B. Money has vanished.

A **transaction** groups several operations into one unit that either **fully happens or doesn't happen at all**. ACID is the set of four guarantees a database makes about transactions.

## The four properties

**A — Atomicity: all or nothing.**
Either every operation in the transaction takes effect, or none do. If step 2 fails, the database undoes step 1 (this undo is called a *rollback*). The half-finished state is never left behind.

**C — Consistency: rules are never broken.**
The database moves from one valid state to another valid state. Any rules you've defined (e.g. "balance can't go negative", "every order must reference an existing user") hold before and after the transaction. If a transaction would break a rule, it's rejected.

**I — Isolation: concurrent transactions don't see each other's half-done work.**
If two transfers run at the same time, each behaves as if it were running alone. Transaction X shouldn't read B's balance while transaction Y is halfway through updating it.

**D — Durability: once committed, it stays committed.**
After the database says "committed," the change survives a crash or power loss. Databases achieve this by writing changes to a log on disk *before* confirming success.

## How it looks in SQL

```sql
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 'A';
UPDATE accounts SET balance = balance + 500 WHERE id = 'B';
COMMIT;
```

If anything fails between `BEGIN` and `COMMIT`, you (or the database) issue `ROLLBACK`, and both updates are discarded.

## A preview of isolation levels

Full isolation is expensive, because it forces transactions to wait on each other. So databases offer **isolation levels** that trade some isolation for speed. You'll hear names like *Read Committed*, *Repeatable Read*, and *Serializable*. Each level allows or prevents specific anomalies (like reading data another transaction hasn't committed yet). This gets its own deep-dive later. For now, remember: "Isolation" in ACID is a dial, not an on/off switch.

## Java angle: `@Transactional` in Spring

```java
@Service
public class TransferService {

    @Transactional
    public void transfer(String fromId, String toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
        // if an unchecked exception is thrown anywhere above, both changes roll back
    }
}
```

Spring wraps the method: it opens a transaction before the method runs, commits when it returns normally, and rolls back if a runtime exception escapes.

## Common pitfalls (these come up a lot in real Spring projects)

- **Self-invocation skips `@Transactional`.** If another method *in the same class* calls `transfer(...)` directly, no transaction is started. Spring applies `@Transactional` through a proxy object, and an internal `this.transfer()` call bypasses the proxy.
- **Checked exceptions don't trigger rollback by default.** Spring only rolls back on unchecked exceptions (`RuntimeException` and `Error`). Throw a checked exception and the transaction commits anyway, unless you set `@Transactional(rollbackFor = Exception.class)`.
- **Long-running transactions.** Keeping a transaction open while calling a slow external API holds database locks the whole time, blocking other requests. Keep transactions short and do slow work outside them.
- **Using `double` or `float` for money.** Not strictly a transaction issue, but it shows up in every transfer example: floating-point types can't represent values like `0.1` exactly. Use `BigDecimal`.

## Interviewer follow-ups

**"What happens if the database crashes right after it says COMMIT?"** Durability guarantees the change survives. The database wrote the change to a durable log on disk before acknowledging the commit, and replays that log on restart.

**"Why doesn't `@Transactional` work when I call the method from inside the same class?"** Spring implements it with a proxy that sits in front of your bean. External calls go through the proxy, which starts the transaction. An internal `this.method()` call goes straight to the real object and skips the proxy entirely.

## Retention Check

1. What does atomicity guarantee, and what's the name for undoing a partial transaction?
2. In the bank transfer example, what goes wrong without a transaction?
3. How is consistency different from atomicity?
4. What does isolation protect against when two transactions run at the same time?
5. How does a database make sure a committed change survives a crash?
6. Why is isolation described as "a dial, not an on/off switch"?
7. Why does calling a `@Transactional` method from another method in the same class skip the transaction?
8. By default, which exceptions make Spring roll back a `@Transactional` method, and how do you change that?

**Key points to self-check against:**
- All operations happen or none do; undoing partial work is a rollback.
- Money can leave account A without arriving in B if a failure hits between the two writes.
- Atomicity is about all-or-nothing execution; consistency is about the data obeying its defined rules before and after.
- Each transaction sees a stable view, not another transaction's half-finished changes.
- It writes changes to a durable log on disk before confirming the commit, then replays the log after a crash.
- Databases offer isolation levels that trade stricter isolation for better performance.
- `@Transactional` works through a proxy; an internal `this.method()` call bypasses the proxy.
- Unchecked exceptions (`RuntimeException`, `Error`); use `rollbackFor = Exception.class` to include checked ones.
