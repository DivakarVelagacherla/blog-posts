---
title: "What Actually Happens When You Make an API Call"

date: 2026-04-20

slug: "what-actually-happens-when-you-make-an-api-call"

summary: "You make hundreds of API calls a day — but what actually happens between the tap and the data appearing on screen? Most engineers have never stopped to think about it."

categories: Learning → Explanation

readTime: 6
---

# What Actually Happens When You Make an API Call

## The Tap That Hides a Thousand Steps

You open your weather app. You tap it. Half a second later you see today's forecast.

As an engineer you probably think: _"The app made a GET request, got the data back, rendered it."_

That's true. But it's also hiding almost everything that actually happened.

Between that tap and the number appearing on your screen, your phone and a server somewhere in the world went through an entire choreography — one that most engineers never stop to think about. Until something breaks. And then suddenly, understanding each step is the difference between finding the bug in 10 minutes and spending 3 hours in the dark.

Let's walk through every step.

---

## Step 1: DNS Resolution — Translating a Name to an Address

Your app knows the server as `api.weather.com`. But the internet doesn't work on names. It works on IP addresses — numerical addresses like `142.250.80.46`.

Before your request goes anywhere, your phone asks: _"Where is [api.weather.com](http://api.weather.com)?"_

This is **DNS — Domain Name System.** Think of it as the internet's phone book. You look up a name, you get back an address.

The flow:

1. Your phone checks its local DNS cache — has it looked this up recently?
2. If not, it asks your router
3. Router asks your ISP's DNS server
4. If nobody knows, it works its way up to the root DNS servers that know everything
5. Eventually an IP address comes back and gets cached for future use

**Why this matters:** DNS resolution adds latency to your first request. Subsequent requests to the same domain are faster because the IP is cached. This is why the first load of an app sometimes feels slower — DNS hasn't been resolved yet.

---

## Step 2: TCP Handshake — Establishing the Connection

Now your phone knows the IP address. But it can't just start sending data. First it needs to establish a reliable connection with the server.

This is the **TCP handshake** — a three step process:

1. **SYN** — your phone says _"I want to connect"_
2. **SYN-ACK** — the server says _"Got it, I'm ready"_
3. **ACK** — your phone says _"Great, let's go"_

Only after this three-way handshake is the connection considered established and data can flow.

**Why TCP and not UDP?**

TCP guarantees delivery. Every packet sent is acknowledged. If something is lost, it's retransmitted. This matters for API data — you need complete, ordered responses.

UDP is faster but unreliable — packets can arrive out of order or not at all. It works for video streaming or gaming where a missed frame is invisible. But for structured API data where completeness matters, TCP is the right choice.

**Cost:** The TCP handshake typically adds ~50ms depending on the distance between client and server.

---

## Step 3: TLS Handshake — Securing the Connection

You've noticed websites use HTTPS — the S stands for secure. The protocol that provides that security is **TLS (Transport Layer Security).**

Before your actual request is sent, your phone and the server need to agree on how to encrypt the conversation. Think of it like two people agreeing on a secret language before they start talking — so nobody in the middle can eavesdrop.

The TLS handshake:

1. Server proves its identity — _"I am actually [api.weather.com](http://api.weather.com) — here's my certificate"_
2. Client verifies the certificate is legitimate
3. Both sides agree on encryption keys
4. Connection is now encrypted

Only after this does your actual request go through.

**Cost:** TLS handshake adds another ~100-300ms on top of the TCP handshake.

**Real world impact:** A misconfigured TLS certificate — say the certificate is set up incorrectly and the client has to retry the handshake multiple times — can add hundreds of milliseconds to every request. The user sees a blank screen or stale data. The app looks broken. But the bug has nothing to do with your code. It's in the infrastructure beneath your code.

This is why understanding the full request lifecycle matters. You can't debug what you don't know exists.

---

## Step 4: The HTTP Request — Finally Sending Your Data

Now the connection is established and secured. Your actual GET request goes through.

The request travels from your device to the server carrying:

