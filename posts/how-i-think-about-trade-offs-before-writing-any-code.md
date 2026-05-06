---
title: "How I Think About Trade-offs Before Writing Any Code"
date: 2026-04-13
slug: "how-i-think-about-trade-offs-before-writing-any-code"
summary: "How do you decide what technology to use before writing a single line of code? It starts with three questions — and the answer is never about what sounds impressive."
categories: Software Architecture
readTime: 5
---

## The Question That Changes Everything

A PM walks up to you and says — _"We need to add search to the app."_

Two options are on the table:

- **Option A** — A simple LIKE query on your existing database
- **Option B** — Integrate Elasticsearch

Most engineers immediately have an opinion. Some jump to Elasticsearch because it sounds impressive and scalable. Others dismiss it because the LIKE query is already working fine.

Both are wrong.

Not because of what they chose — but because of **how** they chose it. They reached for a conclusion before understanding the context.

That's the trap. And it's how teams end up with solutions that are either embarrassingly under-built or hopelessly over-engineered.

The engineers I've learned the most from don't have instant opinions. They have a framework. And that framework always starts with the same three questions.

## The Framework: Feasibility, Maintainability, Operability

Before any significant technical decision, I run it through three lenses.

Not sequentially. Not as a checklist. As a triangle — three forces that pull against each other, and your job is to find the right balance given your specific context.

### 1. Feasibility

_Can we actually build this, and at what cost?_

Feasibility isn't just time. It's:

- How long will this take to build?
- What does it cost — in engineering hours, infrastructure, licenses?
- Does our team have the skills to build it well?

That last point is the one engineers consistently underestimate. Elasticsearch might take a week if someone on your team has used it before. It might take two months if nobody has. The tool doesn't change. The context does.

A technically superior solution that your team can't build confidently is not actually superior. It's a risk.

### 2. Maintainability

_How easy is it to change, debug, and understand over time?_

Maintainability is about the long game. Every line of code you write today, someone has to read tomorrow. Maybe that someone is you in six months, having completely forgotten what you were thinking. Maybe it's a new engineer who just joined.

Maintainability asks:

- How quickly can a new engineer understand this?
- How easy is it to change when requirements shift?
- How hard is it to debug when something goes wrong?

A LIKE query is three lines of SQL. Any engineer on your team can read it, change it, and debug it in minutes. Elasticsearch involves index configurations, mapping definitions, sync logic, and query DSL that takes time to learn.

Neither is inherently better. But the maintainability cost is very different — and that difference matters more as your team grows.

### 3. Operability

_How easy is it to run this in production?_

Operability is the one engineers forget most often in design discussions. You build the feature, it works locally, you ship it — and then discover that running it in production is a completely different problem.

Operability asks:

- How complex is deployment and configuration?
- What does monitoring look like?
- What happens when it breaks at 2am? Who fixes it and how?
- Does this introduce new infrastructure you don't have experience managing?

Elasticsearch is a separate system. You need to host it, keep it in sync with your database, monitor it for failures, and handle the cases where search results go stale because the sync fell behind. That's real operational overhead.

A LIKE query has almost zero operability cost. It runs on your existing database. You already know how to monitor it, debug it, and fix it.

## Applying the Triangle: The Search Example

Let's apply all three lenses to our original question — LIKE query vs Elasticsearch.

But here's the key: **the answer depends entirely on context.** So let's look at two different contexts.

### Context 1: Internal tool, 500 users, search is a secondary feature

|                     | LIKE Query                    | Elasticsearch              |
| ------------------- | ----------------------------- | -------------------------- |
| **Feasibility**     | Hours to build                | Days to weeks              |
| **Maintainability** | Simple SQL, everyone knows it | New system, learning curve |
| **Operability**     | Zero extra infra              | Hosting, sync, monitoring  |

**Decision: LIKE query.** No evidence of a performance problem. No user complaints. Introducing Elasticsearch here is paying a real cost to solve an imaginary problem.

### Context 2: Consumer product, 2M users, search is the core feature

|                     | LIKE Query                         | Elasticsearch                   |
| ------------------- | ---------------------------------- | ------------------------------- |
| **Feasibility**     | Fast but will break under load     | Investment upfront, scales well |
| **Maintainability** | Simple but won't handle complexity | More complex but built for this |
| **Operability**     | Will become a bottleneck           | Designed for production search  |

**Decision: Elasticsearch.** The scale justifies the cost. The operability investment is worth it because search is core to the product.

Same two options. Completely different decisions. Because the context changed.

## The Question Behind the Framework

Notice what this framework is really doing — it's forcing you to gather context before forming an opinion.

Specifically:

- Who are your users and how many?
- How often do they use this feature?
- Has anyone actually complained about the current solution?
- Does your team have the skills to operate the more complex option?

That last question — _has anyone actually complained?_ — is more important than engineers admit. It's YAGNI applied to performance. Don't optimize what nobody has reported as broken. Wait for evidence.

This is evidence-based decision making. And it's one of the clearest signals of an experienced engineer — they don't have opinions before they have data.

## When the Triangle Pulls in Different Directions

Here's where it gets hard. The three lenses often conflict.

Maybe Elasticsearch scores high on feasibility because your team knows it well — but low on operability because you don't have DevOps bandwidth to manage it. What do you do?

There's no formula. That's the point. The triangle doesn't give you the answer — it gives you the **right conversation.** It surfaces the trade-offs clearly so the decision is conscious, not accidental.

And sometimes the right answer is: _"Not yet. Let's revisit when we have more evidence."_

That's not indecision. That's judgment.

## What This Signals

When a hiring manager asks _"how did you decide to use X over Y?"_ — they're not looking for the right answer. They're looking for evidence that you thought about it deliberately.

_"We evaluated feasibility, maintainability, and operability given our context at the time. Here's what we found and here's the trade-off we consciously accepted."_

That answer — regardless of what X and Y are — signals someone who can make decisions without being told what to decide.

That's the engineer hiring managers want in the room.

---

_The right technical decision isn't the most impressive one. It's the most appropriate one for the context you're actually in._
