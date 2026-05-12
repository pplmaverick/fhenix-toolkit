---
name: fhenix-contracts
description: Use when working on Solidity files importing @fhenixprotocol/cofhe-contracts/FHE.sol or using euint*/ebool/eaddress types. Teaches confidential contract patterns — branchless updates with FHE.select, the four ACL verbs, the three decryption flows, and the confidential-token standards picker. Activates on Fhenix CoFHE contract work.
---

# fhenix-contracts — write confidential Solidity for Fhenix CoFHE

You activate when the user is writing or editing Solidity that uses Fhenix's FHE library: files importing `@fhenixprotocol/cofhe-contracts/FHE.sol`, files using `euint*` / `ebool` / `eaddress` types, or prompts like "build a confidential X contract."

## Hard rules — never violate

1. **Never branch on encrypted booleans.** `if (FHE.gt(a, b))` does not work. `ebool` is a ciphertext handle, not a runtime `bool`. Use `FHE.select(cond, ifTrue, ifFalse)` instead. See `references/concepts/branchless-update.md`.

2. **Call `FHE.allowThis(x)` after every encrypted op whose result is stored.** Without it, the contract cannot reuse `x` in any future transaction — the network's ACL rejects access. See `references/concepts/allow-cascade.md`.

3. **Prefer `FHE.shr` over encrypted `mul` / `div` for ratios.** Encrypted division is expensive; bit-shifts approximate percentages cheaply. See `references/concepts/bit-shift-ratio.md`.

4. **There is no allowance for ciphertexts.** ERC-20's `approve` / `transferFrom` doesn't translate. Use the operator pattern: `setOperator(operator, until)` + `isOperator(...)`. See `references/concepts/operator-pattern.md`.

5. **`trivialEncrypt` is not private.** The plaintext is visible in calldata. Use it only for constants you don't mind revealing.

6. **Decryption is asynchronous.** `FHE.decrypt(ct)` is a request, not a read. Poll `FHE.getDecryptResultSafe(ct)` in a later transaction. See `references/concepts/async-decrypt.md`.

7. **Confidentiality is not anonymity.** FHE hides values, not the transaction graph. Anyone watching the chain still sees who called what, when, and for how much gas.

Full rule list with explanations: `references/hard-rules.md`.

## The four ACL verbs

| Verb | Grants | Call when |
|---|---|---|
| `FHE.allowThis(ct)` | The current contract | After every encrypted write that's stored — mandatory. |
| `FHE.allowSender(ct)` | `msg.sender` | The caller will decrypt off-chain (e.g. show their own balance). |
| `FHE.allow(ct, addr)` | Specific address | Pass the ciphertext to another contract, or grant a known user. |
| `FHE.allowPublic(ct)` | Everyone | Intentional protocol reveal. Pair with off-chain `decryptForTx(...).withoutPermit()` + on-chain `FHE.verifyDecryptResult(...)`. |

Decision tree: `references/decision-trees.md` (ACL section).

## The three decryption flows

When the contract needs a value as plaintext, pick one:

1. **Async on-chain decrypt + poll.** `FHE.decrypt(ct)` → in a later tx, `FHE.getDecryptResultSafe(ct)` → `(plaintext, ready)`. The contract itself acts on the plaintext. Used in auction settlement.
2. **Client decrypts + contract verifies.** Contract calls `FHE.allowPublic(ct)` or `allow(ct, user)`. Client calls `decryptForTx(ctHash).withoutPermit().execute()` (or `.withPermit()`), gets `{decryptedValue, signature}`, and calls back into the contract which validates with `FHE.verifyDecryptResult(...)`. Used in Equle's "claim victory."
3. **Client-side reveal only.** Contract calls `FHE.allow(ct, user)`. Client calls `decryptForView(ctHash, utype).execute()` — purely off-chain. Contract never sees the plaintext. Used in Secret Santa's target reveal.

Decision tree: `references/decision-trees.md` (decryption-flow section).

## Confidential-token standards picker

| Standard | Use when |
|---|---|
| `ERC20Confidential` (Fhenix's own) | Vanilla confidential ERC-20; Fhenix's standard is sufficient. |
| Vendored FHERC20 | You need to extend transfer semantics (callbacks, AVS, custom approval flows). |
| ERC-7984 (Ethereum draft) | You want cross-protocol composability with the emerging Ethereum standard. |

Detailed picker: `references/concepts/confidential-token-standards.md`.

## Concepts to read on demand

Each `references/concepts/<name>.md` is one focused pattern with links to canonical examples in real public repos. **Read on demand** — don't load them all up front:

- `branchless-update.md` — `FHE.select` instead of `if` / `require`.
- `allow-cascade.md` — re-grant access after every derivation; permissions don't inherit.
- `encrypted-input.md` — `InEuintXX` handling from client to chain.
- `async-decrypt.md` — submit + poll pattern.
- `bit-shift-ratio.md` — cheap percentage approximations via `FHE.shr`.
- `operator-pattern.md` — replacing ERC-20 allowance for ciphertexts.
- `randomness-via-entropy.md` — user-contributed entropy XOR (no trusted seed).
- `confidential-token-standards.md` — picking ERC20Confidential / FHERC20 / ERC7984.

## Looking up the FHE.sol API surface

For function signatures, op×type availability, exact ACL semantics, or gas behavior, **never recall — always look up**. Read `references/lookup-recipes.md` for the canonical commands. The on-chain library evolves; lookup recipes always find current truth.

## Default workflow for any encrypted-state function

1. Receive `InEuintXX calldata` from the client (if user-supplied input).
2. Convert via `FHE.asEuintXX(in)` — this verifies the signature and registers the handle.
3. Perform encrypted operations, branchless via `FHE.select`.
4. **Call `FHE.allowThis(result)`** for anything stored.
5. Call `FHE.allowSender(result)` if the caller needs to decrypt it off-chain.
6. Call `FHE.allow(result, addr)` for any downstream contract that needs access.
7. Pick a reveal path (one of the three flows above) — and **don't reveal what doesn't need to be revealed**.

Verify each step against a concept file before writing. When in doubt, look up.
