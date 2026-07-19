---
title: "Finding a Strategy, Part 1: Sniping on pump.fun — New Tokens, the Bonding Curve, and Migrations"
date: 2026-07-05
summary: "How I went through three flavors of pump.fun sniping — buying at token creation, entering on bonding curve progress, and sniping migrations — and how each one lost money in its own way."
translationKey: pump-sniping
---

This is the first post in a series about my search for a working trading-bot strategy on Solana. Spoiler: I lost a few hundred dollars along the way, but I came out understanding how the on-chain market actually works — and that turned out to be worth more than what I spent.

## Why pump.fun

To a beginner algo trader, pump.fun looks like the perfect sandbox: thousands of new tokens per day, a single contract, predictable bonding curve mechanics, and a migration to a full AMM pool once the threshold is reached. Everything is event-driven, everything can be read from the logs of one program. It feels like being slightly faster and slightly smarter than the crowd should be enough for the math to be on your side.

That's how plan number one was born.

## Version 1: sniping new tokens

The idea is simple: subscribe to the pump.fun program logs over a websocket (`logsSubscribe` by program id), catch the `create` instruction, decode the new token's mint, and buy within the first blocks after creation. Then sell into the pump as buyers pile in.

Technically it all worked quickly: I saw the creation event within hundreds of milliseconds, and my buy transaction landed one or two slots later. The problem wasn't speed — it was the premise itself.

The reality is that the overwhelming majority of new pump.fun tokens are either an instant dump by the creator, or a bundle where the dev buys their own token in the same transaction as the creation and then unloads on whoever comes in next. That is, on the snipers. On me.

My stats over a couple of weeks looked roughly like this:

- ~70% of snipes: down 50–90% within minutes — the token simply died;
- ~25%: exited around break-even or at a small loss after fees;
- ~5%: real profit, occasionally a 2–3x.

The rare multi-baggers didn't cover the pile of corpses. On top of that, every trade paid the pump.fun fee (1%), a priority fee, and slippage on both entry and exit. Creator filters (wallet age, deploy history) cut out some of the garbage — but also some of that precious 5%.

Version 1 result: about **−$150** and the first important realization — *in creation sniping you are, by definition, someone's exit liquidity, unless you can tell a "live" launch from an assembly-line scam better than the other bots can.*

## Version 2: buying on bonding curve progress

Fine, I thought — why catch tokens at zero, where 95% are trash? The bonding curve is a built-in filter. A token that has reached 70–90% of the curve has already attracted real buys, survived the first dumps, and is approaching migration — an event that usually comes with hype.

The bot monitored curve states, computed progress (SOL collected relative to the migration threshold), and entered tokens crossing a chosen threshold — planning to hold until migration and sell into the migration pump.

What went wrong:

- **Progress isn't an arrow, it's a pendulum.** A curve at 85% will happily roll back to 60%: early buyers take profit on the people entering "on progress." I turned out to be the person they took profit on.
- **A stalled curve is dead money.** A token can sit at 80% for hours or days. Capital is frozen, and on pump.fun a day is an eternity: the crowd's attention has long moved to other tickers.
- **The finish line is crowded.** Near 95–100% sit bots just like mine, all wanting to enter before migration and exit right after. Entry slippage ate a noticeable chunk of the expected edge.

Version 2 result: about **−$80** and a lesson: *curve progress is an indicator of past demand, not future demand. It tells you people bought — it says nothing about who will buy after you.*

## Version 3: sniping migrations

Third attempt — the cleanest event in a pump.fun token's life cycle: the migration. The curve fills up, liquidity moves to an AMM pool, and the first seconds of trading in the new pool often see a spike. The migration is detectable from logs, the timing is predictable from the curve itself — everything invites a precise strike.

This is where I first experienced a real bot race. The first blocks after pool creation are a stampede: dozens of transactions with obscene priority fees, some transactions simply failing, and the ones that do land moving the price so much that your entry ends up 10–20% above your estimate. Then the classic: the initial pump — and a dump from those who bought on the curve and were waiting for exactly this moment to unload into the migration crowd.

A few times it worked beautifully: entry in the new pool's first block, exit half a minute later at +30–40%. But on average the picture was the same: the winners of this race are those with faster infrastructure, bigger capital, and the budget for aggressive fee wars. Plus a cost line I initially underestimated: **failed transactions cost money too** — the base fee and the priority fee burn even when the swap doesn't execute.

Version 3 result: roughly **−$70**, a noticeable share of which was fees for transactions that accomplished nothing at all.

## What I understood after all three versions

1. **Speed is necessary but not sufficient.** I genuinely achieved decent latency (there will be a separate post about it — it's possible even on free infrastructure), but speed only buys you a ticket to the game, not a win.
2. **In games against the crowd, your counterparty isn't the crowd — it's the other bots.** Their authors are solving the same problem, with the same data access, often with better infrastructure.
3. **Fees are a first-class strategic variable.** Hundreds of small trades and reverts quietly eat the deposit even when your trading is "flat."
4. **I needed a strategy where the profit is mathematically locked inside the transaction, instead of depending on whether a buyer shows up after me.**

Point four led me straight to arbitrage: if the same token trades in several pools at different prices, you can buy cheap and sell dear atomically, in a single transaction — either profit or revert. Sounds like the solution to everything, right?

More on that in [part two]({{< relref "arbitrage-all-pools" >}}).
