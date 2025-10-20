---
title: "Tactical DDD by Example: Can We Implement Aggregates Using Redis?"
seoTitle: "Tactical DDD by Example: Can We Implement Aggregates Using Redis?"
seoDescription: "How to protect business rules using aggregates? Can we use a Redis database for it?"
datePublished: Mon Oct 20 2025 07:34:31 GMT+0000 (Coordinated Universal Time)
cuid: cmgytl4f5000102l5brtx1cg5
slug: tactical-ddd-by-example-can-we-implement-aggregates-using-redis
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1760945568653/b0bf5401-596a-4642-9ca7-e731d17a356f.png
tags: software-development, redis, programming, ddd, domain-driven-design

---

When building distributed systems, protecting business rules becomes increasingly complex. Concurrent operations can violate constraints, and failures can leave systems in inconsistent states. These aren't just technical problems — they're business problems that can cost real money.

In this post, I'll show you how Domain-Driven Design (DDD) aggregates can elegantly solve these challenges, using a streaming system with lease-based concurrency control as our example. We'll explore three approaches: a naive solution you might intuitively build, a traditional aggregate pattern, and an unconventional Redis-based approach. By comparing them, you'll see how the focus shifts from *structure* to *behavior*, and why this matters for building robust systems.

## The Streaming System: Our Real-World Problem

Imagine you're building a platform that manages concurrent sessions across customer accounts. Here are your requirements:

* Each account can run a maximum of X concurrent sessions
    
* Sessions can be started and stopped on demand
    
* The streaming service (which actually runs the sessions) is separate from the orchestration service (which manages and coordinates sessions)
    
* If the streaming service fails or becomes unreachable, the orchestration service should remain consistent
    

This separation between services is intentional and realistic. The streaming service might be slow, it might fail, and network calls might timeout. We need a system that gracefully handles these real-world failures.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1760945416466/7b2966fc-f0b5-440b-bb6f-5fa56e22d59a.png align="center")

## The Lease-Based Model: Elegance Through Time

At the heart of a robust solution lies the lease-based concurrency model. Instead of thinking in absolute terms — "this session is running" — we think in time-limited terms: "this session has a lease until time T."

Here's how it works in practice.

The orchestration service acquires a lease on behalf of a session, then attempts to start the actual session on the streaming service. While the session runs, the streaming service periodically sends session-alive events back to the orchestration service. Each alive event renews the lease, extending its expiration time. When the session stops, the streaming service notifies the orchestration service, which explicitly releases the lease.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1760945427510/66fffa94-13ee-4c8a-b15d-b6e9de6ef956.png align="center")

But here's where the lease model shines: if the streaming service crashes or becomes unreachable, it can no longer send alive events. The leases on the orchestration side simply expire, automatically releasing the slots. No explicit coordination needed. No need to wait for failure detection or manual intervention.

This is remarkably elegant. We've replaced a complex protocol to handle service failures with a simple mechanism: time. The lease model is forgiving of failures, network issues, and slowness — all without explicit coordination. If the streaming service is down for five minutes, the leases expire, and those sessions' slots become available. The system self-heals.

The lease model works particularly well when you have two independent services that need to coordinate, you want failure handling to be simple and automatic rather than explicit and complicated.

In the next sections, we consider three implementation options. To not overwhelm the readers, we will focus only on the most complex operation - acquiring a lease.

## The Naive Approach: Following Intuition

Most of us start building domain models by thinking in nouns. We create a `User` object, an `Account` object, maybe a `Session` object. Each object manages its own state, and we query the database to answer questions.

In our example, you might model it like this:

```go
Account
  - id
  - name
  - maxConcurrentSessions
  
Session
  - id
  - accountId
  - leaseExpiration
  - status
```

To answer the question "can we start a new session for this account?" you'd write something like that:

```sql
SELECT COUNT(*) FROM sessions 
WHERE accountId = ? AND leaseExpiration > NOW()
```

This works. It's intuitive. And it can have serious problems that compound as traffic increases.

First, there's no explicit protection of the rule. The rule "only X sessions per account" lives nowhere in your code — it lives in your head and in your documentation. Different parts of your codebase might check this rule differently, or forget to check it entirely. You might have an anemic domain model: objects that are just data containers with no real behavior.

Or the opposite problem happens. Over time, the `Account` object accumulates more and more responsibilities. It manages sessions, handles concurrency, coordinates with the streaming service, and suddenly it's a god object that does everything. Good naming is crucial here — if you call it `Account` instead of something more specific, it's even easier for the object to drift into becoming a catch-all.

But there's a deeper performance problem hiding here. With high concurrency — multiple requests trying to start sessions for the same account simultaneously — you need [pessimistic locking](https://en.wikipedia.org/wiki/Lock_\(computer_science\)#Database_locks) to preserve atomicity. You need to lock the `Account` row (and possibly `Session` rows) to ensure that two concurrent requests don't both think there's an available slot when there isn't one. Imagine two requests arriving at the same moment when your account has exactly one slot left. Without locks, both queries see one available slot, both add a session, and you've violated your invariant.

