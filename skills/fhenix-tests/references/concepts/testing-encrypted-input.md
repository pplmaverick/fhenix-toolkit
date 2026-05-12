# Testing encrypted input — InEuintXX fixtures

## What

To test a function accepting `InEuintXX`, the test needs to fabricate (or genuinely encrypt) the input struct before calling the function. Two paths:

1. **Real encryption** via the SDK / Hardhat plugin — exercises the full flow including signature verification.
2. **Mock fabrication** via Foundry mocks — fast, skips signature verification (mocks accept fake sigs).

## Pattern A: real encryption (Hardhat)

```
import { Encryptable, FheTypes } from '@cofhe/hardhat-plugin';

it('accepts a valid encrypted bid', async () => {
  const [signer] = await ethers.getSigners();
  await cofhejs_initializeWithHardhatSigner(signer);

  const [encrypted] = await cofhe
    .encryptInputs([Encryptable.uint64(100n)])
    .execute();

  await myContract.bid({
    ctHash: encrypted.ctHash,
    securityZone: encrypted.securityZone,
    utype: encrypted.utype,
    signature: encrypted.signature,
  });

  // assert state changed as expected
});
```

## Pattern B: mock fabrication (Foundry)

```
import { CofheMockSetup, mockInEuint64 } from "cofhe-foundry-mocks/CofheMockSetup.sol";

function test_acceptsValidBid() public {
    new CofheMockSetup();
    InEuint64 memory inBid = mockInEuint64(100);   // helper from the mock library
    myContract.bid(inBid);
    // assert state changed
}
```

(Helper name varies; check `cofhe-foundry-mocks` source.)

## Testing rejection paths

Forge a sig mismatch / utype mismatch / expired zone:

```
// E.g., wrong utype
InEuint32 memory wrongType = mockInEuint32(100);
vm.expectRevert();
myContract.bid(wrongType);   // function expects InEuint64
```

This is a critical test class — confirms the contract correctly delegates verification to `FHE.asEuintXX`.

## Multi-input fixtures

```
const [encEq, encResult] = await cofhe
  .encryptInputs([
    Encryptable.uint128(0xDEADBEEFn),
    Encryptable.uint16(42n),
  ])
  .execute();

await myContract.guess(encEq, encResult);
```

Order matters; the array of inputs maps 1:1 to the order in `encryptInputs`.

## Gotchas

- **Mock helpers don't enforce signature validity** — your test for "rejects invalid signature" will pass in mocks even if the contract is broken on real chains. Run such tests on testnet.
- **`Encryptable.uintN` takes `bigint`** — use the `n` literal suffix or wrap with `BigInt(...)`.
- **Real-encryption tests are slow** — first call triggers WASM init (~1-3s). Plan around it (e.g., warm up in `before(...)`).
- **The `signature` field is ABI-typed as `bytes`** in Solidity, but TypeScript types it more strictly. Cast `as 0x${string}` if needed.
