---
layout: post
title: "Reentrancy in a Staking Vault — Draining Rewards"
date: 2026-10-09 12:00:00 +0000
categories: [evm, writeups]
tags: [reentrancy, solidity, bug-bounty]
description: "A checks-effects-interactions violation that let an attacker reenter claim() and drain the reward pool."
---

**Target:** `StakingVault` · **Severity:** High · **Status:** Reported / Fixed

A classic checks-effects-interactions violation let an attacker reenter
`claim()` and drain the reward pool. This is a template — replace the details
with the real finding.

<!--more-->

## Summary

Lead with what the attacker gains and what the protocol loses, in one paragraph.

## Root cause

```solidity
function claim() external {
    uint256 reward = rewards[msg.sender];
    (bool ok, ) = msg.sender.call{value: reward}(""); // external call first
    require(ok);
    rewards[msg.sender] = 0;                           // state updated after
}
```

The external call happens **before** `rewards[msg.sender]` is zeroed, so a
malicious receiver reenters `claim()` and withdraws repeatedly.

## Proof of concept

```solidity
// forge test --match-test testReentrancy -vvv
contract Exploit {
    // attacker contract + test ...
}
```

## Impact

Quantify it: funds at risk, who is affected, preconditions.

## Recommendation

Apply checks-effects-interactions, or a `nonReentrant` guard:

```solidity
rewards[msg.sender] = 0;                 // effects first
(bool ok, ) = msg.sender.call{value: reward}("");
require(ok);
```

## Timeline

- **2026-10-09** — Reported via Immunefi
- **2026-10-11** — Triaged
- **2026-10-15** — Fixed & rewarded
