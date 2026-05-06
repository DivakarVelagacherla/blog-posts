---
title: "Why Our API Was Slow and the Fix Wasn't the Database"

date: 2026-04-27

slug: "why-our-api-was-slow-and-the-fix-wasnt-the-database"

summary: "Our SLA was 200ms. We were hitting 1.8s. Everyone blamed the database. Here's the 3-step debugging session that found the real culprit — and it was three layers up."

categories: Problem → Solution

readTime: 6
---

## The Alert Nobody Wanted

A Slack message from the PM: _"Users are complaining the dashboard is taking forever to load. Can you look into it?"_

API response time: 1.8 seconds. SLA: 200ms. Something was very wrong.

The first instinct of every engineer on the team was the same — must be the database. It always is, right?

So we looked at the database. And found nothing.

That's when the real debugging started.

---

## Step 1: Eliminate Systematically

The mistake most engineers make when debugging performance is they have a hypothesis and go fix that hypothesis. Someone says "API is slow" and everyone immediately assumes database — because that's where performance problems usually live.

Experienced engineers don't assume. They eliminate systematically. They follow the request through every layer and ask: where is the time actually being spent?

So that's what we did.

**First stop: the query.**

We ran EXPLAIN ANALYZE on the database query. Indexes were in place. The query itself returned in 40ms. Clean.

The database was not the problem.

**Second stop: the connection.**

If the query takes 40ms but the API takes 1800ms — where are the other 1760ms going?

This is where it got interesting.

---

## The Real Culprit: Connection Pool Exhaustion

To understand what we found, you need to understand how database connections work.

Opening a database connection is expensive. It involves a network handshake, authentication, and memory allocation on both sides. Doing this for every single request is wasteful and slow.

This is why applications use a **connection pool** — a set of pre-opened connections created at startup and reused across requests. Think of it like a library. The library has 10 books. You borrow one, read it, return it. Someone else borrows the same book. The book never gets destroyed — it just circulates.

The pool works the same way:

- Request comes in → borrow a connection
- Run the query
- "Close" the connection → pool intercepts, puts it back
- Next request borrows the same connection

When your code calls `connection.close()` with a pool — it doesn't actually close the connection. The pool intercepts that call and returns it for reuse. The real connection stays alive.

**So what was happening in our case?**

Our connection pool was configured with a maximum of 10 connections. Our API was receiving 50 concurrent requests. 40 requests were sitting in a queue waiting for a connection to free up.

The query took 40ms. But each request spent 1760ms just waiting in line for a borrowed connection to be returned.

That's connection pool exhaustion. The database was never slow. The bottleneck was the queue outside the database.

---

## Why Were Connections Being Held So Long?

Bumping the connection pool size is the obvious fix. But you can't increase it forever — your database has a limit on concurrent connections based on its instance size. More connections than the database can handle and everything degrades or crashes.

The real question was: why were connections being held for 1760ms when the query only took 40ms?

We looked at the code and found this pattern:

```
open DB connection
query DB                    ← 40ms
call external fraud API     ← 1600ms waiting here
process result
release DB connection
```

The database connection was opened at the top and not released until the very end — after the external fraud detection API call completed.

The connection wasn't being used during those 1600ms. It was just being held hostage while the code waited for an external service.

This is **thread blocking** — holding a resource while doing work that doesn't need that resource.

The fix was simple once we saw it: release the connection immediately after the database work is done, before making any external calls.

```
open DB connection
query DB                    ← 40ms
release DB connection       ← returned to pool immediately
call external fraud API     ← 1600ms, no connection held
process result
```

Same logic. Same queries. Same external API. But now connections are available for other requests while waiting for the fraud service.

**The rule: hold a connection only while talking to the database. Release it the moment you're done.**

Everything else — processing, external API calls, business logic — should happen without a connection held.

---

## A Note on Lambda and Connection Pools

If you're using serverless functions like AWS Lambda, connection pooling works differently.

Lambda spins up a new container per invocation. If you open a connection inside the handler, it gets destroyed when the invocation ends — expensive on every call.

The trick is initializing the connection outside the handler, so it survives across warm container invocations. But at scale, 200 Lambda containers each holding their own direct database connection can overwhelm the database.

The solution is **RDS Proxy** — it sits between your Lambdas and your database and manages a centralized connection pool. Your Lambdas connect to the proxy, not directly to the database. The database sees a manageable number of connections regardless of how many Lambdas are running.

---

## The Bonus Discovery: N+1 Queries

While we were in the code, we found another problem that wasn't causing immediate pain — but would have at scale.

The dashboard was fetching a list of transactions and then for each transaction fetching the associated user details separately.

```
Query 1: SELECT * FROM transactions LIMIT 100

Then for each transaction:
Query 2:  SELECT * FROM users WHERE id = 1
Query 3:  SELECT * FROM users WHERE id = 2
Query 4:  SELECT * FROM users WHERE id = 3
... 100 more queries
```

101 database queries to render one dashboard. This is the **N+1 problem** — 1 query to fetch N records, then N queries to fetch related data one by one.

With 100 transactions at 5ms per query — that's 500ms just for user details. At 1000 transactions it's 5 seconds.

The fix is straightforward — get everything in as few queries as possible:

```sql
SELECT transactions.*, users.name
FROM transactions
JOIN users ON transactions.user_id = users.id
LIMIT 100
```

One query instead of 101.

N+1 is particularly sneaky when using ORMs — the extra queries are hidden behind clean looking code. `post.getAuthor().getName()` looks innocent but silently fires a database query for every post in a loop. Always check what your ORM is actually doing under the hood.

---

## GraphQL and N+1

N+1 gets especially interesting in GraphQL architectures. Each field has its own resolver. If you have a query that returns 10 users and each user has a posts field — each resolver fires independently, potentially making 10 separate database calls.

The standard solution is **DataLoader** — a batching utility that collects all IDs requested across resolvers in a single event loop tick and fetches them in one query:

```
Instead of:
SELECT * FROM posts WHERE user_id = 1
SELECT * FROM posts WHERE user_id = 2
... 10 times

DataLoader batches to:
SELECT * FROM posts WHERE user_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
```

One query. Same result.

In AppSync architectures with Lambda resolvers, batching can be configured at the resolver level — collecting IDs up to a configured batch size before firing the Lambda, achieving the same effect automatically.

---

## The Lesson

Three problems. One debugging session. None of them were where we initially looked.

- **Connection pool exhaustion** — too many requests waiting for too few connections
- **Thread blocking** — connections held during work that didn't need them
- **N+1 queries** — silent query multiplication hiding in loops

The database query was fine the whole time.

This is the most important takeaway: **performance problems are rarely where you first look.** The symptom points one direction. The root cause is usually somewhere else entirely.

You can only find it if you understand every layer between the user's request and the data coming back. And you eliminate systematically — not by assumption, not by gut feel, but by measuring each layer until you find where the time is actually going.

---

_The engineers who debug fastest aren't the ones who guess right. They're the ones who know exactly where to look next when their first guess is wrong._
