# Proof of Plaintext Input — validating InEuintXX arrivals

## What

When a user submits an `InEuintXX` to a contract, the on-chain library verifies the signature and registers the handle — but it does NOT verify the underlying plaintext satisfies any application-specific invariant. The plaintext could be anything from `0` to `2^N - 1`.

If the protocol's correctness depends on the cleartext meeting bounds (e.g., "amount > 0 and < user's balance" or "guess is between 1 and 100"), the contract must enforce those bounds — either:

1. **Via encrypted comparison** (`FHE.gt`, `FHE.lt`) combined with `FHE.select` to clamp, OR
2. **Via off-chain ZK proof** that the plaintext meets the constraint (a "Proof of Plaintext Input").

## Pattern 1: Clamp on-chain

```
function bid(InEuint64 calldata bidIn) external {
  euint64 bid_ = FHE.asEuint64(bidIn);
  euint64 bal = encBalances[msg.sender];
  ebool valid = FHE.and(FHE.gt(bid_, FHE.asEuint64(0)), FHE.lte(bid_, bal));
  euint64 clamped = FHE.select(valid, bid_, FHE.asEuint64(0));
  // operate on clamped instead of bid_
}
```

The user can submit any value, but only valid ones have effect.

## Pattern 2: zk-Proof of Plaintext

The user submits both the `InEuintXX` AND a separate zk-proof (Groth16, PLONK, etc.) that the underlying plaintext satisfies the desired predicate. The contract verifies the proof on-chain before accepting the input.

Heavier (zk-proof gas costs) but stronger — invalid inputs are rejected before they enter encrypted state.

Reference: the "Cracking the Code" blog series describes this pattern. SherLOCKED uses Risc0 Bonsai for verifiable computation.

## Review heuristic

For each function accepting `InEuintXX`:

1. Does the protocol's correctness depend on the plaintext meeting bounds?
2. If yes, does the contract enforce those bounds (via clamping or zk-proof)?
3. If neither: **Medium-High** finding. Adversarial inputs may produce undefined behavior, gas griefing, or state corruption.

## Source

- https://www.fhenix.io/blog/cracking-the-code-overcoming-challenges-of-on-chain-fhe-part-1
- https://www.fhenix.io/blog/cracking-the-code-overcoming-challenges-of-on-chain-fhe-part-2
