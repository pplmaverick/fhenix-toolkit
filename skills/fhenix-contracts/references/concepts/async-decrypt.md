# Async decrypt — request and poll

## What

`FHE.decrypt(ct)` is a request to the threshold network, not a read. The plaintext becomes available in a later transaction. To act on it, the contract calls `FHE.getDecryptResultSafe(ct)` (non-reverting) or `FHE.getDecryptResult(ct)` (reverts if not ready) in a follow-up function.

## The flow

```
Tx 1 — request:
  function requestSomething() external {
    euint64 ct = ...;            // some encrypted value
    FHE.allowPublic(ct);         // OR FHE.allow(ct, addr) depending on flow
    FHE.decrypt(ct);             // queues the decrypt task; returns immediately
    pendingHandle = euint64.unwrap(ct);   // stash the raw ct-hash for tx 2
  }

Tx 2 — settle (called later, after the network finishes):
  function settle() external {
    (uint64 plaintext, bool ready) = FHE.getDecryptResultSafe(euint64.wrap(pendingHandle));
    require(ready, "not yet");
    // act on plaintext: transfer funds, branch, emit, etc.
  }
```

## The rule

- **Two transactions, minimum.** The request and the settlement cannot be in the same tx.
- **Store the unwrapped ct-hash** (`euint64.unwrap(ct)`) between the request and the settlement so you can re-wrap it later. The handle's identity is the hash.
- **Poll, don't busy-wait.** A frontend or keeper triggers tx 2 when the network is done (typically seconds after tx 1).
- **`getDecryptResult` reverts if not ready**; use `getDecryptResultSafe` and check the `ready` flag if you want graceful retry.

## Canonical examples

- **Sealed-bid auction — full request + settle cycle.**
  https://github.com/FhenixProtocol/poc-sealed-bid-auction
  → `packages/hardhat/contracts/SealedBidAuction.sol`. Two-step flow: `requestSettlement()` calls `FHE.allowPublic` then `FHE.decrypt`; `finalizeSettlement(winner, amount, proofs)` runs in a later tx after the client has fetched the result and packaged the signatures.

- **fhe-hooks — decryption via queue.**
  https://github.com/marronjo/fhe-hooks
  → `src/market-order/MarketOrder.sol` queues ct-hashes in a `DoubleEndedQueue` and processes them in `_afterSwap` once `getDecryptResultSafe` reports `ready`.

## Gotchas

- **`FHE.decrypt` requires the right ACL.** If neither `allowPublic` nor `allow(ct, network_authority)` is set, the network refuses. Check the decrypt-flow decision tree in `decision-trees.md`.
- **Latency is variable.** Network busy-ness, security zone, op complexity all affect time-to-ready. Don't hard-code timeouts.
- **Re-decrypting the same ct is idempotent** — calling `FHE.decrypt(ct)` twice doesn't double-queue or cost extra. Safe to retry.
- **Don't store plaintexts longer than needed.** Once you've acted on the value, consider zeroing the slot — the plaintext is now a public blockchain fact.
