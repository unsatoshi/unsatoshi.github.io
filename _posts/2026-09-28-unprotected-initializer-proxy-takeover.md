---
layout: post
title: "Unprotected Initializer — Proxy Takeover"
date: 2026-09-28 10:00:00 +0000
categories: [evm, writeups]
tags: [access-control, proxy, upgradeable, bug-bounty]
description: "A missing initializer guard on an upgradeable contract let anyone claim ownership and upgrade the implementation."
---

**Target:** `UpgradeableToken` · **Severity:** Critical · **Status:** Reported / Fixed

A missing `initializer` guard on an upgradeable (UUPS) contract let anyone call
`initialize()` after deployment, claim ownership, and upgrade the implementation
to arbitrary code. Template — swap in the real details.

<!--more-->

## Summary

On an upgradeable contract the constructor does not run in the proxy's context,
so initialization happens via `initialize()`. If that function is callable more
than once — or was never called on deployment — an attacker front-runs it and
becomes `owner`.

## Root cause

```solidity
function initialize(address owner_) external {   // no initializer modifier
    _transferOwnership(owner_);
}
```

With no `initializer` modifier (or no `_disableInitializers()` in the
implementation's constructor), anyone can call `initialize()` and seize control.

## Proof of concept

```solidity
// Attacker calls initialize() with their own address, then upgradeTo(evil)
vm.prank(attacker);
token.initialize(attacker);
assertEq(token.owner(), attacker);
```

## Impact

Full takeover: attacker owns the proxy and can upgrade to a malicious
implementation, draining or freezing all funds.

## Recommendation

```solidity
function initialize(address owner_) external initializer {
    _transferOwnership(owner_);
}

constructor() { _disableInitializers(); }   // in the implementation
```

## Timeline

- **2026-09-28** — Reported
- **2026-09-30** — Confirmed Critical
- **2026-10-04** — Reinitialized / fixed
