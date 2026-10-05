# Day 13: Database Replication

## What problem does it solve?

So far, every read assumed **one database server**. That creates two problems:
1. **Single point of failure.** If that server's disk dies or it crashes, the entire application is down, and maybe the data is lost.
2. **Read bottleneck.** Most applications read far more than they write (browsing products vs buying one). One server has to handle all of it.

**Replication** keeps **copies of the same data on multiple database servers**. You saw this coming: Day 7 spread *application* load across servers; replication does it for the database.

## Leader-follower (primary-replica): the most common setup

```
             writes
App ───────────────────► Leader (primary)
  │                        │  │
  │ reads                  │  │ replicates changes
  ├──────────► Follower 1 ◄┘  │
  └──────────► Follower 2 ◄───┘
```

- **All writes go to the leader.** It's the single source of truth.
- The leader sends every change to the **followers** (also called **replicas**).
- **Reads can go to any follower**, which spreads the read load.

This gives you:
- **Read scaling:** add more followers to handle more reads.
- **High availability:** if the leader dies, a follower can be **promoted** to become the new leader (**failover**).
- **Safer data:** copies live on separate machines, often in separate data centers.

## Synchronous vs asynchronous replication

**Synchronous:** the leader waits until a follower confirms it has the change **before** telling the app "write successful."
- ✅ The follower is guaranteed up to date, so no data is lost if the leader dies.
- ❌ Every write is slower, and if that follower is down or slow, writes stall.

**Asynchronous:** the leader confirms the write immediately, and sends changes to followers in the background.
- ✅ Fast writes, unaffected by slow followers.
- ❌ Followers can **lag behind**. And if the leader crashes before shipping a recent change, that write is **lost**, even though the app was told it succeeded.

Many systems compromise: **one** synchronous follower (guaranteeing at least one up-to-date copy) and the rest asynchronous.

## The big consequence: replication lag

With asynchronous replication, a follower might be a few milliseconds, or occasionally seconds, behind. That causes very real bugs:

**Read-your-own-writes problem.** A user updates their profile name (written to the leader), the page reloads, and the read goes to a lagging follower. They see their **old** name, and think the save failed.

Common fixes:
- After a user writes something, **read their own data from the leader** for a short period.
- Route reads that **must** be fresh (account balance, just-placed order) to the leader. Send everything else, like product listings and search, to followers.

This is **eventual consistency** in action, the same idea from the microservices and message queue reads: followers *will* catch up, just not instantly.

## Failover isn't free either

Promoting a follower sounds simple, but:
- With async replication, the new leader may be missing the old leader's last few writes.
- **Split brain:** if the old leader was only *temporarily* unreachable and comes back still thinking it's the leader, you now have two leaders accepting conflicting writes. Systems prevent this with careful failure detection and by "fencing" off the old leader.

Managed services (like Amazon RDS or Google Cloud SQL) handle much of this automatically, but it's important to know what they're doing for you.

## Replication vs sharding (preview)

Don't confuse them:
- **Replication:** *same* data, copied to several servers. Helps reads and availability. Doesn't help when **writes** or **total data size** outgrow one server, because every server still holds everything.
- **Sharding:** data *split* across servers, each holding a different portion. Helps write scaling and data size. That's a later topic.

## Java angle

In Spring, you can route reads to replicas by marking read-only transactions:

```java
@Transactional(readOnly = true)
public Product getProduct(Long id) { ... }   // can be routed to a follower
```

`readOnly = true` doesn't route anything by itself. You configure a routing `DataSource` (Spring's `AbstractRoutingDataSource`) that checks it and picks the follower's connection pool (Day 10). Some database drivers and proxies can also split reads and writes for you.

## Common pitfalls

- **Assuming followers are always current.** Replication lag causes "my change disappeared" bugs.
- **Treating replicas as backups.** If someone runs a bad `DELETE`, it replicates to every follower within moments. Replication protects against machine failure, not human mistakes. You still need real backups.
- **Expecting replication to scale writes.** Every write still goes through the one leader.

## Interviewer follow-ups

**"How would you scale a read-heavy application?"** Add read replicas and route reads to them, with caching (Day 6) in front. Send reads that must be fresh, especially a user reading their own recent writes, to the leader.

**"What's the risk of asynchronous replication?"** Followers lag, causing stale reads. And if the leader fails before replicating recent writes, those acknowledged writes can be lost on failover.

## Retention Check

1. What two problems does a single database server have?
2. In leader-follower replication, where do writes go, and where can reads go?
3. What is failover?
4. Compare synchronous and asynchronous replication: one advantage and one risk each.
5. What is the read-your-own-writes problem, and how can you fix it?
6. What is split brain?
7. Why aren't replicas a substitute for backups?
8. What's the difference between replication and sharding?

**Key points to self-check against:**
- It's a single point of failure, and it has to handle every read alone.
- Writes go to the leader; reads can go to any follower.
- Promoting a follower to become the new leader when the leader fails.
- Sync: no data lost on failover, but slower writes. Async: fast writes, but followers lag and recent writes can be lost.
- A user reads from a lagging follower and doesn't see their own update; fix it by reading their own data from the leader for a while.
- Two servers both believing they're the leader and accepting conflicting writes.
- Mistakes like a bad `DELETE` replicate to every copy almost immediately.
- Replication copies the same data everywhere (helps reads and availability); sharding splits the data (helps writes and size).
