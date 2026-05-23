---
title: "What a Load Balancer Actually Does (and What It Doesn't)"

date: 2026-05-18

slug: "what-a-load-balancer-actually-does-and-what-it-doesnt"

summary: "Most engineers use load balancers every day but can't explain how traffic actually gets distributed. Here's every algorithm, every trade-off, and the one architectural decision that makes most of the complexity disappear."

categories: Software Architecture

readTime: 7
---

## The Simple Concept With a Lot of Depth

Your app is growing. One server can't handle the traffic. You put a load balancer in front of multiple servers.

Simple enough. But how does the load balancer actually decide which server gets each request? And what happens when you add a new server? Or when one goes down?

Most engineers click through the AWS console to set one up without ever thinking about these questions. Here's what's actually happening.

---

## Algorithm 1: Round Robin

The simplest possible approach. Rotate through servers in order.

```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1
Request 5 → Server 2
...and so on
```

Implementation is a single counter that increments and wraps back to zero. Zero state beyond that. Extremely fast to compute.

**The problem:** Round robin is fair by count, not by load.

Not all requests are equal. Some take 5ms. Some take 2 seconds. If Server 1 gets a heavy 2 second request, round robin still sends the next request to Server 1 in its turn — even though Server 2 and Server 3 are sitting idle.

**When round robin is still the right choice:**

When all requests are roughly equal in duration — a typical API where every request takes 10-50ms — round robin distributes load almost as well as any smarter algorithm. And at extremely high throughput where millions of requests per second flow through, round robin's zero state overhead matters. Every microsecond counts.

---

## Algorithm 2: Least Connections

Instead of rotating blindly, the load balancer tracks how many active HTTP connections each server is currently handling and always sends the next request to the server with the fewest.

```
Server 1 — 5 active connections
Server 2 — 2 active connections
Server 3 — 8 active connections

Next request → Server 2
```

When Server 1 finishes that heavy 2 second request — its count drops and it becomes eligible again. No wasted capacity sitting idle.

**Important:** These are HTTP connections between the load balancer and your servers — not database connections. The load balancer operates at the HTTP layer. It sees requests coming in and responses going out. When a response is sent back, the connection count decrements.

**When least connections wins:**

Anytime request duration varies significantly. Background jobs, file uploads, complex queries — anything where some requests take much longer than others. Least connections naturally avoids overloading busy servers.

---

## Algorithm 3: Consistent Hashing

Consistent hashing solves a specific problem that round robin and least connections can't — **distributed cache locality.**

Imagine you have a Redis cluster with multiple nodes instead of a single Redis instance. You need to decide which Redis node is responsible for which cached data. If the assignment changes every time you add or remove a node, you get a massive cache invalidation event — every cached item needs to be remapped, your database gets hammered with requests.

Consistent hashing places both your data keys and your server nodes on an imaginary ring. Each key is always routed to the nearest node clockwise on the ring. When you add a new node, only the keys between the new node and its predecessor need to be remapped — typically 1/N of all keys where N is the number of nodes.

Add a 4th node to a 3 node cluster — only ~25% of keys move. Without consistent hashing — potentially 100% of keys move.

**When you actually need it:**

- Multiple Redis nodes in a cluster
- Stateful servers with local caches where the same request must hit the same server
- WebSocket connections where a user must maintain connection to the same server

**When you don't need it:**

If you have a single Redis instance and stateless servers — consistent hashing adds complexity without benefit. Round robin or least connections and you're done.

---

## The Session Problem and the Real Solution

Here's a scenario that trips up a lot of engineers.

User logs in. Their session data is stored on Server 1. Next request goes to Server 2. Server 2 has no idea who this user is. They get logged out.

Two approaches to solve this:

**Sticky sessions** — the load balancer remembers which server each user was routed to and always sends them back to the same one. Solves the problem but creates new ones. Server 1 goes down — all stuck users lose their session. Server 1 gets overloaded with too many active sticky users — you can't rebalance without breaking sessions.

**Stateless architecture** — the real solution. Don't store session state on the server at all.

Session data lives in a shared external store — Redis, DynamoDB — that all servers can access. User hits Server 1, Server 2, Server 3 — doesn't matter. Any server reads the same session from the same store.

```
Compute (Lambda / ECS / EC2) — stateless, disposable
Session Store (Redis / ElastiCache) — fast, shared, persistent
Database (RDS) — permanent storage
```

With stateless servers, it doesn't matter which server handles which request. The session affinity problem disappears entirely. And this is why Lambda works so well at scale — stateless by design, state always lives externally.

---

## Health Checks: More Than Just "Is the Server Alive?"

Before sending traffic to a server, the load balancer needs to know if that server can actually handle requests.

It does this by periodically sending an HTTP request to a `/health` endpoint on each server. Healthy response — traffic flows. No response or error — server is pulled from rotation immediately.

**But here's where most engineers get it wrong.**

A health check that just returns 200 OK is not a health check. It's a heartbeat. There's a difference.

Consider: your server is running fine but its database connection pool is exhausted. Every real request is failing. But your `/health` endpoint returns 200 because it never touches the database. The load balancer thinks everything is healthy. Traffic keeps flowing. Every request fails.

**A meaningful health check verifies that the server can actually do its job:**

```
GET /health

Checks:
→ Database connectivity — can I run a simple query?
→ Redis connectivity — can I reach the cache?
→ Critical dependencies — can I reach what I depend on?

Healthy: 200 OK { status: "healthy" }
Unhealthy: 503 { status: "unhealthy", reason: "database unreachable" }
```

**The trade-off to watch:**

Too shallow — misses real problems like connection pool exhaustion.

Too deep — calling 5 external services makes your health check slow. Load balancer times out. Healthy servers get marked unhealthy.

The sweet spot: check your direct dependencies — database, cache — but don't cascade into every downstream service your application touches.

---

## The Full Picture

```
Request comes in
↓
Load Balancer
├── Health check: is this server able to handle requests?
├── Algorithm: which healthy server gets this request?
│    ├── Round robin — uniform requests, high throughput
│    ├── Least connections — variable request duration
│    └── Consistent hashing — distributed cache, WebSockets
↓
Server (stateless)
├── Session data → Redis
└── Persistent data → Database
```

---

## What This Teaches You as an Engineer

**The right algorithm depends on your architecture**

There is no universally best load balancing algorithm. Round robin for uniform stateless APIs. Least connections for variable workloads. Consistent hashing for distributed caching. Know the trade-offs, pick what fits.

**Stateless architecture solves more problems than it creates**

Sticky sessions, session affinity, server-local caching — all of these problems disappear when your servers hold no state. Design your compute layer to be stateless from day one. Put state where it belongs — in dedicated storage.

**Health checks should reflect real readiness**

A server that's running but can't reach its database is not healthy. Your health check should know the difference. Shallow health checks create blind spots. Deep health checks create false positives. Find the balance.

---

_A load balancer is only as smart as the architecture behind it. Get the architecture right and the load balancing becomes simple._
