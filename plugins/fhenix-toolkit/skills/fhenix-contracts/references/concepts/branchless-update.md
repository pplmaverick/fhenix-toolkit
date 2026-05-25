# Branchless updates with FHE.select

## What

Solidity cannot branch on encrypted booleans. `if (FHE.gt(a, b))` does not work — `ebool` is a ciphertext handle, not a runtime `bool`. Use `FHE.select(cond, ifTrue, ifFalse)` instead: both arms execute, the encrypted condition picks the result.

## The rule

Anywhere you'd write `if (encrypted_cond) x = a; else x = b;`, write:

```
x = FHE.select(encrypted_cond, a, b);
FHE.allowThis(x);   // if x is stored
```

Both arms compute. There is no short-circuit. Avoid expensive operations in either branch.

## Canonical examples

- **Sealed-bid auction — winner update.** The reference implementation.
  https://github.com/FhenixProtocol/poc-sealed-bid-auction
  → `packages/hardhat/contracts/SealedBidAuction.sol`, function `bid()`. Grep for `FHE.select` — both `highestBid` and `highestBidder` are updated branchlessly using a shared `isHigher` ebool, with `FHE.allowThis` after each.

- **FHERC20 balance-bounded transfer.** Clamps a transfer to the sender's balance without revealing either.
  https://github.com/marronjo/fhe-hooks
  → `src/HybridFHERC20.sol`. Grep for `FHE.select(amount.lte(`. Both arms of the select run (the "send" and "send zero" paths); the encrypted comparison picks.

- **Gameplay feedback (Wordle-style).** Per-digit feedback via `FHE.select` on encrypted equality.
  https://github.com/FhenixProtocol/miniapp-equle
  → `packages/hardhat/contracts/Equle.sol`, function `guess()`.

## When Claude needs the actual code

Use `WebFetch` against the file URL, grep for the function name, quote ~10-20 lines. Do not paste whole files.

## Gotchas

- **Always `FHE.allowThis(result)` after the select** if the result is stored. Without it, the contract can't reuse the value next transaction.
- **For `eaddress`, the false arm must also be an `eaddress`.** Use `FHE.asEaddress(address(0))` as a sentinel if you don't have a meaningful "previous" value.
- **Both branches always run.** Don't put expensive operations in either side just because you assume one side "wins."
- **Type unification.** Both arms must be the same encrypted type. Cast explicitly if needed.
- **`FHE.select(ebool, eaddress, eaddress)` works**; `select` is overloaded for all encrypted types.
