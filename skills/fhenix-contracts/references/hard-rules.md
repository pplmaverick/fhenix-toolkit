# Hard rules — fhenix-contracts

These rules are timeless and version-independent. Violating any of them produces a bug; some bugs are silent — the contract compiles, but ciphertexts are inaccessible or revealed unintentionally.

## Rule 1: No `if` / `require` / `while` on encrypted booleans

`ebool` is a ciphertext handle. Solidity's `if` / `require` / `while` need runtime booleans. Encrypted control flow doesn't exist in Solidity; both branches always execute.

```
// Wrong — does not behave like you think
ebool isHigher = FHE.gt(a, b);
if (isHigher) { ... }
require(isHigher, "too low");

// Right
euint64 newValue = FHE.select(isHigher, a, b);
FHE.allowThis(newValue);
```

See `concepts/branchless-update.md`.

## Rule 2: `FHE.allowThis(result)` after every stored encrypted op

Every time you store an encrypted value to state (or pass it to another contract), call `FHE.allowThis(x)`. Without it, the contract cannot operate on `x` in any future transaction — the threshold network's ACL rejects access.

```
// Wrong — count is unreachable next tx
count = FHE.add(count, FHE.asEuint32(1));

// Right
count = FHE.add(count, FHE.asEuint32(1));
FHE.allowThis(count);
```

See `concepts/allow-cascade.md`.

## Rule 3: Avoid encrypted `mul` and `div` when `shr` suffices

Encrypted multiplication and division are expensive. For powers-of-two ratios and approximations, use `FHE.shr`:

- `x / 2`   →  `FHE.shr(x, 1)`
- `x / 4`   →  `FHE.shr(x, 2)`
- `x * 7/8` ≈ `FHE.sub(x, FHE.shr(x, 3))`
- `x * 9/8` ≈ `FHE.add(x, FHE.shr(x, 3))`

This is how RFQ approximates 112.5% / 125% tiers without `mul` / `div`. See `concepts/bit-shift-ratio.md`.

## Rule 4: No allowance for ciphertexts

`approve` / `transferFrom` cannot be replicated for encrypted balances — the amount is encrypted, so the allowance check would have to happen against a ciphertext, and `require` doesn't work on `ebool` (Rule 1).

Use the operator pattern instead: `setOperator(operator, until)` + `isOperator(...)`. See `concepts/operator-pattern.md`.

## Rule 5: `trivialEncrypt` exposes plaintext

`FHE.asEuintXX(<literal>)` calls `trivialEncrypt(literal, ...)` under the hood. The literal travels through calldata in cleartext. Only safe for constants you don't mind revealing (loop bounds, sentinels). For user-supplied secret values, the user must encrypt off-chain and pass an `InEuintXX`.

## Rule 6: Decryption is asynchronous

`FHE.decrypt(ct)` is a request. It does not return the plaintext synchronously. To read the result, call `FHE.getDecryptResultSafe(ct)` (or `getDecryptResult(ct)` which reverts if not ready) in a **later transaction**. Never write code that expects synchronous reads of decrypted values within the same function. See `concepts/async-decrypt.md`.

## Rule 7: Confidentiality is not anonymity

FHE hides values, not the transaction graph. Observers see:

- That an address called your contract
- Which function they called
- Gas used
- That events fired (and any plaintext fields in them)

Confidential dApps that need anonymity layer mixers or shielded pools on top of FHE.

## Rule 8: Order of `allow*` calls matters

Call `FHE.allowThis(x)` (and other `allow*` calls) **before** emitting events or returning. The off-chain network observes handles and expects ACL bits to be set by the time it sees them.

## Rule 9: Uninitialized encrypted state acts as zero

`euint32 x;` in storage starts at the zero handle. Operations on uninitialized state act as if it were `FHE.asEuint32(0)`. Don't rely on "is this set" semantics — track presence with a separate plaintext flag if needed.

## Rule 10: Both branches of `FHE.select` always execute

There's no short-circuit. If one arm is expensive, both costs are paid. For `eaddress`, the false arm must also be an `eaddress` — use `FHE.asEaddress(address(0))` as a sentinel if no meaningful prior value exists.
