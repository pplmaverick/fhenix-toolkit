# Testing encrypted input — InEuintXX fixtures

## What

To test a function accepting `InEuintXX`, the test needs to fabricate the input struct before calling the function. Two paths:

1. **Real encryption** via the SDK + Hardhat plugin — exercises the full flow including signature verification by the mocks (in MOCK mode) or the real ZkVerifier (TESTNET).
2. **Mock fabrication** via Foundry mocks — fast, on-chain only. `CoFheTest` exposes `createInEuintXX` helpers.

## Pattern A: real encryption (Hardhat)

```
import { loadFixture } from "@nomicfoundation/hardhat-toolbox/network-helpers";
import hre from "hardhat";
import { Encryptable } from "@cofhe/sdk";
import { expect } from "chai";

describe("Bidder", () => {
  async function fixture() {
    await hre.run("task:cofhe-mocks:deploy");
    const [signer] = await hre.ethers.getSigners();
    const client = await hre.cofhe.createClientWithBatteries(signer);
    const myContract = await hre.ethers.deployContract("MyContract");
    return { client, myContract };
  }

  it("accepts a valid encrypted bid", async () => {
    const { client, myContract } = await loadFixture(fixture);

    const [encrypted] = await client
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
});
```

## Pattern B: mock fabrication (Foundry)

`CoFheTest` exposes typed `createInEuintXX(value, securityZone, sender)` helpers (and `createInEaddress` / `createInEbool`). Inherit `CoFheTest` so they're in scope:

```
import { Test } from "forge-std/Test.sol";
import { CoFheTest } from "@fhenixprotocol/cofhe-mock-contracts/CoFheTest.sol";

contract MyContractTest is Test, CoFheTest {
    function setUp() public {
        etchFhenixMocks();
        // deploy MyContract...
    }

    function test_acceptsValidBid() public {
        InEuint64 memory inBid = createInEuint64(100, 0, address(this));
        myContract.bid(inBid);
        // assert via CoFheTest.assertHashValue(...) on the resulting stored handle
    }
}
```

Look up the exact `createInEuintXX` signature in `cofhe-mock-contracts/contracts/CoFheTest.sol`.

## Testing rejection paths

Forge a sig mismatch / utype mismatch / wrong sender:

```
// E.g., wrong utype — function expects InEuint64, we pass InEuint32
InEuint32 memory wrongType = createInEuint32(100, 0, address(this));
vm.expectRevert();
myContract.bid(wrongType);
```

This is a critical test class — confirms the contract correctly delegates verification to `FHE.asEuintXX`.

## Multi-input fixtures

```
const [encEq, encResult] = await client
  .encryptInputs([
    Encryptable.uint128(0xDEADBEEFn),
    Encryptable.uint16(42n),
  ])
  .execute();

await myContract.guess(encEq, encResult);
```

Order matters; the array of inputs maps 1:1 to the order in `encryptInputs`.

## Gotchas

- **Mock helpers don't enforce ZkVerifier-grade signature validity** — the mocks accept `createInEuintXX`-generated structs. Production rejects malformed inputs strictly; "rejects invalid signature" tests are only meaningful on testnet.
- **`Encryptable.uintN` accepts `string | bigint`** — wrap small numbers with `BigInt(...)` or use the `n` literal suffix.
- **Real-encryption tests trigger WASM init on the first call** (~1–3s). Plan around it (e.g., warm up in `before(...)` / fixture).
- **The `signature` field is ABI-typed as `bytes`** in Solidity, but TypeScript types it more strictly. Cast `as \`0x${string}\`` if the call site complains.
- **`createInEuintXX(...)` takes a `sender`** as the third arg — this is whose perspective the input is "from" for ACL purposes. Use `address(this)` from inside the test contract, or pass a specific signer's address when simulating different users.
