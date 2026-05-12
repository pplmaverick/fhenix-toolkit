# Randomness via user-contributed entropy

## What

`FHE.randomEuintXX` exists but it puts a single node (the FHE engine) in a trusted position over the seed. For applications where any single trusted seed is unacceptable — fair shuffles, sealed-bid auctions, gift-exchange — collect entropy from each participant and combine it on-chain.

## The pattern

```
Each joiner contributes encrypted entropy:
  function join(InEuint32 userEntropy) external {
    euint32 contribution = FHE.asEuint32(userEntropy);
    game.entropy = FHE.xor(game.entropy, contribution);
    FHE.allowThis(game.entropy);
  }

Final randomness is derived from the combined entropy:
  euint32 shift = FHE.add(
    FHE.rem(game.entropy, FHE.asEuint32(uint32(n - 1))),
    FHE.asEuint32(1)
  );
  // shift is in [1, n-1] — guarantees no self-assignment in a rotation shuffle
```

## The rule

- **XOR-combine, don't sum.** XOR has the property that one honest contributor in a set of dishonest ones still randomizes the result. Addition does not.
- **Don't reveal contributions individually.** Each participant's `userEntropy` should be `allowThis`'d but not `allowSender`'d or `allowPublic`'d.
- **The last joiner has an information advantage.** They see the combined entropy before submitting (well — they see the handle, but cannot decrypt it without their own permit). Mitigations: timed phases (commit before any reveal), or accept the imperfection if stakes are low.
- **Beware modulo bias** for small ranges. `FHE.rem(entropy, n)` is unbiased only when `2^bits` is divisible by `n`. For tiny `n` (e.g. n=3, n=5) and `euint32` (2^32), the bias is negligible. For very large `n` close to `2^32`, do rejection sampling.

## Canonical example

- **Secret Santa — combined entropy with XOR.**
  https://github.com/FhenixProtocol/encrypted-secret-santa
  → `packages/hardhat/contracts/SecretSantaFactory.sol`. The `joinGame` function XOR-combines each joiner's `userEntropy` into `game.entropy`. The shuffle uses `FHE.rem(entropy, n-1) + 1` to get a rotation shift in `[1, n-1]`.

## Gotchas

- **`FHE.randomEuintXX` is fine for cases where the FHE engine can be trusted.** Don't conflate "FHE means trustless" — the engine ultimately holds keys.
- **`FHE.xor` on integers behaves bitwise**, same as plaintext `^`. For booleans use the `FHE.xor(ebool, ebool)` overload.
- **A participant who refuses to submit entropy can grief.** Design phases with a deadline — if entropy not in by T, the game proceeds with whatever's collected.
- **The combined entropy is itself a sensitive value** — never `allowPublic` it before the shuffle is committed; otherwise an adversary can recompute assignments.
