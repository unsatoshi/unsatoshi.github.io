---
layout: post
title: "Finding Bugs in Staking Protocols"
date: 2026-09-16 09:00:00 +0000
categories: [patterns, evm]
tags: [staking, rewards, accounting, precision, bug-bounty]
description: "Reward accounting, precision loss, and timing games — where staking and reward-distribution contracts leak value."
---

Staking contracts look simple and almost never are. The bugs live in the reward
accounting — the `rewardPerToken` accumulator pattern that half the ecosystem
copied from Synthetix, gotchas and all.

<!--more-->

## 1. The accumulator pattern, and where it breaks

Most reward contracts track a global `rewardPerTokenStored` and a per-user
`userRewardPerTokenPaid`, accruing on every stake/withdraw/claim:

```solidity
function rewardPerToken() public view returns (uint256) {
    if (totalSupply == 0) return rewardPerTokenStored;
    return rewardPerTokenStored
        + (lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / totalSupply;
}
```

Things to check:

- **Is `updateReward` called on *every* balance-changing path?** Miss one — a
  transfer, a migration, an emergency withdraw — and users accrue against stale
  checkpoints. Over- or under-payment follows.
- **`totalSupply == 0` windows.** Rewards streamed while nobody is staked are
  lost (stuck) or, worse, retroactively captured by the first staker after the
  gap. Which one is it here?

## 2. Precision loss eats small stakers

That `* 1e18 / totalSupply` loses precision when `totalSupply` is large or
`rewardRate` is small. If `rewardRate * 1e18 < totalSupply`, `rewardPerToken`
can round to **zero** for whole periods and rewards silently vanish. Check the
scaling factor against realistic token decimals and supply.

## 3. Flash-stake reward theft

If rewards can be claimed in the same transaction as staking, an attacker can:

1. Flash-loan a huge amount of the stake token.
2. Stake, immediately claim a disproportionate slice of a just-distributed
   reward, and unstake.
3. Repay the loan.

The fix is usually time-weighting or a lock. The bug is when distribution
(`notifyRewardAmount`) lands as a lump that the current `totalSupply` splits
instantly — sandwich the distribution and you capture it.

## 4. notifyRewardAmount gotchas

- **Rounding down `rewardRate`.** `rewardRate = reward / duration` truncates;
  the remainder is stranded in the contract forever unless swept.
- **Extending during an active period.** When a new reward is added mid-period,
  the leftover must be rolled in correctly (`leftover = remaining * rewardRate`).
  Get the arithmetic wrong and you either strand funds or let the rate be
  inflated.
- **Reward token == stake token.** If they're the same asset, does
  `notifyRewardAmount` accidentally let staked principal be counted as, or
  drained as, rewards?

## 5. Reentrancy on claim

`getReward` transfers the reward token before zeroing the user's owed balance,
or the reward token has a transfer hook — either lets an attacker reenter and
claim twice. Checks-effects-interactions, or a guard.

## 6. Withdraw/exit edge cases

- Can you withdraw more than you staked due to a rounding or checkpoint bug?
- Does `exit()` (withdraw + claim) stay consistent if one half reverts?
- Emergency withdraw paths that skip `updateReward` are a recurring source of
  stuck or double-counted rewards.

## How I test it

Fork or deploy locally, then:

1. Stake with two accounts of very different sizes across several reward
   periods.
2. Assert total rewards paid never exceeds total rewards funded.
3. Try the flash-stake sequence around a `notifyRewardAmount` call.

The invariant to break is simple: **sum of claimable rewards ≤ rewards
deposited.** If you can push it over, the contract is insolvent in rewards.
