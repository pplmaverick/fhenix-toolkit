# Foundry mocks setup

## What

`cofhe-foundry-mocks` provides Solidity-only stand-ins for the off-chain CoFHE components (TaskManager, FheOS, dispatch path). It lets you test contracts that import `FHE.sol` in a pure Foundry environment — no Node.js, no SDK, fast iteration.

## Install

```
forge install FhenixProtocol/cofhe-foundry-mocks
```

In `foundry.toml`, ensure the remappings point to both the mocks and `cofhe-contracts`:

```
remappings = [
  '@fhenixprotocol/cofhe-contracts/=lib/cofhe-contracts/contracts/',
  'cofhe-foundry-mocks/=lib/cofhe-foundry-mocks/src/',
]
```

## Use

```
import { Test } from "forge-std/Test.sol";
import { CofheMockSetup } from "cofhe-foundry-mocks/CofheMockSetup.sol";
import { MyContract } from "../src/MyContract.sol";

contract MyContractTest is Test {
    MyContract c;

    function setUp() public {
        new CofheMockSetup();   // installs mock TaskManager + FheOS
        c = new MyContract();
    }

    function test_branchlessUpdate() public {
        // operate on the contract; encrypted ops run against the mocks
    }
}
```

## What the mocks DO simulate

- `FHE.add` / `FHE.sub` / `FHE.mul` etc. — produce mock ciphertext handles backed by tracked plaintext.
- `FHE.allowThis` / `FHE.allow` — track ACL bits.
- `FHE.decrypt` + `getDecryptResultSafe` — resolve the request immediately (single-tx in mocks).

## What the mocks DO NOT simulate

- **Real gas costs.** Mock op costs are stand-ins; numbers don't match the threshold network.
- **Async decrypt latency.** Mocks resolve `decrypt` synchronously; real chains take seconds.
- **Signature verification on InEuintXX.** Mock inputs accept fabricated signatures; real `asEuintXX` enforces strictly.
- **The threshold network's MPC semantics.** Edge cases at the protocol layer aren't reproduced.

## Canonical example

- **Internal test contracts in this monorepo.**
  `cofhe/cofhe-contracts/tests/contracts/*.sol` (and `cofhe/tests/contracts/*.sol`) — small Solidity contracts exercising each FHE op category via mocks.

## Gotchas

- **Mock-only constructor logic** doesn't run on mainnet. Don't bake assumptions into `setUp` that production contracts can't satisfy.
- **`forge build` may need explicit `--via-ir`** for the larger generated `FHE.sol` — check the Foundry version compatibility.
- **`CofheMockSetup` constructor must be called before any FHE op** in the test — including in test fixtures.
- **Solidity version mismatch.** `FHE.sol` pins specific compiler features; ensure `solc-version` in foundry.toml matches.
