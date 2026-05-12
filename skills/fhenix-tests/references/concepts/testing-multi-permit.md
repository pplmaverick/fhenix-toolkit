# Testing multi-permit and cross-user flows

## What

Some confidential dApps involve multiple parties holding different permits over the same encrypted state. Tests need to exercise the full permission graph — not just "user A can decrypt their own value."

Patterns:

- Two users each have their own encrypted balance; each can only decrypt their own.
- A **sharing permit** lets a holder authorise a specific recipient to decrypt their value.
- A protocol-public reveal at settle exposes a previously-private value to everyone.

## Pattern: two users, separate permits

```
import { loadFixture } from "@nomicfoundation/hardhat-toolbox/network-helpers";
import hre from "hardhat";
import { Encryptable, FheTypes } from "@cofhe/sdk";
import { expect } from "chai";

async function fixture() {
  await hre.run("task:cofhe-mocks:deploy");
  const [alice, bob] = await hre.ethers.getSigners();

  const aliceClient = await hre.cofhe.createClientWithBatteries(alice);
  const bobClient   = await hre.cofhe.createClientWithBatteries(bob);

  // Permits — verify the current signature via lookup-recipes.
  // Canonical 3-arg form: (publicClient, walletClient, options).
  await aliceClient.permits.getOrCreateSelfPermit(undefined, undefined, {
    issuer: alice.address, name: "myapp",
    expiration: Math.floor(Date.now() / 1000) + 3600,
  });
  await bobClient.permits.getOrCreateSelfPermit(undefined, undefined, {
    issuer: bob.address, name: "myapp",
    expiration: Math.floor(Date.now() / 1000) + 3600,
  });

  // ... deploy contract, connect Alice, deposit encrypted balance ...
  return { alice, bob, aliceClient, bobClient, contract };
}

it("each user decrypts only their own balance", async () => {
  const { alice, aliceClient, bobClient, contract } = await loadFixture(fixture);

  const [encAmount] = await aliceClient
    .encryptInputs([Encryptable.uint64(100n)])
    .execute();
  await contract.connect(alice).deposit(encAmount);

  const aliceBalCt = await contract.balanceOf(alice.address);
  const { decryptedValue: aliceBal } = await aliceClient
    .decryptForView(aliceBalCt, FheTypes.Uint64)
    .execute();
  expect(aliceBal).to.equal(100n);

  // Bob CANNOT decrypt Alice's balance
  await expect(
    bobClient.decryptForView(aliceBalCt, FheTypes.Uint64).execute()
  ).to.be.rejected;
});
```

## Pattern: sharing permit (selective disclosure)

The SDK exposes sharing permits via `PermitUtils.createSharing(...)` and `PermitUtils.importShared(...)` (option types `CreateSharingPermitOptions` / `ImportSharedPermitOptions`). Exact API shape evolves — verify against `references/lookup-recipes.md` before authoring.

```
import { PermitUtils } from "@cofhe/sdk/permits";

// Holder creates a sharing permit naming the verifier:
const sharingPermit = await PermitUtils.createSharing(/* ...options matching CreateSharingPermitOptions... */);

// Off-chain: holder ships the serialized permit JSON to the verifier.
// Verifier imports it into their client:
await PermitUtils.importShared(/* ...options matching ImportSharedPermitOptions... */);

// Verifier can now decrypt the holder's value:
const { decryptedValue } = await verifierClient
  .decryptForView(holderBalCt, FheTypes.Uint64)
  .execute();
expect(decryptedValue).to.equal(expectedBalance);

// A third party (eve) cannot
await expect(
  eveClient.decryptForView(holderBalCt, FheTypes.Uint64).execute()
).to.be.rejected;
```

## Pattern: protocol-public reveal at settle

```
// Contract calls FHE.allowPublic(winnerCt) inside requestSettlement
await contract.requestSettlement();

const winnerCt = await contract.winnerHandle();
// Any client can decrypt without a permit because the handle is public
const { decryptedValue: winner, signature } = await eveClient
  .decryptForTx(winnerCt).withoutPermit().execute();

// Pass through the contract's verifier (FHE.verifyDecryptResult inside)
await contract.finalizeSettlement(winner, signature);
```

## Canonical examples

- **Sealed-bid auction E2E.**
  https://github.com/FhenixProtocol/poc-sealed-bid-auction/blob/main/packages/hardhat/test/ — multi-bidder flow with a single settler.

- **Secret Santa multi-participant.**
  https://github.com/FhenixProtocol/encrypted-secret-santa/blob/main/packages/hardhat/test/ — each participant has a different encrypted target visible only to them.

- **Selective-disclosure demo.**
  https://github.com/FhenixProtocol/selective-disclosure-demo — sharing-permit creation + recipient import flow.

## Gotchas

- **Clients are per-signer.** Use `hre.cofhe.createClientWithBatteries(signer)` once per user, then re-use; don't expect a single client to "switch user" cleanly.
- **`getOrCreateSelfPermit` real signature is 3-arg**: `(publicClient, walletClient, { issuer, name, expiration })`. Single-options-object form (`{ ... }` as the only arg) is wrong.
- **Permit `name` collisions cause stale data.** Use distinct `name` strings if two users issue permits with semantically different purposes.
- **`client.permits.createPermit(...)` does NOT exist.** Real method family is `createSelf`, `createSharing`, `importShared` (via `PermitUtils`).
- **Sharing-permit serialization is a capability token** — treat the JSON like a one-shot capability; do NOT log it in test output or commit it to fixtures.
- **Test isolation matters.** Leftover permits from previous tests can affect outcomes; reset via fixture (`loadFixture`) or `beforeEach`.
