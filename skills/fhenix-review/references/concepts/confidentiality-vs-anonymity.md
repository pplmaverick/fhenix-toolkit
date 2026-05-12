# Confidentiality vs. anonymity

## The distinction

- **Confidentiality** — hiding the *values* in a computation. FHE delivers this.
- **Anonymity** — hiding the *parties* and their *relationships*. FHE alone does NOT deliver this.

A dApp can be fully confidential and fully de-anonymized at the same time. Real example: an encrypted-DEX swap hides the amounts, but the public tx graph shows Alice → DEX → Bob with timestamps. Anyone watching can infer the link.

## What FHE hides

- The cleartext value of any `euint*` / `ebool` / `eaddress`.
- The plaintext of computations performed on those values.
- The intermediate state of multi-step encrypted flows.

## What FHE does NOT hide

- That an address called the contract.
- Which function they called.
- Gas consumed by the call (which correlates with op complexity).
- That an event fired (and its non-encrypted fields).
- The sequence of contract interactions (tx graph).
- Block timing.

## Layered solutions for anonymity

If a dApp promises anonymity, it needs additional infrastructure on top of FHE:

- **Mixers / shielded pools** for address unlinkability (Tornado-style, but for confidential tokens).
- **Private mempools** (Flashbots, SUAVE) for tx-graph obfuscation between submission and inclusion.
- **Encrypted-intent batching** for amount-and-pair unlinkability (see AlphaEngine pattern).
- **Off-chain matching with on-chain settlement** to limit graph exposure to net flows.

## Review hooks

When reviewing a "private" dApp, ask:

1. What does the dApp claim about privacy in user-facing copy?
2. What does a passive observer actually see (the threat-model layer)?
3. Is the gap between (1) and (2) acceptable for the use case? Or does the marketing oversell?

If (1) claims anonymity and (2) shows full tx-graph visibility, the gap is misleading. Either fix (1) (honest copy) or build the additional layer.

## Source

- https://www.fhenix.io/blog/anonymity-on-ethereum-fhenix-flutons-path-to-programmable-privacy — the canonical treatment from Fhenix.
