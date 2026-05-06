```yaml
title: "Why Most Software is Overengineered"
date: 2026-04-06
slug: "why-most-software-is-overengineered"
summary: "The dangerous default of building for 10x scale on day one — and why the engineers I trust most push back on unnecessary complexity before a single line of code is written."
categories:
  - Engineering Mindset
readTime: 4
```

---

# Why Most Software is Overengineered

## The Dangerous Default

Most engineers, when starting a new feature, ask the wrong question.

Not "what does this need to do today?" — but "what if this needs to handle millions of users?"

And then they build for that answer. Before they have 100 users. Before they have 10.

I've seen it play out the same way every time:

- A notification system that supports 6 channels before anyone's signed up
- A microservices architecture for a product that doesn't exist yet
- Kafka for events that a database trigger would handle fine
- Kubernetes for an app that fits comfortably on a single server

This is overengineering. And it's the default behavior for a lot of smart engineers.

## The Real Cost Nobody Talks About

Overengineering doesn't just waste build time. It compounds.

**Complexity debt** — every abstraction you add is code someone has to maintain forever. Not just you. Every engineer who joins after you.

**Onboarding friction** — new engineers spend weeks understanding clever solutions instead of shipping. The more elegant the architecture, the more expensive it is to learn.

**Wrong assumptions** — here's the kicker. By the time you actually hit scale, the constraints you'll face are completely different from what you imagined on day one. You optimized for hypothetical problems with real time.

## The Three Principles Behind This

This isn't a new idea. It has three names that engineering has borrowed from different centuries.

**YAGNI** — You Aren't Gonna Need It. Don't build functionality until you actually need it. Not "might need." Not "will probably need." Actually need, today, with evidence.

**Occam's Razor** — from 14th century philosopher William of Ockham: don't multiply complexity beyond necessity. In engineering terms: the simplest solution that solves the real problem is almost always the right starting point. The "razor" metaphor means shave away everything that isn't necessary.

**Premature Optimization** — Donald Knuth called it "the root of all evil." Optimizing before you have evidence of a problem means spending real time solving imaginary problems. 200ms for a feature with 500 users? Nobody complained. Wait for evidence before paying the cost of optimization.

Three different ideas. Three different centuries. All fighting the same enemy: **complexity added before it's earned.**

## But What About Scale?

Here's where it gets nuanced — and where junior engineers often misread the advice.

YAGNI doesn't mean write throwaway code. It doesn't mean ignore the future entirely.

There's a critical difference between **designing for scale** and **building for scale**.

You can build the email notification system today — just email, nothing else. But you structure it so that adding SMS is a one-day job, not a three-week rewrite. You didn't build the SMS integration. But you didn't make it impossible either.

That's the SOLID principle at work — specifically Open/Closed: open for extension, closed for modification. You leave the door open without walking through it.

The senior move isn't "ignore the future." It's **"make the future easy without paying for it today."**

## The Question I Ask Before Adding Any Complexity

Before adding anything — an abstraction, a new service, an optimization, a new channel — I ask three questions:

1. **Do we have this problem today?** Not "will we." Not "might we." RIGHT NOW, with real users and real data.
2. **What breaks if we solve it simply?** Usually the answer is: nothing.
3. **Can we refactor when we know more?** Usually the answer is: yes.

If the answer to question one is "no" — the solution is to wait.

## What This Signals

The engineers I trust most aren't the ones with the most impressive architectures. They're the ones who suggest the simplest solution that works — and can articulate exactly what they're deferring and why.

That's the judgment that matters in a senior engineer. Not cleverness. Not predicting every future requirement.

The engineer who pushes back on complexity before a single line of code is written is often the most senior person in the room.

---

_Start simple. Complicate intentionally. Never complicate by default._
