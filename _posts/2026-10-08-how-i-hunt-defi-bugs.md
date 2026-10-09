---
layout: post
title: "How I Hunt DeFi Bugs"
date: 2026-10-08 09:00:00 +0000
categories: [guides]
tags: [methodology, defi, bug-bounty, auditing]
description: "The workflow I use to find live vulnerabilities in DeFi protocols — picking targets, reading code, and turning a hunch into a paid report."
---

Finding your first bug takes longer than anyone tells you. This is the process
I actually use — not the romantic version, the one that survives a month of
finding nothing.

<!--more-->

## Pick the right target

Most of the edge is in target selection, before you read a single line.

- **Complexity attracts bugs.** Live bugs are usually simple checks missing from
  complex paths. Large DeFi protocols, math-heavy systems, and anything with
  many integrations are where simple mistakes hide inside paths nobody fully
  traced.
- **Innovation attracts bugs.** When a team does something new — a novel AMM
  curve, a new collateral type, a custom accounting trick — some attack path was
  almost certainly left unexplored.
- **Read the audit reports first.** The count of highs and criticals tells you
  how much attention the code got. Lots of simple findings is a tell: where
  there were five, there is often a sixth.
- **Review the fixes.** Patches introduce bugs. A rushed fix for a critical is
  one of the highest-signal places to look.

## Build context before you build theories

I don't start hunting for a specific bug. I start by understanding the system
well enough that a *wrong* value jumps out at me.

1. Map the money. Where do assets enter, where do they leave, and who is allowed
   to move them.
2. Map the trust. Which roles are privileged, which inputs are attacker
   controlled, and where those two meet.
3. Map the math. Every exchange rate, index, accumulator, and price. Write down
   the invariant each one is supposed to preserve.

Once the invariants are written down explicitly, a bug is just a path that
breaks one of them.

## Where the bugs actually are

Across protocol types, the same families repeat:

- **Pricing** — spot price used where a TWAP was needed, a manipulable oracle,
  an LP token priced naively.
- **Accounting** — rounding in the wrong direction, precision loss, an index
  that can be inflated or frozen.
- **Ordering** — state updated after an external call, or a check that runs
  against stale state.
- **Edge conditions** — the first depositor, `totalSupply == 0`, an empty
  market, a paused asset, a fee-on-transfer or rebasing token the code assumes
  behaves normally.

The companion posts go deep on the first three protocol types I look at —
lending, vaults, and staking — with the concrete patterns I check every time.

## Prove it or drop it

A report you can't prove is worthless, and often worse than nothing. Before I
write anything up:

- I reproduce it in a Foundry test against a fork, with real addresses.
- I quantify the impact in funds at risk and preconditions.
- I reduce the PoC to the smallest sequence that still triggers the loss.

If I can't get the exploit to run, I treat it as a lead, not a finding.

## On getting paid

You lose all leverage the moment you disclose. So the project evaluation
matters as much as the bug: treasury size, program rules, cap, and how they've
treated past reporters. Vague rules, low caps, and prior disputes are red flags.
Expect delays and pushback even on a clean critical — stay professional, and
keep hunting while you wait.

The one rule that matters most: **don't give up early.** The gap between zero
bugs and your first one is the hardest part of the whole game.
