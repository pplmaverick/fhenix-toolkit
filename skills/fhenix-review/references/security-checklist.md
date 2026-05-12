# Security checklist — fhenix-review

A concrete walkthrough to apply before signing off on confidential CoFHE code. Treat as a punch-list.

## Contract layer

- [ ] Every stored encrypted write is followed by `FHE.allowThis(result)`.
- [ ] Every encrypted handle returned to (or shared with) another address has a corresponding `FHE.allow(ct, addr)` / `FHE.allowSender(ct)`.
- [ ] Every `FHE.allowPublic(ct)` is intentional and paired with a settlement / verification step.
- [ ] No `if` / `require` / `while` on `ebool`. All conditional updates use `FHE.select`.
- [ ] Both arms of every `FHE.select` are the same encrypted type and neither leaks via expensive side effects.
- [ ] `trivialEncrypt(literal)` is only used for non-secret constants.
- [ ] No `FHE.decrypt(...)` calls remain (that initiator has been removed from the library); reveals route through the SDK-mediated `decryptForView` / `decryptForTx` flow, with on-chain `FHE.verifyDecryptResult` for tx-bound reveals.
- [ ] Storage slots that hold decrypted plaintexts are zeroed after use, OR the design accepts the resulting plaintext-on-chain.
- [ ] No encrypted `approve` semantics; uses operator pattern (`setOperator` / `isOperator`) with explicit expiry.
- [ ] Randomness for fair shuffles / sealed bids uses user-contributed XOR entropy, not `FHE.randomEuintXX`.
- [ ] `InEuintXX` inputs that constrain protocol invariants are accompanied by "Proof of Plaintext Input" (zk-proof that the cleartext meets bounds), or the design accepts unbounded inputs.
- [ ] `allow*` calls precede any event emission referencing the handle.

## SDK / off-chain layer

- [ ] Every `decryptForView` is paired with a `getOrCreateSelfPermit` call (or assumes a prior one).
- [ ] Every `decryptForTx(...).withoutPermit()` is paired with `FHE.allowPublic(ct)` on the corresponding contract call.
- [ ] Every `decryptForTx(...).withPermit()` is paired with `FHE.allow(ct, user)` and a permit scoped to the right name.
- [ ] Permit `expiration` uses `Math.floor(Date.now() / 1000) + DURATION_SECONDS` (NOT raw `Date.now()`).
- [ ] Permit `name` is scoped per-cycle if state cycles, app-wide if it doesn't.
- [ ] Errors are caught as `CofheError` (or generic `Error` fallback), not Result-tuple-destructured.
- [ ] In SSR frameworks: the SDK client is wrapped in a Proxy or otherwise gated to browser-only access.
- [ ] `.connect(publicClient, walletClient)` is called after wallet ready and re-runs on chain/account change.

## Threat model

- [ ] What does a passive chain observer see? List the *facts* leaked: function calls, tx graph, gas, event existence.
- [ ] What does a malicious contributor of inputs see / influence? (e.g., entropy contributors in randomness flows)
- [ ] What's revealed by failure paths? (a `require` that reverts can leak — though FHE prevents most direct value-based reverts, gas profile may still leak.)
- [ ] If the dApp promises "private," can the promise be falsified by graph analysis, gas analysis, or event correlation?
- [ ] What's the dispute / griefing surface? (e.g., a participant who refuses to submit their entropy share.)

## Composability

- [ ] If interacting with other confidential tokens, are they on the same standard? (`ERC20Confidential` ≠ FHERC20 ≠ ERC-7984.)
- [ ] If using a wrapper / unwrapper, has it been audited?
- [ ] Are encrypted amounts ever stored in slots that off-chain indexers might decrypt with a leaked permit?

## Deployment & ops

- [ ] Security zone matches the deployed chain.
- [ ] FHE.sol and `@cofhe/sdk` versions are pinned (not `latest`).
- [ ] Mock-environment gas estimates were validated against testnet before mainnet deploy.
- [ ] Tests exercise the async decrypt poll loop (not just the request).

## Output format

When reviewing, produce:

```
## Critical
- <issue, with file:line and quote>
- <fix or follow-up question>

## High
- ...

## Medium
- ...

## Low / Notes
- ...
```

If no issues, briefly summarize what was checked and why you're confident.
