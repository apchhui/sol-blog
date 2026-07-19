---
title: "Finding a Strategy, Part 3: Same-Block Buys on Free Data and Python"
date: 2026-07-19
summary: "How to land a transaction in the same slot as the trigger event using only Solana's public websocket, free APIs, and a VPS in North America. No Rust, no paid nodes — Python, websockets, and plain HTTPS."
translationKey: same-block
---

[Part 1]({{< relref "pump-fun-sniping" >}}) and [part 2]({{< relref "arbitrage-all-pools" >}}) ended with the same diagnosis: too much time passed between an on-chain event and my transaction. Before burying all event-driven strategies, I decided to find out: what is the minimum achievable latency if you refuse, on principle, to pay for infrastructure?

The answer surprised me: **buying in the same block (slot) as the trigger event is achievable on a fully free stack.** Solana's public websocket, the APIs of free websites, Python, the `websockets` library, and default HTTPS. No geyser plugins, no paid RPCs, no Rust.

## Where the time actually goes

A Solana slot is roughly 400 ms. To land in the same block as an event, the whole chain of "learned → decided → sent → transaction arrived and got included" has to fit inside that budget. Let's break it down:

| Link | Typical latency | Compressible for free? |
|---|---|---|
| Event → websocket notification | tens of ms (on `processed`) | partially (commitment level) |
| Network: RPC → your machine | **geography-dependent: 5–200 ms** | **yes, radically** |
| Event handling in your code | 1–5 ms | yes |
| Building and signing the transaction | 1–2 ms (if prepared in advance) | yes |
| Network: your machine → RPC | geography again | **yes, radically** |
| RPC → leader, block inclusion | out of your control | no |

The key discovery: **the fattest link is neither your code nor "slow Python" — it's geography.** From Eastern Europe, the round trip to North American RPC endpoints is 120–200 ms. The event takes ~80 ms to reach you, the transaction takes ~80 ms to get back — 160 ms, nearly half a slot, burned in transoceanic fiber.

## The $5 fix: a VPS in North America

A significant share of Solana's infrastructure — RPC endpoints, validators, relays — lives in US data centers. A cheap VPS somewhere in New York or Ashburn, Virginia changes the picture completely:

- ping to public RPCs: from ~160 ms down to **5–25 ms**;
- round trip: **~150–300 ms saved per cycle** — a full slot, sometimes a slot and a half.

This is the only paid item in the whole setup (~$5/month), and strictly speaking you're not paying for data or access — you're paying for physical distance. Every data source stays free.

## The stack: Python, websockets, HTTPS — and that's enough

The common belief is "on-chain speed requires Rust." For colocated HFT — sure. But look at the table above: "slow Python's" share of the budget is single-digit milliseconds. Optimizing that before removing geography is pointless; once geography is removed, there's nothing left worth optimizing.

The working stack:

- **`websockets` + `asyncio`** — a subscription to Solana's public endpoint (`logsSubscribe` / `blockSubscribe` where available) with `commitment: processed` — the earliest moment an event can be seen at all without geyser;
- **a plain HTTPS client** for sending transactions and hitting free APIs (screeners, price aggregators, pump.fun stats) — default `httpx`/`aiohttp`, no tricks.

The techniques that matter, each saving tens of milliseconds:

1. **Keep connections warm.** A TLS handshake to the RPC costs 1–2 round trips. Open the HTTP session in advance and never close it: the transaction must go out over an already-established connection.
2. **Cache the blockhash in the background.** A separate coroutine refreshes `getLatestBlockhash` every second. When the event fires, the transaction is assembled from ready-made parts — no network trip for a blockhash.
3. **Pre-build the transaction.** All instructions, compute budget limits, priority fee — everything is prepared before the event; the hot path only substitutes parameters, signs (microseconds), and sends.
4. **`skipPreflight: true`.** The preflight simulation is an extra round trip to the RPC; in a race for a slot it's unaffordable.
5. **Zero extra work on the hot path.** No disk logging, no JSON parsing beyond the necessary, and the buy decision comes from pre-computed filters.

## Measured results

The methodology is simple: record the slot of the trigger event (from the notification) and the slot in which my transaction confirmed — the difference is the score.

With a North American VPS, on the public websocket and a free RPC:

- a meaningful share of buys landed **in the same slot as the event**;
- the bulk landed in slot **N+1**;
- worse than N+2 almost always meant a problem (a 429, an overloaded endpoint, a stale blockhash).

For comparison, the same code running from my home machine in Europe: consistently N+2 to N+4. Moving the point of presence achieved more than rewriting the code in any language ever could.

## Honest limitations

- **Rate limits.** Public endpoints throttle frequent requests (429) and occasionally drop the websocket. You need automatic reconnect with resubscription and backoff. For a single event-driven strategy the limits are enough; for scanning the whole market they are not.
- **No guarantees.** A public RPC can lag, change its limits, or go into maintenance. This is a proof of concept and a working tool for niche strategies — not production infrastructure.
- **Same-block ≠ profit.** Parts 1 and 2 of this series are precisely about how speed only admits you to the game. But at least now we know the entry ticket is almost free.

## The conclusion of the series

I started with the question "which strategy makes money?" and lost about $400 cycling through creation sniping, bonding curve entries, migration sniping, and atomic arbitrage. I ended with a better question: **"what is the full cost of one attempt, and where in the chain is the time lost?"**

The answer to the second question turned out to be constructive: the dominant delay is geographic, and it's removed by a $5 VPS — not by a Rust rewrite and not by paid infrastructure. Free data — the public websocket and open APIs — is enough to play one-block games.

What to do with that speed is the subject of future experiments.
