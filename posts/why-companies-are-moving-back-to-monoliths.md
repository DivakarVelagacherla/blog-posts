---
title: "Why Companies Are Moving Back to Monoliths"

date: 2026-06-03

slug: "why-companies-are-moving-back-to-monoliths"

summary: "Everyone told you to break everything into microservices. Amazon, Shopify, and Stack Overflow are now moving back to monoliths. Here's the engineering reality behind the trend."

categories: Software Architecture

readTime: 6
---

## The Microservices Dream

For the last decade, microservices were the answer to everything.

Slow deployments? Break it into microservices.

Scaling problems? Break it into microservices.

Team coordination issues? Break it into microservices.

Amazon did it. Netflix did it. Every tech blog told you to do it.

Now Amazon is consolidating services. Shopify moved their core back to a monolith. Stack Overflow serves millions of users from a handful of servers.

What happened?

---

## What Nobody Told You About Distributed Systems

When you move from a monolith to microservices, you don't just change your architecture. You change the nature of failure itself.

In a monolith, failures are loud and complete. Something throws an exception. You get a stack trace. You know exactly what happened and where.

In distributed systems, failures are silent and partial.

### Partial Failure

Service A calls Service B. Service B receives the request, processes it, but the response never makes it back due to a network blip.

From Service A's perspective — did Service B succeed or fail?

It has no idea. Service B could have:

- Succeeded but the response got lost
- Failed halfway through processing
- Still be processing
- Never received the request at all

Four completely different situations. Service A sees the same thing in all four — silence.

In a monolith this never happens. A function either returns or throws. There is no in-between.

### Retry Storms and Duplicate Processing

The natural response to silence is to retry. But retrying without careful design causes a new problem.

Service A times out waiting for Service B. Retries. But Service B already processed the request the first time.

If that request was a payment — the user just got charged twice.

If that request was sending a welcome email — the user gets two.

The solution is **idempotency** — designing operations so that executing them multiple times has the same effect as executing them once.

In practice this means generating an **idempotency key** — a unique ID tied to the user's action, not the retry attempt — and passing it with every request. The receiving service checks if it has already processed a request with that key. If yes, it returns the stored result without processing again.

Critical detail: the idempotency key must be generated before the retry loop, not inside it. One user action = one key. Whether that results in 1 attempt or 5 — same key every time.

Stripe requires this on every payment request. AWS SQS uses message deduplication IDs for exactly this purpose.

### Cascading Failures

This is the failure mode that ends careers.

You have 5 services in a chain. Service E gets slow — taking 10 seconds instead of 100ms.

Service D calls Service E and waits. Service D is now slow.

Service C calls Service D and waits. Service C is now slow.

Service B, Service A — same.

One slow service brought down the entire system. Not because anything crashed. Because slowness propagated silently up the dependency chain until every service ran out of threads.

In a monolith, one slow function slows down one request. Everything else keeps working.

The solution is a **circuit breaker** — when a service detects that a dependency has been failing or slow for a threshold period, it stops sending requests to it entirely and returns a fallback response immediately.

```
Circuit CLOSED — normal, requests flow through
↓ Service starts failing
Circuit OPEN — stop calling the failing service, return fallback
↓ After timeout, send one test request
Circuit HALF-OPEN — if test succeeds, close. If not, stay open.
```

The fallback isn't always an error. Netflix shows popular content when their recommendation service is down. Checkout flows show "limited availability" when inventory service is slow. Graceful degradation over hard failure.

---

## The Hidden Cost Nobody Counted

Beyond the failure modes, microservices introduced operational overhead that teams consistently underestimated.

**Deployment complexity** — instead of deploying one thing, you're coordinating dozens of independent services with their own CI/CD pipelines, versioning, and rollback strategies.

**Observability overhead** — debugging a monolith means reading one log. Debugging a distributed system means correlating logs across 10 services, tracing a request as it hops between them, and figuring out which service introduced the latency.

**Network costs** — every function call that used to be in-memory is now a network hop with latency, serialization overhead, and failure probability.

**Organizational overhead** — microservices were partly sold as an org chart solution. One team per service. But coordination between teams doesn't disappear — it just moves from code to meetings and API contracts.

---

## So Why Did Everyone Move to Microservices?

Microservices solve real problems — at the right scale.

When you have hundreds of engineers working on the same codebase, a monolith becomes a coordination nightmare. Merge conflicts, slow build times, one team's bad deploy taking down everyone else's work.

Microservices solve the **people scaling problem**, not the technology scaling problem.

Amazon, Netflix, Google — they moved to microservices because they had thousands of engineers. The distributed systems complexity was worth it to enable independent team velocity.

Most companies don't have that problem. They have 10 engineers treating a 10-engineer problem like a 1000-engineer problem.

---

## What Companies Are Actually Doing Now

The smarter companies aren't going back to a big ball of mud monolith. They're building **modular monoliths** — a single deployable unit with clear internal boundaries.

You get:

- The deployment simplicity of a monolith
- The code organisation of microservices
- In-process function calls instead of network hops
- One log to debug instead of ten

And when a specific part genuinely needs to scale independently or be owned by a separate team — extract it then. With real evidence. Not speculation.

That's YAGNI applied to architecture.

---

## The Real Lesson

Distributed systems don't make your problems smaller. They make them harder to see.

Partial failures, cascading slowness, duplicate processing — these are problems that don't exist in a monolith. You inherit all of them the moment you put a network between two pieces of your system.

Microservices are a powerful tool. They're also one of the most expensive architectural decisions you can make. The companies moving back to monoliths aren't admitting failure. They're admitting they applied a solution designed for a problem they didn't have.

Knowing which tool fits which problem — that's the judgment that matters.

---

_Microservices solve people scaling problems. Most companies have technology scaling problems. That confusion is expensive._