Locking works, but it kills scalability. As traffic increases, lock contention increases, and requests pile up waiting for locks. Your throughput is hindered, and your system becomes slower the more successful you are.

This is the fundamental problem with the naive approach: the lack of explicit aggregate boundaries forces you into pessimistic locking, which destroys your concurrency and scalability.

## The Aggregate Pattern: Behavior Over Structure

This is where aggregates come in. An aggregate is one of the most powerful tactical patterns in Domain-Driven Design (but also the most misunderstood…). Instead of thinking about entities and their relationships, think about clusters of domain objects that must be consistent as a single unit.

However, the keyword is **behavior**.

An aggregate protects invariants. An invariant is a rule that must always be true. In our case: "an account never has more than X active sessions." We design the aggregate boundary to contain everything needed to enforce this rule, and we make all modifications go through a single entry point — the aggregate root.

Here's the crucial insight: the aggregate isn't defined by the data structures it contains. It's defined by the behavior it must protect. Data is secondary — it's just the tool we use to protect the rule.

In our streaming system, the aggregate root might be called `AccountSessions`. The name itself should signal responsibility.

An aggregate should be small. This isn't arbitrary — small aggregates mean faster transactions and better scalability. If your aggregate includes everything in your system, you've defeated the purpose.

The aggregate doesn't have to be an object. It can be a function, a procedure, or even a script. The key requirement is: be atomic and transactionally consistent. All modifications to the aggregate must happen in a single transaction, with no way for the invariant to be violated.

## The Traditional Aggregate: Logic in Code

The traditional aggregate pattern puts your domain logic in code — plain classes or functions — not in the database. This is usually the best approach.

You create an aggregate root with an identity. You might have additional data structures inside the aggregate boundary, but all modifications go through the root. You implement a repository pattern to load and save the aggregate (repository as a database adapter).

Here's the conceptual shape of it:

```go
AccountSessions (aggregate root)
  - accountId
  - maxConcurrentSessions
  - activeSessions: List[ActiveSession]
  - version (for optimistic locking)
  
ActiveSession (objects inside aggregate)
  - sessionId
  - leaseExpiresAt
```

The key is that all business logic lives in the aggregate, protecting the invariant in code. When you want to start a new session, you load the aggregate, call `addSession()` on it, and if it succeeds, you save it back. The aggregate is responsible for enforcing the rule: you can never exceed X sessions.

```go
func (m *SessionManager) AcquireSession(ctx context.Context, spec AcquireSpec) (error) {
	// Transaction wraps entire operation
	tx, err := m.db.BeginTx(ctx, nil)
	defer tx.Rollback()

	// Load the entire aggregate from database
	accountSessions, version, err := m.repo.LoadAccountSessionsInTx(ctx, tx, spec.AccountID)

	// Execute business logic (invariant protection happens here)
	err = accaoutSessions.AddSession(spec.SessionID, spec.LeaseDurationSeconds)

	// Save back with optimistic locking
	// This will fail if another transaction modified the aggregate
	err = m.repo.SaveAccountSessionsInTx(ctx, tx, spec.AccountID, accountSessions, version)

	return tx.Commit()
}
```

This approach has real benefits. Your domain logic is testable and explicit (within `AccountSessions.AddSession` method). Your boundaries are clear. Anyone reading the code knows what invariants are being protected and where.

