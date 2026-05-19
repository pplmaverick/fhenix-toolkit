# Pattern leakage — gas, events, tx graph as side channels

## What

Even when values are encrypted, the *shape* of execution can leak information. Three common side channels:

### 1. Gas profile

Different op paths cost different amounts of gas. If branch A involves `FHE.mul + FHE.shr + FHE.select` and branch B involves only `FHE.add`, an observer can distinguish them by gas alone.

**Mitigation:** make both arms of every conditional structurally identical at the op level, even if values differ. Pad cheaper arms with no-op encrypted operations.

> **Padding closes the gas channel but isn't free.** Each no-op encrypted op still costs real gas and adds latency to the FHE engine's queue. Budget for it — don't claim "private and cheap" when you've added a mul to every branch.

### 2. Event-fact leakage

Even with all event arguments encrypted, the *occurrence* of an event is public. If `BidPlaced(encryptedAmount, encryptedBidder)` fires only when the bid exceeds the reserve, the event itself reveals that the bid was high enough.

**Mitigation:** fire the event unconditionally with a "did it count?" encrypted flag, or batch events so the timing/count doesn't correlate with private state.

### 3. Tx-graph topology

Which addresses call which contracts in what order is fully public. A privacy claim that hinges on hidden routing is broken if `tx graph: User → RouterA → AMM → User` is visible.

**Mitigation:** layered solutions (private mempools, batching, off-chain matching with on-chain net settlement).

## Review checklist

For each privacy-sensitive flow:

- [ ] Does gas usage differ between cases that *should* be indistinguishable?
- [ ] Does any event fire/not-fire in a way that leaks encrypted state?
- [ ] Does the tx graph reveal what the dApp claims to hide?
- [ ] Does block timing correlate with private state (e.g., reveals are always at a specific time)?

If any answer is yes, classify the leakage severity and flag in the review.

## Examples in the ecosystem

- **AlphaEngine's "balanced batch"** is designed so the net settlement reveals nothing about individual intents.
- **Encrypted gaming** moves to commit-reveal alternatives to avoid per-action ZKPs that obscure fairness.
- **Encrypted lending** publishes only health-factor binary indicators (`HF ≥ 1`), not exact ratios, to defeat "precision targeting."

## Source

- https://www.fhenix.io/blog/alphaengine-fixing-defis-leaky-foundation
- https://www.fhenix.io/blog/encrypted-lending-ethereum-fully-homomorphic-encryption-private-defi