- The method (GET, POST, PUT etc)
- The path (`/api/weather?city=london`)
- Headers (authentication tokens, content type, accepted formats)
- Body if applicable (for POST/PUT requests)

On the server side, your request hits the HTTP server first. The server listens for incoming connections, parses the raw HTTP request, and hands it to your application.

This is why your backend application needs to run inside a server. Your application code doesn't know how to speak raw TCP or HTTP — the server handles that layer. Your controller just sees a clean, parsed request object.

From there your application processes the request — calls the database, runs business logic, builds a response — and sends it back through the same connection.

---

## Step 5: HTTP/2 — How Modern Apps Handle Multiple Requests

Here's the problem with what we've described so far.

A modern app doesn't make one API call on startup. It makes many. A social media feed might need posts, notifications, messages, ads, and suggested friends — all on load. Each from a different service.

In **HTTP/1.1**, each request needed its own connection. Browsers worked around this by opening up to 6 parallel connections per domain — but that's still 6 separate TCP and TLS handshakes.

**HTTP/2** solved this with **multiplexing** — multiple requests can be sent simultaneously over a single connection. One TCP handshake. One TLS handshake. All requests fly in parallel over the same pipe.

So your app on startup:

1. Opens one connection to the server
2. TCP handshake — once
3. TLS handshake — once
4. All API calls go out simultaneously over that one connection
5. Responses come back as they're ready

You don't manage any of this yourself. The HTTP client library and browser handle connection pooling automatically — maintaining a pool of already-open connections and reusing them for new requests.

---

## Step 6: The Event Loop — How the Frontend Handles It All Without Multithreading

Here's something that surprises most frontend engineers.

JavaScript is **single threaded.** There is literally one thread. No multithreading.

And yet an app with 4 components making 4 simultaneous API calls on page load works perfectly. No freezing. No blocking. How?

The answer is the **event loop** and **asynchronous programming.**

JavaScript doesn't sit and wait for a response. It says _"go fetch this, and when it comes back call me."_ Then it moves on. When the response arrives, the event loop picks it up and handles it.

Your 4 components on page load:

1. Component 1 fires API call — _"let me know when it's back"_
2. Component 2 fires API call — _"let me know when it's back"_
3. Component 3 fires API call — _"let me know when it's back"_
4. Component 4 fires API call — _"let me know when it's back"_

All 4 go out almost simultaneously. HTTP/2 multiplexes them over one connection. Responses come back whenever they're ready. The event loop handles each one as it arrives.

This is exactly what Observables and Promises abstract for you. Under the hood — event loop. You just subscribe and react.

And that debounce pattern on a search box? Same thing. You're not blocking the thread waiting for the user to stop typing. You're saying _"call me after 300ms of silence"_ and the event loop handles the rest.

---

## The Full Picture

Next time you make an API call, this is what's actually happening:

```
App makes request
       ↓
DNS Resolution — translates domain to IP
       ↓
TCP Handshake — establishes connection (SYN → SYN-ACK → ACK)
       ↓
TLS Handshake — secures connection (certificate → encryption keys)
       ↓
HTTP Request — your actual data goes through
       ↓
Server receives → App processes → DB called → Response built
       ↓
Response travels back through the same connection
       ↓
Event loop picks it up → Component renders the data
```

Every one of these steps can be a bottleneck. Every one of these steps can fail in a different way.

---

## Why This Matters in Practice

You can't debug what you don't know exists.

- Slow first load? Could be DNS resolution time
- Intermittent failures on first request? Could be TCP connection timeout
- 400ms added to every request out of nowhere? Could be a TLS misconfiguration forcing handshake retries
- App feels slow despite fast database queries? Could be HTTP/1.1 making sequential requests that HTTP/2 would parallelize

Understanding the full request lifecycle doesn't just make you better at system design interviews. It makes you faster at debugging production issues — because you know exactly where to look.

---

_The engineers who find bugs fastest aren't the ones who know the most technologies. They're the ones who understand what happens between the technologies._
