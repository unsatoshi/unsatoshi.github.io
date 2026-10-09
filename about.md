---
layout: default
title: About
---
## About {{ site.name }}

<img class="user-avatar" src="{{ site.owner.avatar }}">

*Trust God. Fuzz Code.*

Independent smart contract security researcher focused on the EVM.

I break protocols before attackers do — reentrancy, access control, oracle and
price manipulation, accounting and rounding bugs, and broken cross-contract
invariants — and publish the findings here as writeups with reproducible
proofs of concept.

**Focus**

- Audits &amp; bug bounties on live protocols
- Foundry — fuzzing, invariant testing, PoC development
- Solidity today; Rust / Solana next

> Responsible disclosure only. No unauthorized testing.

<div class="pagination">
  {% if site.owner.twitter %}
    <a href="https://twitter.com/{{ site.owner.twitter }}" class="social-media-icons"><i class="fa-brands fa-2x fa-square-x-twitter" aria-hidden="true"></i></a>
  {% endif %}
  {% if site.owner.github %}
    <a href="{{ site.owner.github }}" class="social-media-icons"><i class="fa-brands fa-2x fa-square-github" aria-hidden="true"></i></a>
  {% endif %}
</div>
