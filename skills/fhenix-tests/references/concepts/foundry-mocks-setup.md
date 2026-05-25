# Foundry mocks setup

## What

`cofhe-mock-contracts` (formerly `cofhe-foundry-mocks`, now archived) provides Solidity-only stand-ins for the off-chain CoFHE components (TaskManager, ACL, ZkVerifier, QueryDecrypter). It lets you test contracts that import `FHE.sol` in a pure Foundry environment — no Node.js, no SDK, fast iteration.

## Install

```
forge install fhenixprotocol/cofhe-mock-contracts
```

(npm: `@fhenixprotocol/cofhe-mock-contracts` — used by `@cofhe/hardhat-plugin`.)

In `foundry.toml`, point `remappings` at both the mocks and the real `cofhe-contracts` library:

```
remappings = [
  '@fhenixprotocol/cofhe-contracts/=lib/cofhe-contracts/contracts/',
  '@fhenixprotocol/cofhe-mock-contracts/=lib/cofhe-mock-contracts/contracts/',
]
```

## Use

Tests **inherit** from `CoFheTest` (an abstract `Test` extension) and call `etchFhenixMocks()` once in `setUp` to deploy the mock TaskManager / ACL / ZkVerifier at the well-known addresses CoFHE expects.

```
import { Test } from "forge-std/Test.sol";
import { CoFheTest } from "@fhenixprotocol/cofhe-mock-contracts/CoFheTest.sol";
import { MyContract } from "../src/MyContract.sol";

contract MyContractTest is Test, CoFheTest {
    MyContract c;

    function setUp() public {
        etchFhenixMocks();   // installs mock TaskManager / ACL at fixed addresses
        c = new MyContract();
    }

    function test_branchlessUpdate() public {
        // operate on the contract; encrypted ops run against the mocks
    }
}
```

## Test helpers exposed by CoFheTest

- **`createInEbool(bool, securityZone, sender)`** / **`createInEuint8/16/32/64/128/256(value, securityZone, sender)`** / **`createInEaddress(address, securityZone, sender)`** — fabricate `InEbool` / `InEuint*` / `InEaddress` calldata structs ready to pass into your contract. The `securityZone` and `sender` args default to `0` and the test contract's address in the simpler overloads.
- **`assertHashValue(eXXX handle, primitive expected)`** (overloads per encrypted type) — read the mock's plaintext-backed storage for a ciphertext handle and assert equality.
- **`mockStorage(uint256 ctHash)` / `inMockStorage(uint256 ctHash)`** — direct plaintext lookup / existence check.
- **`setLog(bool)`** — toggle on-chain mock op logging via `hardhat/console.sol`.

Always look up exact signatures against the live source — see `references/lookup-recipes.md`.

## What the mocks DO simulate

- `FHE.add` / `FHE.sub` / `FHE.mul` etc. — produce mock ciphertext handles backed by tracked plaintext.
- `FHE.allowThis` / `FHE.allow` / `FHE.allowPublic` — track ACL bits in the mock ACL contract.
- The `verifyDecryptResult` / `publishDecryptResult` flow used by the SDK-mediated decrypt path.
- Encrypted-input verification (`FHE.asEuintXX(in)`) — mock signatures from `createInEuintXX` are accepted.

## What the mocks DO NOT simulate

- **Real gas costs.** Mock op costs are stand-ins; numbers don't match the threshold network.
- **Async decrypt latency.** Mocks resolve decrypt requests synchronously within the same block (the README notes a small random "async duration" but it's still in-test-time, not real network seconds).
- **Real signature verification on `InEuintXX`** — fake-sig rejection tests aren't meaningful here; only testnet enforces the full ZkVerifier path.
- **Threshold network MPC semantics.** Edge cases at the protocol layer aren't reproduced.

## Canonical example

- **cofhe-mock-contracts README** has the full canonical test setup with `CoFheTest`.
  https://github.com/FhenixProtocol/cofhe-mock-contracts
- **Hardhat starter integrates these mocks** via `@cofhe/hardhat-plugin`:
  https://github.com/FhenixProtocol/cofhe-hardhat-starter

## Gotchas

- **Don't `new CoFheTest()`** — it's an abstract contract that your test inherits and `etchFhenixMocks()` deploys the actual mocks at fixed addresses.
- **Call `etchFhenixMocks()` first** in `setUp`, before deploying any contracts that import `FHE.sol`. Otherwise the FHE library targets uninitialised mock storage.
- **`forge build` may need `--via-ir`** for the larger generated `FHE.sol` — check the Foundry version compatibility.
- **Solidity version mismatch.** `FHE.sol` pins specific compiler features; ensure `solc-version` in foundry.toml matches.
- **The old `cofhe-foundry-mocks` repo is archived** — any tutorial pointing there is stale; migrate to `cofhe-mock-contracts`.
