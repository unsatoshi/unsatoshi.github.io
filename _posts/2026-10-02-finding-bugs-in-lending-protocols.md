---
layout: post
title: "Finding Bugs in Lending Protocols"
date: 2026-10-02 09:00:00 +0000
categories: [patterns, evm]
tags: [lending, defi, oracle, liquidation, bug-bounty]
description: "The recurring bug patterns in Aave/Compound-style lending markets — oracles, liquidations, interest accrual, and the edge cases that create bad debt."
---

Lending markets concentrate a lot of value behind a few numbers: a price, a
health factor, an interest index. Break any of them and the protocol pays out
more than it should. Here's where I look.

<!--more-->

## 1. The oracle is the soft underbelly

Almost every catastrophic lending exploit is really a pricing exploit.

- **Spot price used as truth.** If collateral value reads a DEX spot price, a
  flash loan moves it, and you borrow against inflated collateral or liquidate
  healthy positions. Check whether the feed is a manipulable spot, a TWAP, or a
  real oracle like Chainlink.
- **Stale or unchecked feeds.** Does the code check `updatedAt` and the round
  freshness? An unchecked Chainlink answer that returns a stale price during an
  outage is a classic.
- **LP tokens and wrappers priced naively.** Pricing an LP token by
  `reserves / supply` is manipulable. Pricing a rebasing or yield-bearing
  wrapper at 1:1 with the underlying is a common mistake.

```solidity
// Red flag: health check reading a manipulable spot price
uint256 price = pool.getReserves();        // spot, flash-loan movable
uint256 collateralValue = amount * price;   // inflate price -> over-borrow
```

## 2. Liquidations: rounding and incentives

- **Bad debt from rounding.** When seizing collateral and repaying debt, which
  way does each division round? If the protocol rounds in the borrower's favor
  on both sides, repeated small liquidations can leave dust debt with no
  collateral — bad debt the protocol eats.
- **Liquidation bonus math.** Can an attacker self-liquidate at a profit, or
  liquidate a position that is only barely unhealthy for an outsized bonus?
- **Close factor bypass.** Is the fraction of debt you can repay per liquidation
  enforced correctly, or can you liquidate 100% in one shot when you shouldn't?
- **Frozen/paused assets.** If one collateral asset is paused mid-liquidation,
  can a position become un-liquidatable and accrue bad debt?

## 3. Interest accrual and indexes

- **Missing accrual.** Every state-changing entry point should accrue interest
  *first*. A function that reads balances before `accrueInterest()` works on
  stale numbers — borrow/repay/liquidate on a stale index is exploitable.
- **Index precision.** Low-precision indexes lose dust every accrual. Over many
  blocks, or with a tiny market, that drift becomes real money.
- **Utilization at the boundary.** What happens at 0% and 100% utilization?
  Division by `totalBorrows` or `totalSupply` when either is zero is a frequent
  revert-or-worse.

## 4. First-depositor and empty-market edge cases

A market with near-zero liquidity behaves nothing like a full one.

- Can the first supplier manipulate the exchange rate by donating underlying
  directly to the contract, so later suppliers get shortchanged?
- Does `exchangeRate = cash / totalSupply` divide by zero, or round to a value
  the attacker chose?

## 5. Weird tokens break assumptions

- **Fee-on-transfer:** the contract credits `amount`, but receives
  `amount - fee`. Now the books overstate real balances.
- **Rebasing:** balances change out from under the accounting.
- **Reentrancy via hooks:** ERC777 `tokensReceived` or ERC677/ERC1363 callbacks
  let a token reenter deposit/borrow before state settles.

## How I test it

Fork mainnet, add the real market, and write a Foundry test that:

1. Takes a flash loan.
2. Moves the price or donates to the pool.
3. Borrows / liquidates against the distorted state.
4. Asserts the protocol ends with less than it started.

If the invariant "total debt is always backed by sufficient collateral" can be
broken in one transaction, you have a critical.
