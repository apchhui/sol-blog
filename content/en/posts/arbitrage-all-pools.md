---
title: "Finding a Strategy, Part 2: Arbitraging Every Available Pool — Beautiful Math, Brutal Practice"
date: 2026-07-12
summary: "After sniping, I moved on to atomic arbitrage across all Solana pools: Raydium, Orca, Meteora, PumpSwap. The profit was locked inside the transaction — yet someone kept taking it before me."
translationKey: pools-arbitrage
---

In [part one]({{< relref "pump-fun-sniping" >}}) I described how three flavors of pump.fun sniping — buying at creation, entering on bonding curve progress, and sniping migrations — left me about $300 poorer. The main takeaway: I needed a strategy where profit doesn't depend on whether the next buyer shows up after me. Arbitrage looked like exactly that.

## The idea

On Solana the same token often trades in several pools at once: Raydium AMM v4, Raydium CLMM, Orca Whirlpools, Meteora DLMM, PumpSwap. After every large trade in one pool, prices briefly diverge: in pool A the token just got more expensive, in pool B not yet.

The plan:

1. Index **every available pool** for the selected tokens — not just the biggest ones, but literally everything with any liquidity.
2. For each pair of pools, compute the cross rate and search for a cyclic route `SOL → token (pool A) → SOL (pool B)` that stays positive after fees.
3. Execute both legs **atomically in a single transaction**: if the final SOL balance hasn't grown, the whole transaction reverts.

The beauty of atomicity is that classic market risk disappears entirely: you cannot "buy and fail to sell." Either profit or revert. It seemed like the only way to lose money had been engineered away.

A way was found.

## What the paper trading showed

For the first weeks the bot ran in observer mode: computing spreads, logging "virtual" trades. The picture was encouraging: dozens of opportunities per hour with 0.5–2% spreads, and 3–5% after volatile minutes. On paper the strategy made several percent a day. I switched on real execution.

## What happened with real money

### 1. The spread lives shorter than your transaction flies

An opportunity exists from the moment it appears until someone takes it. On Solana that is usually **less than one slot** — under ~400 milliseconds. Professional arbitrageurs don't "chase" the imbalance like I did: their transactions land in the same block as the trade that created the imbalance — including via Jito bundles, where a searcher pays to have their transaction placed immediately after the target one.

My chain of "websocket event → recompute → build transaction → send via RPC" delivered the transaction to the network 1–3 slots later. By then the pool was already rebalanced. Atomicity did its honest job: the transaction reverted.

### 2. A revert is free only for the market, not for you

A failed transaction still gets paid for: base fee plus priority fee. Pennies each — but an aggressive strategy produces hundreds of reverts a day. My log for one fairly quiet day: 412 transactions sent, 9 successful, 403 reverts. Profit from the successful ones: ~$6. Fees for everything: ~$11. And that was a *good* day — the strategy reliably generated a small, steady loss, perfectly disguised as "almost working."

### 3. The long tail of pools turned out to be toxic

"Every available pool" sounded like an edge: smaller pools have wider spreads and less competition. In practice, the liquidity tail is a minefield:

- pools where the advertised 5% spread evaporates on any size above $20 — price impact eats everything;
- tokens with a **freeze authority**, where your token account can simply be frozen;
- tokens with transfer fees and other non-standard mechanics that turn a "profitable" route into a guaranteed loss;
- bait pools created specifically for bots like mine.

Every filter against this zoo shrank the universe of opportunities, and what remained were exactly the competitive pools where I kept getting outrun.

### 4. Free infrastructure hits a ceiling

Public RPCs return 429s under frequent requests, cap subscriptions, and — most importantly — the path to them is simply not built for a race measured in hundreds of microseconds. Atomic arbitrage on the majors is an infrastructure competition: colocation with validators, direct connections, Jito, your own nodes. Python didn't lose to that — it was playing a different sport in a different league.

## The money, totalled

| Stage | Result |
|---|---|
| Sniping token creations | ≈ −$150 |
| Entries on bonding curve progress | ≈ −$80 |
| Sniping migrations | ≈ −$70 |
| Arbitraging all pools (revert fees) | ≈ −$90 |
| **Total** | **≈ −$390** |

## Key takeaways

1. **Atomicity removes market risk, not competitive risk.** You can't lose to a price move — but you can pay indefinitely for the right to lose the race.
2. **Paper-traded arbitrage lies by construction.** It shows opportunities that existed, but not the fact that they were taken 50 ms before your transaction would have reached the leader.
3. **Bot strategies must be judged by the full cost of an attempt**, not by the average trade: reverts, 429s, stuck transactions — all of that is cost of goods sold.
4. **Latency is the through-line of every one of these stories.** In every strategy I hit the same wall: too much time passed between the event and my transaction.

Which is why my next step was to stop searching for a strategy and start attacking latency. It turned out that even **on entirely free data sources** — Solana's public websocket and the APIs of free websites — you can get your buy **into the same block as the event**. No Rust, no geyser, no paid nodes: Python, plain websockets, and default HTTPS. You just have to remove one — the main — delay.

That's [part three]({{< relref "same-block-buys" >}}).
