# Bit-shift ratios — cheap percentage approximations

## What

Encrypted multiplication and division are among the most expensive FHE operations. For powers-of-two ratios — and approximations of common percentages — bit-shifts (`FHE.shr`) plus adds/subs are dramatically cheaper.

## The patterns

```
x / 2    →  FHE.shr(x, 1)
x / 4    →  FHE.shr(x, 2)
x / 8    →  FHE.shr(x, 3)
x / 2^n  →  FHE.shr(x, n)

x * 1.5      ≈ FHE.add(x, FHE.shr(x, 1))      // x + x/2
x * 1.25     ≈ FHE.add(x, FHE.shr(x, 2))      // x + x/4
x * 1.125    ≈ FHE.add(x, FHE.shr(x, 3))      // x + x/8
x * 0.875    ≈ FHE.sub(x, FHE.shr(x, 3))      // x - x/8
x * 0.75     ≈ FHE.sub(x, FHE.shr(x, 2))      // x - x/4
```

For non-power-of-two ratios you build up combinations:

```
x * 1.875 ≈ FHE.add(x, FHE.sub(x, FHE.shr(x, 3)))   // ~= x + 7/8 x
```

## When to use

- Tier pricing in confidential markets (112.5%, 125% triggers without revealing the base).
- Rolling-window discounts that don't need exact precision.
- LP fee math where 1-2 bps of error is acceptable.

## When NOT to use

- Final settlement amounts where the imprecision compounds.
- Anything compared against a user-provided exact threshold (the approximation drift will surprise them).
- Reward math where the user can see the difference between "1/8" and "12.5%."

## Canonical example

- **RFQ — encrypted tier rating.**
  https://github.com/FhenixProtocol/rfq-demo
  → `contracts/rfq.sol`. Tier comparisons use `FHE.shr(minReceiveA, 3)` ≈ x/8 to approximate 112.5% / 125% thresholds without ever calling encrypted `mul` or `div`. Grep for `FHE.shr`.

## Gotchas

- **`FHE.shr` truncates** — same as plaintext `>>`. Round-trips are lossy.
- **Loss of precision is non-deterministic from the user's POV** — they can't see the encrypted intermediate, so document the rounding behavior in user-facing copy.
- **Bit-shifts have a fixed shift amount.** Variable shift (`FHE.shr(x, encryptedN)`) is not supported. If you need variable, you're back to expensive math.
- **For exact halving, `FHE.shr(x, 1)` is preferable to `FHE.div(x, FHE.asEuintXX(2))` — orders of magnitude cheaper.**