However, it has a small tradeoff: [optimistic locking](https://en.wikipedia.org/wiki/Optimistic_concurrency_control). With high traffic and many concurrent requests trying to start sessions for the same account, you'll get conflicts. If two transactions load the same aggregate and both try to save, only one will succeed (the other will get a version mismatch error). Your application needs to retry, which adds complexity and latency.

However, here's the critical difference from the naive approach: you're not blocking other requests while you hold a lock (you “optimistically” assume that the blocking is not needed). Readers can still load the aggregate. Other accounts aren't affected. You scale much better because you're not serializing all requests on one account — you're just retrying conflicts, which are typically rare unless you're already at capacity.

Nevertheless, in some cases, a lot of conflicts on write can be troublesome. In that case, you can turn to Redis.

## The Unorthodox Approach: Redis and Lua - A Deep Dive

We're used to thinking about Redis as a cache layer — a temporary store to speed up access to data that lives in a real database. But modern cloud Redis services (eg, GCP Memorystore) have changed the game. They are highly available, replicated, and reliable.

This opens a new possibility: for certain use cases, Redis can be your *primary* storage system, not just a cache in front of something else.

Redis offers two critical features for implementing aggregates: it's extremely fast, and it supports Lua scripting with atomic execution. A Lua script that runs on Redis is atomic — either all of it executes, or none of it does. No race conditions, no retry logic needed. This is perfect for aggregate enforcement.

How to model an aggregate in Redis?

In our example, the aggregate boundary should consist of three interconnected data structures that must always stay consistent:

```go
session:<sessionId>           // JSON of Session (individual session lease)
accountSessions:<accountId>   // set of account's session IDs
activeSessions                // sorted set of all sessions by expiration time
```

Each structure serves a purpose:

* `session:<sessionId>` stores the session data (JSON). This is the primary record for a session.
    
* `accountSessions:<accountId>` is a Redis SET that keeps track of all session IDs for an account. This lets us quickly count active sessions for an account.
    
* `activeSessions` is a Redis ZSET (sorted set) that tracks all sessions across all accounts, sorted by expiration time. This is used for efficient lease expiration because querying over keys in Redis is highly inefficient.
    

Together, these three structures form the aggregate boundary. They're logically a single unit that must be kept consistent.

As we said earlier, to provide atomicity in Redis, we need to utilize [Lua Scripts](https://redis.io/docs/latest/develop/programmability/eval-intro/).

```lua
local session_key = KEYS[1]              -- session:<sessionId>
local account_sessions_key = KEYS[2]     -- accountSessions:<accountId>
local active_sessions_key = KEYS[3]      -- activeSessions (all sessions)
local sessions_limit = tonumber(ARGV[1])
local session_id = ARGV[2]
local session_payload = ARGV[3]
local expiration_time = tonumber(ARGV[4])

-- Check if account reached the limit
local sessions_usage = tonumber(redis.call("SCARD", account_sessions_key))
if sessions_usage >= sessions_limit then
    return redis.error_reply("account has reached the maximum number of sessions")
end

-- If we reach here, we know we can safely add - all three operations happen atomically
redis.call("SADD", account_sessions_key, session_id)          -- Add to account's set
redis.call("ZADD", active_sessions_key, expiration_time, session_id)  -- Add to global sorted set
redis.call("SET", session_key, session_payload)               -- Store session data

return sessions_usage
```

This is a single atomic operation. No locks, no retries, no race conditions. Let's trace through what happens with concurrent requests:

Imagine two requests arrive simultaneously, both trying to start a session for the same account that's at capacity (already has 4 sessions, limit is 5):

1. Request A's script begins executing on Redis
    
2. Request A check: `SCARD accountSessions:acc1` → returns 4
    
3. Request A verifies: 4 &lt; 5, so it proceeds
    
4. Request A executes all three Redis calls atomically
    
5. Request B's script, then begins (Redis serializes scripts)
    
6. Request B checks: `SCARD accountSessions:acc1` → returns 5 (because A just added one)
    
7. Request B's script hits the limit check and fails
    

Redis is single-threaded. When you send a Lua script to Redis, the entire script executes on a single thread before any other command is processed. This means that even with thousands of concurrent clients, Redis serializes Lua script execution at the engine level. You're not competing with other transactions or race conditions — you're executing your entire script atomically, then the next script begins. There are no partial states or interleaved operations.

This single-threaded model might sound like a bottleneck, but it's actually why Redis is so performant. There's no context switching, no lock overhead, no coordination protocols. Each script runs to completion, and the throughput is determined by how fast Redis can execute scripts sequentially. For sub-millisecond operations like our session count check, this throughput is remarkably high — tens of thousands of operations per second even under extreme concurrency.

Lua scripts in Redis have constraints: [they can only read/write keys that are explicitly passed in via KEYS and ARGV](https://redis.io/docs/latest/develop/programmability/eval-intro/#script-parameterization). This can make complex logic tricky. For example, if you wanted to query sessions across multiple accounts in a single script, you'd hit limitations.

This constraint actually forces good design: your aggregate boundaries become very clear. You can't accidentally reach outside your aggregate. The price is that some operations that would be easy in a relational database require careful thought in Redis.

The whole Redis aggregate implementation can be found [here](https://github.com/jorzel/streaming-system/blob/main/sessions-service/internal/redis/sessionmanager.go).

## Choosing Your Approach

The naive approach works for small, non-critical systems where lock contention isn't a problem yet. It's the simplest to understand, but it scales poorly as traffic increases.

The traditional aggregate is the right choice for most systems. It's maintainable, testable, and puts your domain logic where it belongs: in code you can easily understand and change.

But for this specific problem — high concurrency handling where retries on conflicts are problematic — Redis becomes compelling. You gain massive performance and simplicity, at the cost of learning a different paradigm and accepting in-memory storage. In that case, Redis should be your domain logic enforcer, but you also need a reporting database for long persistence.

The decision isn't about which approach is "better" — it's about which constraints matter most for your specific problem.

## The Real Insight

All three approaches show a progression from thinking about structure to thinking about behavior. The naive approach asks, "What data should I store?" The aggregate approaches ask, **"What rules must I protect?"**

This is the fundamental shift that makes Domain-Driven Design powerful. Data is a means to an end. Behavior — protecting your business rules — is the point.

When you start designing systems this way, you'll notice that your code becomes clearer, your invariants become explicit, and your systems become more robust. You're not building data structures that happen to have some logic attached. You're building coherent units of behavior that exist to protect what matters: your business rules.

The next time you're building a system, before you create your first object or table, ask yourself: what invariants must I protect? What rule must always be true? Then design your aggregate around that rule, not around your intuition about what data you might need.

That's the real power of aggregates.