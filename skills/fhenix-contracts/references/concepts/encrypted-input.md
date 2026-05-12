# Encrypted input — InEuintXX handling

## What

User-supplied encrypted values enter the contract as `InEuintXX` (or `InEbool`, `InEaddress`) — a struct holding the ciphertext hash, security zone, type tag, and a signature from the SDK. The on-chain library verifies the signature and registers the handle when you call `FHE.asEuintXX(in)`. The returned `euintXX` is the usable handle.

## The flow

```
Off-chain (client, @cofhe/sdk):
  client.encryptInputs([Encryptable.uint64(secretValue)]).execute()
    → returns InEuint64 { ctHash, securityZone, utype, signature }

Client passes the struct as a calldata argument:
  writeContract({ ..., args: [{ ctHash, securityZone, utype, signature }] })

On-chain (your contract):
  function takeInput(InEuint64 calldata inVal) external {
    euint64 v = FHE.asEuint64(inVal);   // verifies signature, registers handle
    // v is now usable — operate, store, allow* as needed
    FHE.allowThis(v);
    FHE.allowSender(v);
  }
```

## The rule

- **Always declare the input type explicitly** (`InEuint64 calldata`, not `bytes`). The Solidity type matches the SDK's `Encryptable.uintN`.
- **`asEuintXX` reverts on signature mismatch, type tag mismatch, or expired security zone.** No fallback casting.
- **Each input is single-use per transaction** — verifying the same `InEuint64` twice in one tx works, but the handle is the same.

## Canonical examples

- **Sealed-bid auction — encrypted bid input.**
  https://github.com/FhenixProtocol/poc-sealed-bid-auction
  → `packages/hardhat/contracts/SealedBidAuction.sol`, function `bid(InEuint64 calldata bidIn)`. The first line of the body is `FHE.asEuint64(bidIn)`.

- **Equle — multi-input guess.**
  https://github.com/FhenixProtocol/miniapp-equle
  → `packages/hardhat/contracts/Equle.sol`, function `guess(InEuint128 inEq, InEuint16 inResult)`. Two different-width inputs handled in one function.

- **Secret Santa — user-contributed entropy.**
  https://github.com/FhenixProtocol/encrypted-secret-santa
  → `packages/hardhat/contracts/SecretSantaFactory.sol`, function `joinGame`. Receives `InEuint32 userEntropy`, XOR-combines it with existing entropy.

## Gotchas

- **`trivialEncrypt` (via `FHE.asEuintXX(literal)`) is NOT the same as `asEuintXX(InEuintXX)`.** The literal version passes the plaintext through calldata in cleartext.
- **The `signature` field is from the user's wallet, not from the SDK alone.** Stale signatures (different account, different chain) revert.
- **`utype` must match the function signature exactly.** Passing an `InEuint128` to a function expecting `InEuint64` reverts on the signature check.
- **Security zones can expire.** If the user's SDK fetched keys from an old zone, `asEuintXX` reverts. Have them refresh the client.
