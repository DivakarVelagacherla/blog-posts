---
title: "Why Uber Generates a Different OTP for Every Ride While Rapido Uses the Same One"

date: 2026-03-01

slug: "why-uber-generates-different-otp-for-every-ride-while-rapido-uses-same-one"

summary: "Same feature, two companies, two completely different designs. Here's what the OTP difference between Uber and Rapido teaches you about security trade-offs."

categories: Software Architecture

readTime: 5
---

## The Observation Nobody Explains

You book an Uber. Driver arrives. Uber shows you a 4 digit OTP. You tell the driver. Driver enters it. Ride starts.

You book a Rapido. Same OTP every single time. You already know it before the ride.

Same feature. Two companies. Two completely different designs.

Most engineers never stop to ask why. Here's what that difference actually teaches you.

---

## Why Uber Uses a Dynamic OTP

Uber's OTP is generated fresh for every single ride. Only the rider sees it. The driver can only start the ride if the real rider tells them the code.

The threat Uber is protecting against: **wrong vehicle boarding.**

In a car, you're getting into an enclosed space with a stranger. A fake driver could show up before your real driver. Without an OTP, you might get in the wrong car. With a dynamic OTP — a fake driver has no way to know your code. They can't start the ride. You're protected.

And because the OTP is generated fresh every ride — even if someone learned your OTP from a previous ride, it's useless. It expired the moment that ride ended.

This is the principle of **ephemeral credentials** — credentials that exist for one purpose and expire immediately after. Limited damage window. Even if leaked, the attacker has nothing.

---

## Why Rapido Uses a Static OTP

Rapido's OTP is tied to your account, not your ride. It never changes.

The obvious question: isn't that a security risk?

Yes. But Rapido made a conscious trade-off.

**1. Different threat model**

Rapido is primarily bike taxis. You're getting on a bike in open traffic — you can see the driver, the vehicle, the plate. The wrong vehicle boarding risk is lower than in a car. The security requirement is different.

**2. Simpler user experience**

On a moving bike in noisy traffic, sharing a new OTP every single ride adds real friction. A static OTP you've memorized removes that friction entirely. For Rapido's market and price point, that UX trade-off made sense.

**3. Cost and complexity**

Generating, storing, and validating dynamic OTPs per ride requires infrastructure. For a startup optimizing for growth and low cost — static OTP is simpler to build and maintain.

---

## How Rapido Compensates for the Weaker OTP

Rapido doesn't rely solely on OTP for safety. They layer other protections:

- **Real time GPS tracking** — every ride tracked from start to finish
- **Driver verification** — license, vehicle, background check before onboarding
- **SOS button** — panic button that alerts emergency contacts and Rapido support instantly
- **Trip sharing** — share live location with trusted contacts

This is **defense in depth** — a core security principle. No single layer is perfect. You stack multiple layers so that breaking one doesn't compromise everything.

Rapido's OTP layer is weaker. Their other layers compensate. The overall system is still reasonably safe.

---

## What This Teaches You as an Engineer

### 1. Security is a threat model, not a checklist

Before designing any security feature ask: what am I actually protecting against? Who is the attacker? What's the realistic attack scenario?

Uber's threat model demanded dynamic OTPs. Rapido's threat model accepted static ones. Same feature. Different right answers. Because the context was different.

### 2. Defense in depth

Don't rely on one security mechanism. Stack multiple layers so that a weakness in one doesn't compromise everything. Rapido's weak OTP plus strong GPS tracking plus SOS plus driver verification adds up to a reasonably safe system.

You'll apply this principle everywhere — authentication, authorization, data encryption, API security. Never one layer. Always multiple.

### 3. Security trade-offs are product decisions

Security always costs something — complexity, user experience, development time, infrastructure. The decision of how much security is enough is not purely technical.

As an engineer your job is to surface the trade-offs clearly. Not make the decision unilaterally. Uber and Rapido both made conscious trade-offs. Neither is objectively wrong.

### 4. Ephemeral vs persistent credentials

Dynamic OTP per ride = ephemeral credential. Exists for one purpose, expires immediately.

Static OTP = persistent credential. Lives forever.

Ephemeral credentials are almost always more secure because the damage window of a leak is limited. You'll see this pattern everywhere:

- JWT tokens with short expiry
- One-time payment tokens
- Temporary AWS credentials via IAM roles
- Magic login links that expire in 15 minutes

When you design any system involving credentials or access — ask yourself: **does this need to be persistent or can it be ephemeral?**

The answer to that question will shape your security design more than any framework or library you choose.

---

## The Bottom Line

Uber and Rapido didn't make different choices because one team was more security-conscious than the other. They made different choices because they have different users, different threat models, and different product constraints.

That's the engineering lesson. Security decisions are context-dependent trade-offs — not universal rules.

The most secure design is not always the right design. The right design is the one that matches your actual threat model, your users, and the resources you have to build and maintain it.

---

_The best security decision isn't the most sophisticated one. It's the one that matches your actual threat._
