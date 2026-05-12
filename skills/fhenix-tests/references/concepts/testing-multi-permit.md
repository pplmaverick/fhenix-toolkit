# Testing multi-permit and cross-user flows

## What

Some confidential dApps involve multiple parties holding different permits over the same encrypted state. Tests need to exercise the full permission graph — not just "user A can decrypt their own value."

Patterns:
- Two users each have their own encrypted balance; each can only decrypt their own.
- An ACP (Access Control Permit) lets a verifier decrypt a holder's value.
- A protocol-public reveal at settle exposes a previously-private value to everyone.

## Pattern: two users, separate permits

```
const [alice, bob] = await ethers.getSigners();
const aliceSdk = await cofhejs_initializeWithHardhatSigner(alice);
const bobSdk = await cofhejs_initializeWithHardhatSigner(bob);

await aliceSdk.permits.getOrCreateSelfPermit({ issuer: alice.address, name: 'myapp' });
await bobSdk.permits.getOrCreateSelfPermit({ issuer: bob.address, name: 'myapp' });

// Deposit encrypted into the contract as Alice
const [encAmount] = await aliceSdk.encryptInputs([Encryptable.uint64(100n)]).execute();
await contract.connect(alice).deposit(encAmount);

// Alice can decrypt her balance
const balCt = await contract.balanceOf(alice.address);
const { decryptedValue: aliceBal } = await aliceSdk.decryptForView(balCt, FheTypes.Uint64).execute();
expect(aliceBal).to.equal(100n);

// Bob CANNOT decrypt Alice's balance
const aliceBalCt = await contract.balanceOf(alice.address);
await expect(bobSdk.decryptForView(aliceBalCt, FheTypes.Uint64).execute()).to.be.rejected;
```

## Pattern: ACP (selective disclosure)

```
// Holder creates an ACP for the verifier
const acp = await holderSdk.permits.createPermit({
  issuer: holder.address,
  recipient: verifier.address,
  name: 'compliance',
  expiration: Math.floor(Date.now() / 1000) + 3600,
});

// Verifier uses it
verifierSdk.permits.importPermit(acp);   // exact API varies — verify
const { decryptedValue } = await verifierSdk.decryptForView(holderBalCt, FheTypes.Uint64).execute();
expect(decryptedValue).to.equal(expectedBalance);

// A third party (eve) cannot
await expect(eveSdk.decryptForView(holderBalCt, FheTypes.Uint64).execute()).to.be.rejected;
```

## Pattern: protocol-public reveal at settle

```
// Contract calls FHE.allowPublic(winnerCt) inside requestSettlement
await contract.requestSettlement();

// Anyone can decrypt without a permit
const { decryptedValue: winner, signature } = await anySdk
  .decryptForTx(winnerCt).withoutPermit().execute();

// Pass through the contract's verifier
await contract.finalizeSettlement(winner, signature);
```

## Canonical examples

- **Multi-permit / helper migration.**
  Internal `cofhe/tests/contracts/helper.sol` and corresponding test — exercises multi-permit + ACL grant ordering.

- **Sealed-bid auction E2E.**
  https://github.com/FhenixProtocol/poc-sealed-bid-auction/blob/main/packages/hardhat/test/ — multi-bidder flow with one settler.

- **Secret Santa multi-participant.**
  https://github.com/FhenixProtocol/encrypted-secret-santa/blob/main/packages/hardhat/test/ — each participant has a different encrypted target visible only to them.

## Gotchas

- **The SDK is stateful per-signer.** Re-init for each user (or use multiple client instances) — don't expect one client to "switch user" cleanly.
- **Permit `name` collisions cause stale data.** Use distinct `name` strings if two users issue permits in the same test.
- **ACP import API varies** — check `cofhe-sdk` source for the current shape (`importPermit`, `acceptPermit`, etc.).
- **Test isolation matters more here** — leftover permits from previous tests can affect outcomes. Reset in `beforeEach`.
