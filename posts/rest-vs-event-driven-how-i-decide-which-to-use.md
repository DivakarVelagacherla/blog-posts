---
title: "REST vs Event-Driven: How I Decide Which to Use"

date: 2026-05-04

slug: "rest-vs-event-driven-how-i-decide-which-to-use"

summary: "Not everything needs REST. Not everything needs Kafka. Here's how I think through the decision — and why the answer is almost never one or the other."

categories: Architecture Thinking

readTime: 6
---

## The Wrong Question

Most architecture discussions about REST vs event-driven start with the wrong question.

_"Which is better?"_

Neither is better. They solve different problems. The real question is: **what does this specific interaction actually need?**

Let's work through a real scenario to make this concrete.

---

## The Scenario: A Food Delivery App

User taps "Place Order." That single action needs to:

- Confirm the order
- Notify the restaurant
- Assign a delivery driver
- Send a confirmation email
- Update the user's order history

How do you build this?

The naive approach — chain all of this synchronously in one REST call:

```
User taps Place Order
↓
API confirms order
↓
API notifies restaurant ← waits for acceptance
↓
API assigns driver
↓
API sends email
↓
API updates order history
↓
Response back to user
```

User stares at a loading spinner while all of this happens sequentially. If the email service is down — the entire order fails. If driver assignment is slow — the user waits.

This is the problem with applying REST everywhere without thinking about what each interaction actually needs.

---

## Breaking It Down: What Does Each Step Actually Need?

Before picking an architecture, ask this about each step:

**Does the user need to wait for this to complete before moving on?**

Let's apply that to our scenario:

| Step                    | User needs to wait? | Why                                                                            |
| ----------------------- | ------------------- | ------------------------------------------------------------------------------ |
| Order confirmation      | Yes                 | User needs to know the order went through                                      |
| Restaurant notification | Partially           | Need acceptance before proceeding, but user doesn't need to stare at a spinner |
| Driver assignment       | No                  | Happens after acceptance, user can move on                                     |
| Confirmation email      | No                  | User doesn't need to wait for an email to be sent                              |
| Order history update    | No                  | Background operation, user doesn't need this to proceed                        |

This table immediately tells you that a fully synchronous chain is wrong. Some steps need to be async.

---

## The Right Architecture: Both, Used Appropriately

### Step 1: Order Placement — Synchronous but Non-Blocking

User taps Place Order. The order goes into an SQS queue immediately. User gets an instant response — not "Order Confirmed" but "Order Received, Awaiting Restaurant Acceptance."

The user doesn't wait for the restaurant. They move on. The restaurant consumes the queue message and processes the order asynchronously.

This is the key insight: **synchronous from the user's perspective doesn't mean synchronous under the hood.**

### Step 2: Restaurant Acceptance — The Trigger Point

Once the restaurant accepts, an SNS event fires. This single event is the trigger for everything downstream.

```
Restaurant accepts order
↓
SNS event: ORDER_ACCEPTED
↓
├── Driver Assignment Module ← consumes event
├── Email Module ← consumes event
└── Order History Module ← consumes event
```

All three happen in parallel. Independently. If the email service is down — the driver still gets assigned and order history still updates. Each consumer fails independently without taking down the whole flow.

This is **decoupling** — one of the most important benefits of event-driven architecture.

### Step 3: Real-Time Status Updates — WebSockets

User moves away from the order screen but wants live status updates. "Awaiting acceptance" → "Accepted" → "Driver assigned" → "On the way."

Polling — asking the server every few seconds for updates — works but is wasteful. 100 users tracking orders = hundreds of empty requests per minute.

**WebSockets** flip the model. Instead of the client repeatedly asking "any updates?" — the server pushes updates the moment something changes.

A WebSocket is like a phone call vs sending letters. Normal HTTP is a letter — you send it, get a reply, connection closes. WebSocket is a phone call — connection stays open, both sides can talk whenever they want.

```
User opens order tracking screen → WebSocket connection opens
SNS event fires → triggers WebSocket push to user's app instantly
User closes screen → WebSocket connection closes
```

No polling. No wasted requests. Updates arrive the instant they happen.

---

## The Full Architecture

```
User places order
↓
SQS → Restaurant Module
↓
Restaurant accepts → SNS: ORDER_ACCEPTED
↓
├── Driver Assignment Module
├── Email Module
├── Order History Module
└── WebSocket push → User's app (real-time status)
```

REST for fetching data. Event-driven for reactions to actions. WebSockets for real-time updates. Each used where it fits.

---

## The Decision Framework

After working through scenarios like this, here's how I think about it:

**Use REST when:**

- You need an immediate response — the user is waiting on screen
- The result determines what happens next — you can't proceed without knowing the answer
- It's a simple fetch — get this data and show it

**Use Event-Driven when:**

- Multiple things need to happen as a result of one action
- Those things can happen independently and in parallel
- The user doesn't need to wait for all of them
- Failure of one shouldn't block the others

**Use WebSockets when:**

- The server needs to push updates to the client without being asked
- Updates need to be real-time, not periodic
- Polling would be wasteful given the frequency of updates

---

## The Failure Handling Difference

This is the dimension most engineers miss when comparing REST and event-driven.

**REST — tightly coupled failure:**

If your email service is down, a fully synchronous order placement fails entirely. The user can't place an order because the email system is broken. Two completely unrelated problems become one.

**Event-Driven — isolated failure:**

Email service is down? The order still goes through. Driver still gets assigned. Order history still updates. The email just retries when the service recovers. Each consumer owns its own failure.

At scale, isolated failure is not a nice-to-have. It's the difference between a minor blip and a full outage.

---

## What This Signals

Engineers who default to REST for everything build systems that are easy to understand but brittle at scale. Engineers who default to event-driven for everything build systems that are powerful but hard to debug and reason about.

The right call is knowing when each one earns its place. That judgment — not knowledge of the tools themselves — is what separates engineers who can design systems from engineers who can only implement them.

---

_The question is never which architecture is better. It's which one fits what this specific interaction actually needs._
