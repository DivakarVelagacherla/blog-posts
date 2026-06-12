---
title: "How I'd Design a Rate Limiter from Scratch"

date: 2026-06-11

slug: "how-id-design-a-rate-limiter-from-scratch"

summary: "Not just 'use Redis' — the actual thinking behind designing a rate limiter, the three algorithms you need to know, and where in your infrastructure it should actually live."

categories: Backend Engineering

readTime: 6
---

## The Problem

You've built a public API. It's live. Things are going well.

One morning you wake up to alerts — your server is down. You check the logs. One client is making 10,000 requests per second.

You need a technical solution that doesn't rely on people being reasonable.

That solution is a rate limiter.

---

## Step 1: How Do You Identify a Client?

The first design decision — how do you know who is making the request?

**IP address** is the obvious answer. But it breaks immediately for corporate networks where 1000 employees share one IP. Rate limit that IP and you've blocked an entire company.

**API keys** are the right answer for public APIs. When a client registers, you give them a unique key. Every request includes that key. You rate limit by key — each application gets its own bucket, one bad actor doesn't affect everyone else.

**User identity** is the right answer for per-user limits inside an application. When a user logs in, their auth token (JWT) carries their user ID and tier. Your rate limiter reads that and applies the right limit per user.

In practice you layer both:

- **API key** — identifies the application, same for all users of your app
- **User token** — identifies the individual user, enables per-user and per-tier limits

```
Free user → 100 requests per hour
Premium user → 10,000 requests per hour
```

Different problems. Both needed.

---

## Step 2: Where Do You Store the Counter?

Every rate limiter needs to track — how many requests has this client made in the current window?

**Redis** is the standard answer. In-memory so reads and writes are microseconds fast. Supports atomic increment operations so concurrent requests don't corrupt the counter. TTL support so counters automatically expire after the time window.

Basic flow:

```
Request comes in with API key
↓
Check Redis — how many requests in the last minute?
↓
Under limit → allow, increment counter
Over limit → reject, return 429 Too Many Requests
↓
Counter auto-expires after 1 minute
```

---

## Step 3: Which Algorithm?

This is where most explanations stop at "use Redis" without explaining the actual logic. There are three algorithms worth knowing.

### Fixed Window

A counter resets at the start of every time window. 100 requests per minute — counter starts at 0 at :00, resets at :01.

**Simple to implement.** One counter per client per window.

**The problem — boundary exploit.** A client sends 100 requests at 11:59:59 and 100 more at 12:00:01. 200 requests in 2 seconds. Your rate limiter allowed it because each minute was technically within the limit.

**When to use it:** Low volume, sensitive operations like password resets, OTP generation, login attempts. 5 requests in 15 minutes — even if someone exploits the boundary and gets 10, the damage is minimal. Fixed window is simple and sufficient.

### Sliding Window

Instead of resetting at a fixed point, the window rolls with time. At any moment, look back exactly 60 seconds and count requests in that window.

100 requests at 11:59:59? At 12:00:30 the sliding window still sees those requests. They're within the last 60 seconds. No new requests allowed yet.

**No boundary exploit. Accurate.**

**The problem — storage cost.** You need to store the timestamp of every individual request to count how many fall in the last 60 seconds. At high traffic that's significant Redis storage.

**When to use it:** When accuracy is critical regardless of complexity.

### Token Bucket

The most practical algorithm for production systems.

Imagine a bucket that holds 100 tokens. Tokens refill continuously at a constant rate — say 1 token per second up to the maximum of 100. Every request costs 1 token. Bucket has tokens — request goes through. Bucket empty — request rejected.

The refill is **continuous**, not batch. Not "add 10 tokens every 10 seconds" — that creates mini bursts. Instead tokens trickle in smoothly over time.

In Redis you don't run a background job. You calculate mathematically on each request:

```
tokens_to_add = (current_time - last_request_time) × refill_rate
current_tokens = min(max_tokens, stored_tokens + tokens_to_add)
```

Two values in Redis per client. Math on every request. No background jobs.

**Why token bucket wins:**

- No boundary exploit — no fixed reset point to exploit
- Handles legitimate bursts — a client that's been quiet has a full bucket, can burst when needed
- Low storage — just two numbers per client
- Accurate enough for almost all production use cases

| Algorithm      | Storage        | Accuracy | Burst handling | Best for                 |
| -------------- | -------------- | -------- | -------------- | ------------------------ |
| Fixed Window   | One counter    | Low      | Poor           | Low volume sensitive ops |
| Sliding Window | All timestamps | High     | Good           | Critical accuracy        |
| Token Bucket   | Two values     | High     | Excellent      | Most production systems  |

---

## Step 4: Where in Your Infrastructure?

This is the question most rate limiter discussions skip entirely.

**Inside your Lambda or application?**

Works but expensive. By the time the request reaches your Lambda, you've already paid for the invocation. Malicious traffic costs you money even when blocked.

**At API Gateway?**

Better. Blocks before hitting your compute layer. Standard choice for per-client API key limits.

**At the CDN / Edge?**

Best for volume attacks. A bot making 10,000 requests per second — block at the CDN and none of that traffic enters your infrastructure. Zero compute cost. Cloudflare, CloudFront, and most CDNs support edge rate limiting out of the box.

The right answer is **layer all three:**

```
CDN / Edge — block volume attacks, bots, DDoS
↓
API Gateway — enforce per API key quotas
↓
Application — per-user limits, tier based limits
```

Defense in depth. Each layer catches different abuse patterns. Breaking through one doesn't mean breaking through all three.

---

## The Full Design

```
Request arrives at CDN
↓
Edge rate limit: is this IP making too many requests? → block if yes
↓
API Gateway: is this API key over quota? → 429 if yes
↓
Application: token bucket check per user ID in Redis
↓
Under limit → process request, decrement token
Over limit → 429 Too Many Requests
```

---

## What This Signals

Rate limiting is a classic system design interview question. But the signal isn't whether you know the token bucket algorithm.

It's whether you think through the layers — identification, storage, algorithm choice, infrastructure placement — and make deliberate trade-offs at each step.

Knowing when fixed window is enough and when token bucket is needed. Knowing that blocking at the CDN is cheaper than blocking at the application. Knowing that API keys and user tokens solve different problems.

That layered thinking is what separates a textbook answer from a production-ready design.

---

_Rate limiting isn't one decision. It's four decisions that compound on each other._
