# Testing async decrypt — polling pattern

## What

`FHE.decrypt(ct)` is async. The plaintext lands in a *later* transaction. Tests must model this two-tx pattern, even when running against mocks.

## The basic pattern

```
// Tx 1 — request
await contract.requestSettlement();
const ctHash = await contract.pendingHandle();   // contract stashed it

// Wait for the threshold network (or mock) to resolve
await waitForDecryption(contract, ctHash);

// Tx 2 — settle
await contract.finalizeSettlement(/* args */);
```

## The `waitForDecryption` helper (rough shape)

```
async function waitForDecryption(contract, ctHash, timeoutMs = 30000) {
  const start = Date.now();
  while (Date.now() - start < timeoutMs) {
    const [, ready] = await contract.getDecryptResultSafe(ctHash);
    if (ready) return;
    await new Promise(r => setTimeout(r, 200));
  }
  throw new Error('decryption timed out');
}
```

In mocks, the first poll usually returns ready=true. In real environments, expect 2-10 seconds depending on op complexity and network busy-ness.

## Testing the "not ready yet" path

To test that calling `finalizeSettlement` before decryption is ready reverts:

```
await contract.requestSettlement();
// Do NOT wait
await expect(contract.finalizeSettlement(...)).to.be.revertedWith('not yet');
```

In mocks this is hard (resolution is synchronous); test on testnet for the realistic path.

## Foundry equivalent

Foundry's `vm.warp` doesn't help here — decrypt readiness is enforced via the mock's state, not block time. Foundry mocks resolve synchronously, so the "two-tx" semantics are conceptual rather than literal:

```
contract.requestSettlement();
// mock has already resolved
contract.finalizeSettlement(...);
```

To test the not-yet-ready revert path in Foundry, you'd need a custom mock that exposes a "pause resolution" knob. Verify with `cofhe-foundry-mocks` source.

## Gotchas

- **Mock resolution timing varies.** Some mocks resolve immediately; some require a tick. Read the mock source to confirm.
- **Real testnet runs are slow.** Don't put 50 async-decrypt tests in a tight loop on arb-sepolia — each takes seconds.
- **Re-decrypting is idempotent on-chain** but the poll loop should still be debounced — don't spam `getDecryptResultSafe` on every render in a frontend test harness.
- **Failed decryption** (e.g., ACL missing) typically surfaces as `ready=false` indefinitely, not as a revert. Tests should set a timeout, not poll forever.
