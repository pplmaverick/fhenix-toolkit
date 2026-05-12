# Allow-cascade — re-grant access after every derivation

## What

Permissions on a ciphertext do not propagate to its descendants. If `x` is `allowThis`'d, the result of `FHE.add(x, y)` is **not** — it's a brand-new handle with no ACL entries. Every encrypted operation produces a fresh handle, and that handle must be granted access explicitly.

## The rule

After **every** encrypted op whose result is stored in state or passed to another contract:

```
result = FHE.<op>(...);
FHE.allowThis(result);                   // mandatory if stored
FHE.allowSender(result);                 // if the caller needs to decrypt it
FHE.allow(result, addr);                 // if another contract / specific user needs it
```

If you skip `allowThis` on a stored handle, the contract cannot read or operate on it in any future transaction. Failures are runtime, not compile-time — usually surfacing as opaque "permission denied" errors from the threshold network.

## Canonical examples

- **Counter (canonical).** The simplest demonstration. https://www.fhenix.io/blog/what-is-fhenix shows the pattern:
  ```
  count = FHE.add(count, FHE.asEuint32(1));
  FHE.allowThis(count);
  FHE.allowSender(count);
  ```

- **Sealed-bid auction — two stored handles per write.**
  https://github.com/FhenixProtocol/poc-sealed-bid-auction
  → `packages/hardhat/contracts/SealedBidAuction.sol`, function `bid()`. Each `FHE.select` produces a new handle; both `highestBid` and `highestBidder` get explicit `FHE.allowThis` after the select.

- **FHERC20 transfer — four cascading allows.**
  https://github.com/marronjo/fhe-hooks
  → `src/HybridFHERC20.sol`. After updating both `encBalances[from]` and `encBalances[to]`, the code calls `allowThis` + `allow(addr)` on each — four `allow*` calls per transfer.

## Gotchas

- **Reading state into a local does NOT inherit ACL.** Once you do `euint64 x = encBalances[user];`, operating on `x` and storing the result back still requires fresh `allow*` calls. **Why:** every FHE operation produces a brand-new ciphertext handle (a fresh hash). The new handle has no ACL entries until you set them — permissions don't propagate from the inputs. So `FHE.add(x, y)` creates a third handle that knows nothing about `x`'s or `y`'s permissions; you must call `FHE.allowThis(result)` (and any `allowSender` / `allow`) on it explicitly.
- **`allowThis` is cheap but not free.** It's a storage write to the TaskManager's ACL mapping. Don't call it on transient handles you won't reuse.
- **Order matters.** Call all `allow*` calls **before** emitting events or returning. The off-chain network observes handles via events and expects ACL bits set by then.
- **`allowTransient` exists but is identical to `allow` from Solidity's view** — the transience is enforced by the network, not the contract.
