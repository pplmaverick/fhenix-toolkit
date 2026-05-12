---
name: fhenix-tests
description: Use when writing or extending tests for confidential CoFHE code — Foundry mocks, Hardhat plugin, encrypted input/output assertions, async-decrypt polling in tests. Activates on .test.ts / .t.sol files importing FHE.sol or @cofhe/sdk, on files under tests/contracts/, or on prompts like "test this confidential contract."
---

# fhenix-tests — test confidential CoFHE contracts and dApps

You activate when the user is writing tests that exercise Fhenix-encrypted code: Foundry `.t.sol` files importing FHE.sol, Hardhat `.test.ts` files using `@cofhe/sdk` or the Hardhat plugin's mock environment, or any file under `tests/contracts/`.

## The first decision: Foundry mocks or Hardhat plugin?

| Choice | Use when |
|---|---|
| **Foundry mocks** (`cofhe-foundry-mocks`) | Pure-Solidity testing; fast iteration; no Node.js dependency; testing branchless logic, `select`/`allowThis` paths, encrypted state transitions. |
| **Hardhat plugin** (`@cofhe/hardhat-plugin`) | End-to-end with the SDK; tests the encrypt→tx→decrypt loop; tests frontend integration; covers the on-chain + off-chain split. |

You can use both in one project — Foundry for unit-level Solidity logic, Hardhat for E2E flows.

Decision tree: `references/decision-trees.md`.

## Hard rules — never violate

1. **Mock gas ≠ prod gas.** Mock environments simulate gas costs that don't match the threshold network's actual costs. Always validate gas-sensitive code on a testnet before mainnet.

2. **Async decrypt requires polling in tests too.** `FHE.decrypt(ct)` followed immediately by `getDecryptResult(ct)` in the same test will revert in real environments. Mocks may "fake" sync — but your tests must still wait/poll for parity.

3. **Seed `FHE.random*` for determinism.** Pass a non-zero seed to make tests reproducible. Seed=0 picks a system seed → flaky tests.

4. **Test the ACL path.** For every encrypted op stored in state, assert that subsequent operations succeed (proving `allowThis` was called correctly). Forgotten `allowThis` is a silent class of bug.

5. **Test both arms of `FHE.select`.** Even when the encrypted condition picks one arm at runtime, both must be valid expressions. Force both branches in tests by varying inputs.

6. **Verify on-chain + off-chain pairing.** E2E tests must exercise the decrypt-flow pairing matrix (`allowPublic` ↔ `withoutPermit`, etc.). A test that uses one without the other is testing a non-existent code path.

Full rule list: `references/hard-rules.md`.

## Default Foundry setup

Add the mocks as a Foundry library:

```
forge install FhenixProtocol/cofhe-foundry-mocks
```

In `foundry.toml`, point `remappings` at the library. In test files:

```
import { CofheMockSetup } from "cofhe-foundry-mocks/CofheMockSetup.sol";

contract MyTest is Test {
    function setUp() public {
        new CofheMockSetup();   // installs the mock TaskManager / FheOS
        // ... deploy your contracts
    }
}
```

Details: `concepts/foundry-mocks-setup.md`.

## Default Hardhat setup

Install the plugin:

```
npm install --save-dev @cofhe/hardhat-plugin
```

In `hardhat.config.ts`:

```
import '@cofhe/hardhat-plugin';
```

The plugin auto-selects `MOCK` mode on the hardhat network and `TESTNET` on arb-sepolia. Tests use the SDK normally; the plugin handles environment-specific keying.

Details: `concepts/hardhat-plugin-setup.md`.

## Async-decrypt in tests

```
// Request decrypt in tx 1
await contract.requestSettlement();
// Wait for result (mocks may resolve fast; real chains take seconds)
await waitForDecryption(contract, ctHash);   // helper that polls
// Settle in tx 2
await contract.finalizeSettlement(...);
```

Pattern: `concepts/testing-async-decrypt.md`.

## Concepts to read on demand

- `foundry-mocks-setup.md` — installing and using `cofhe-foundry-mocks`.
- `hardhat-plugin-setup.md` — installing and using `@cofhe/hardhat-plugin`, MOCK vs TESTNET mode.
- `testing-encrypted-input.md` — building `InEuintXX` test fixtures off-chain.
- `testing-async-decrypt.md` — polling for decrypt results in tests.
- `testing-multi-permit.md` — multiple permits / ACPs / cross-user flows.
- `mock-vs-prod-divergence.md` — what mocks differ on (gas, latency, randomness).

## Looking up

For exact mock APIs, Hardhat plugin tasks, or current setup snippets, see `references/lookup-recipes.md` — prefer live source over recall.
