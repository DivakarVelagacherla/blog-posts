---
title: "Spotify Plays Music Even When Your Internet Cuts Out Mid Song. How?"

date: 2026-03-29

slug: "spotify-plays-music-even-when-your-internet-cuts-out"

summary: "You go underground. No signal. The song keeps playing. Here's every layer of Spotify's architecture that makes that possible."

categories: Backend Engineering

readTime: 5
---

## The Experience

You're listening to Spotify on your commute. You go underground. No signal. The song keeps playing perfectly.

Signal comes back. Next song loads instantly.

This doesn't happen by accident. There are multiple layers working together to create that experience. Here's every one of them.

---

## Two Completely Different Storage Mechanisms

Before going deeper, it's important to separate two scenarios that feel identical to the user but work completely differently under the hood.

**Scenario 1 — Normal streaming**

You're just listening. Nothing explicitly downloaded. Spotify streams the song in real time.

**Scenario 2 — Downloaded playlists**

You tapped "Download" on a playlist. Full songs are stored permanently on your device. Network is irrelevant.

The offline experience in each scenario is achieved differently. Let's look at both.

---

## How Normal Streaming Survives Connection Drops

### Buffering Ahead

When you're streaming normally, Spotify doesn't wait for you to need the next 10 seconds before fetching them. It buffers roughly **30 seconds ahead** of your current position continuously.

So when you go underground:

- 0-30 seconds underground — playing from buffer, you notice nothing
- 30+ seconds underground — buffer exhausts, brief pause, resumes when signal returns

That 30 second buffer is why a short tunnel never interrupts playback.

### Prefetching the Next Song

While your current song is playing, Spotify quietly prefetches the beginning of the next track in your queue. When you hit next — it's instant. No loading spinner.

### Adaptive Streaming

Spotify offers three quality settings — low (24kbps), normal (160kbps), high (320kbps). In auto mode, Spotify adapts quality based on your real time connection speed.

Strong WiFi → 320kbps chunks

Connection weakens → drops to 160kbps or 96kbps

Connection recovers → steps back up

This is the same concept as YouTube's Adaptive Bitrate Streaming — applied to audio. But audio chunks are dramatically smaller than video chunks.

A 10 second audio chunk at 320kbps is roughly 400KB.

A 10 second video chunk at 1080p is roughly 20MB.

That 50x size difference means Spotify can buffer far more seconds ahead than YouTube can with video. Which is why audio streaming feels more resilient to connection drops than video streaming.

---

## How Downloaded Playlists Work

When you explicitly download a playlist, full songs are stored permanently on your device in Spotify's local storage. The network becomes completely irrelevant for those songs.

### Cache Management and LRU Eviction

Spotify has a local cache with a size limit. When it fills up it needs to decide what to remove.

The strategy is **LRU — Least Recently Used.** The song you listened to least recently gets evicted first. Songs you play regularly stay. Songs you haven't touched in weeks get dropped.

You've probably experienced this — you come back to a song you haven't played in months and there's a brief loading pause. That's a cache miss. Spotify evicted it and is refetching from the server.

---

## The Full Picture

```
Song playing
↓
30 seconds buffered ahead in memory
↓
Next song prefetched in background
↓
Adaptive bitrate adjusts quality per chunk based on connection speed
↓
If connection drops → play from buffer
If buffer exhausts → brief pause, resume when signal returns
If downloaded → play from local storage, network irrelevant
↓
LRU cache eviction manages downloaded storage limits
```

Multiple layers. Each one handles a different failure scenario. The user experiences one seamless thing.

---

## What This Teaches You as an Engineer

### 1. Design for offline as a first class feature

YouTube's offline is an afterthought — you manually download individual videos. Spotify treats offline as a core feature — downloaded playlists, automatic cache management, LRU eviction. The whole system assumes the network will disappear.

If your users are on the go, design for intermittent connectivity from day one. Not as a feature you add later.

### 2. Same principle, different constraints

Spotify and YouTube both use chunking, adaptive bitrate, and prefetching. But audio chunks are 50x smaller than video chunks. That difference changes how much you can buffer, how quickly you can adapt, and how long offline playback lasts.

Don't assume solutions from one domain transfer directly to another. Understand the constraints first.

### 3. Precompute and cache at every layer

Spotify doesn't fetch the next song when you hit next. It fetched it while the current song was still playing. Anticipate what the user needs next and have it ready before they ask.

This applies everywhere — prefetching search results, preloading the next page, warming caches before traffic spikes.

### 4. Graceful degradation over hard failure

Spotify doesn't stop when the connection weakens. It steps down quality. It plays from buffer. It pauses only as a last resort. The experience degrades smoothly rather than failing suddenly.

Design your systems to handle degraded conditions gracefully. The happy path is easy. The resilient path is what separates good systems from great ones.

---

_The best user experiences are built on systems that plan for failure before it happens._
