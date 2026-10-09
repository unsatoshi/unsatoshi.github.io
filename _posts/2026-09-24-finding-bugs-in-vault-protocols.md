---
layout: post
title: "Finding Bugs in Vault Protocols (ERC-4626)"
date: 2026-09-24 09:00:00 +0000
categories: [patterns, evm]
tags: [erc4626, vaults, inflation-attack, rounding, bug-bounty]
description: "Share-price manipulation, rounding direction, and the inflation attack — the bug patterns that live in tokenized vaults."
---

ERC-4626 made vaults a standard, which means the same bugs now appear in dozens
of forks. Almost all of them come down to one question: **how many shares does a
deposit mint, and can an attacker move that number?**

<!--more-->

## 1. The inflation / first-depositor attack

The canonical 4626 bug. In an empty vault:

1. Attacker deposits 1 wei, minting 1 share.
2. Attacker *donates* a large amount of the underlying directly to the vault
   (a raw transfer, not a deposit), so `totalAssets` is huge but `totalSupply`
   is 1.
3. A victim deposits. Because shares are rounded **down**,
   `shares = assets * totalSupply / totalAssets` rounds to 0 — the victim mints
   zero shares and the attacker's single share now owns everything.

```solidity
// shares minted rounds down; with totalSupply=1 and a donated balance,
// a normal deposit can round to zero shares.
shares = assets.mulDiv(totalSupply, totalAssets, Math.Rounding.Down);
```

Check the mitigation: virtual shares/assets offset (OpenZeppelin's approach),
dead shares minted on first deposit, or an internal asset accounting that
ignores direct donations. If none are present, it's almost certainly live.

## 2. Rounding direction is the whole ballgame

Every convert must round in the protocol's favor, never the user's:

- `deposit` / `mint` → round shares **down**, assets owed **up**.
- `withdraw` / `redeem` → round shares **up**, assets paid **down**.

A single `convertToShares` or `convertToAssets` that rounds the wrong way lets a
user loop deposit/withdraw and extract dust each cycle. Dust at scale, repeated
in a loop, is a real finding. Read every `mulDiv` and confirm the rounding
argument.

## 3. totalAssets manipulation

`totalAssets()` is the price oracle of the vault.

- If it reads the vault's raw token balance, direct donations move it (see #1),
  and in strategy vaults a flash-deposit into the underlying strategy can move
  it within a transaction.
- If it reads a market price of strategy positions, it inherits that oracle's
  manipulability.
- Can `totalAssets` be inflated right before a victim's `withdraw` preview and
  deflated after?

## 4. Preview vs. actual

`previewDeposit/previewRedeem` must match what the state-changing call actually
does. If `preview*` ignores a fee or a rounding step that the real call applies,
integrators relying on the preview get a worse execution than quoted — and
sometimes that gap is extractable.

## 5. Hooks and reentrancy

If deposit/withdraw calls out — a transfer to an ERC777 underlying, a strategy
`beforeWithdraw` hook, a callback — an attacker can reenter while
`totalSupply` and `totalAssets` are mid-update and mint/redeem against an
inconsistent ratio.

## 6. Slippage: is there any?

Raw 4626 `deposit/withdraw` have no min-out parameter. If the vault doesn't wrap
them with `minShares` / `minAssets`, users eat whatever the ratio is at
execution — ripe for sandwiching when `totalAssets` is movable.

## How I test it

In Foundry, against a fresh vault:

1. Deposit 1 wei as the attacker.
2. Donate a large balance directly.
3. Have a victim deposit a normal amount.
4. Assert the victim's `maxWithdraw` is far less than they put in.

If the victim can lose a meaningful fraction to a one-wei attacker, that's your
report.
