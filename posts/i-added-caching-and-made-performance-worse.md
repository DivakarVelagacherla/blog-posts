---
title: "I Added Caching and Made Performance Worse"

date: 2026-05-11

slug: "i-added-caching-and-made-performance-worse"

summary: "Cache invalidation is one of the hardest problems in computer science. I learned it the hard way — here's every mistake I made and what I'd do differently."

categories: Backend Engineering

readTime: 6
---

# I Added Caching and Made Performance Worse

## The Obvious Fix That Wasn't

App is slow. Users are complaining. You've heard caching fixes performance problems.

So you add a cache.

App gets slower.

This is more common than engineers admit. And it almost always comes down to the same set of mistakes. Here's every one of them.

---

## Mistake 1: Caching the Wrong Data

Before adding any cache, there's one question you need to answer:

**What is the probability that the next request asks for the same data I just cached?**

If every user fetches different data — unique feeds, personalised results, user-specific history — your cache hit rate is near zero. Every request:

1. Checks the cache — miss
2. Goes to the database anyway
3. Writes the result to cache — extra work for nothing

You added a layer that does nothing useful but costs you on every single request.

The fix is understanding your access patterns before building anything.

Take a news app as an example:

- **Homepage headlines** — same 10 articles shown to every user. Cache this. One database hit serves a million users.
- **User reading history** — unique per user, different for everyone. Don't cache this. Cache hit rate would be near zero.

Same data, many users — cache wins. Unique data per user — cache adds nothing but overhead.

**Rule: Only cache data with high repetition across requests. Measure your access patterns first.**

---

## Mistake 2: Wrong TTL for the Data Type

TTL — time to live — is how long cached data stays valid before expiring.

Set it too long and users see stale data. Set it too short and you're hitting the database almost as often as without a cache.

The mistake is treating TTL as a technical decision. It's actually a product decision.

The right question is: **how stale is too stale for this specific data?**

| Data Type               | Acceptable Staleness | Suggested TTL     |
| ----------------------- | -------------------- | ----------------- |
| Breaking news headlines | Minutes              | 1-5 mins          |
| Article content         | Hours                | 1-24 hours        |
| User profile picture    | Days                 | 24 hours          |
| Stock prices            | Seconds              | 5-30 seconds      |
| Static config data      | Indefinite           | Days or permanent |

Different data has different freshness requirements. A one-size-fits-all TTL is almost always wrong.

And for truly time-sensitive data like breaking news — the solution isn't a shorter TTL. It's a different delivery mechanism entirely. Breaking news goes through push notifications, not the headlines cache. TTL is irrelevant when the delivery channel bypasses the cache.

---

## Mistake 3: The Cache Stampede

You set a 1 minute TTL on headlines. Your app has 1 million active users.

At the exact moment the TTL expires, all 1 million requests hit the cache simultaneously. All get a miss. All go to the database at the same time.

Your database just received 1 million concurrent requests in a split second. The cache was supposed to protect the database. Instead it created a traffic spike far worse than normal load.

This is called the **cache stampede** or **thundering herd problem.**

### How to prevent it

**Cache warming — proactive refresh:**

A background job fetches fresh data and updates the cache before the TTL expires. When the TTL hits, new data is already in place. No stampede.

**Mutex lock:**

When cache expires, only one request is allowed to go to the database and refresh it. All other requests wait for that one to finish, then serve from the refreshed cache. One database hit instead of a million.

**Stale while revalidate:**

Serve the stale data to users while a background request fetches fresh data. Users get an instant response — even if slightly old. Cache refreshes behind the scenes. This is what most CDNs do by default.

### What if the background job fails?

Background job fails. TTL expires. Cache is empty. You need a safety net.

- **Keep stale data as fallback** — don't hard delete on TTL expiry. Mark as expired but keep the data. If the refresh fails, serve stale rather than hitting the database.
- **Circuit breaker** — if cache is empty and database is getting hammered, stop sending requests to the database. Return stale data or a graceful error rather than letting the database crash.
- **Redundant background jobs** — run two independent jobs with offset schedules. Both would have to fail simultaneously for the cache to go cold.
- **Monitoring and alerts** — alert when cache hit rate drops below a threshold. On-call engineer triggers a manual refresh within minutes.

In practice most teams combine these — redundant jobs, stale data fallback, and monitoring. No single point of failure.

---

## Mistake 4: Not Handling Cache Invalidation

Phil Karlton famously said:

\*"There are only two hard things in computer science: cache invalidation and naming things."

Here's why.

You've cached the headlines. Content team updates a headline — breaking news, they need to fix a critical error. Database is updated immediately. But users are still seeing the old headline from cache for the next 5 minutes.

Waiting for TTL to expire is not acceptable when the data is wrong.

**Cache invalidation** means explicitly clearing or updating the cache when the underlying data changes — not waiting for TTL.

Two approaches:

**Trigger based invalidation:**

When the content team saves an update, it fires an event that immediately triggers a cache refresh. No waiting.

**Event driven invalidation:**

CMS publishes an event to a message queue. Cache service consumes it and refreshes. Decoupled, reliable, and retryable if it fails.

Why is cache invalidation genuinely hard? Because in distributed systems you often have multiple cache layers — CDN cache, application cache, database query cache. Invalidating all of them consistently, in the right order, without serving stale data in between — that's where the complexity lives.

And if you invalidate too aggressively — clearing the cache on every small change — you eliminate the performance benefit entirely. You're back to hitting the database on every request.

The art is knowing what to invalidate, when, and at which layer.

---

## The Full Picture

Caching is not a performance silver bullet. It's a trade-off with real costs:

- Complexity — warming, invalidation, stampede prevention all need to be built and maintained
- Staleness risk — every cache is a promise that data might be slightly old
- Operational overhead — one more system to monitor, debug, and maintain

Caching earns its place when:

- The same data is requested frequently by many users
- The data doesn't change so often that invalidation becomes a full time job
- The performance gain justifies the added complexity

And it doesn't earn its place when:

- Data is unique per user
- Data changes faster than your TTL
- Your traffic isn't high enough to justify the overhead

---

## What This Signals

Engineers who add caches without thinking about access patterns, TTL strategy, and invalidation are optimizing blindly. They're adding complexity without understanding the trade-off.

Engineers who ask "what's the cache hit rate?" before building anything — who think about staleness as a product decision, not a technical one — are the ones who make systems faster without making them fragile.

Knowing when NOT to cache is just as important as knowing how to cache.

---

_Cache invalidation is hard. Adding a cache without thinking about invalidation is harder — on your users._
