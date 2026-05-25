# Hardhat plugin setup

## What

`@cofhe/hardhat-plugin` provides Hardhat-native integration for CoFHE: a Hardhat Runtime Environment extension (`hre.cofhe`) that owns mock deployment and client construction, automatic injection of the mock contracts into the Hardhat testnet, and convenience methods that wrap the SDK for test contexts.

## Install

```
npm install --save-dev @cofhe/hardhat-plugin
# or: pnpm add -D @cofhe/hardhat-plugin
```

In `hardhat.config.ts`:

```
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "@nomicfoundation/hardhat-ethers";
import "@cofhe/hardhat-plugin";

const config: HardhatUserConfig = {
  cofhe: {
    logMocks: true,    // verbose log of mock FHE ops
    gasWarning: true,  // flag expensive ops
  },
  solidity: {
    version: "0.8.28",
    settings: { evmVersion: "cancun" },
  },
};
export default config;
```

When loaded, the plugin watches Hardhat's `node` / `test` tasks and registers `task:cofhe-mocks:deploy` for deploying the mock contracts.

## Use in tests

```
import { loadFixture } from "@nomicfoundation/hardhat-toolbox/network-helpers";
import hre from "hardhat";
import { Encryptable, FheTypes } from "@cofhe/sdk";
import { expect } from "chai";

describe("Counter", function () {
  async function deployFixture() {
    // Deploy mock CoFHE contracts at fixed addresses
    await hre.run("task:cofhe-mocks:deploy");

    const [owner, alice] = await hre.ethers.getSigners();
    const Counter = await hre.ethers.getContractFactory("Counter");
    const counter = await Counter.connect(owner).deploy();

    // Construct an SDK client bound to this signer
    const client = await hre.cofhe.createClientWithBatteries(owner);

    return { counter, owner, alice, client };
  }

  it("encrypts, submits, decrypts", async () => {
    const { counter, client } = await loadFixture(deployFixture);

    const [enc] = await client
      .encryptInputs([Encryptable.uint64(42n)])
      .execute();
    await counter.submit(enc);

    const ctHash = await counter.value();
    const { decryptedValue } = await client
      .decryptForView(ctHash, FheTypes.Uint64)
      .execute();
    expect(decryptedValue).to.equal(42n);
  });
});
```

The reference setup lives at https://github.com/FhenixProtocol/cofhe-hardhat-starter — clone it for a working end-to-end template. Verify the current API against `references/lookup-recipes.md` before authoring.

## MOCK vs TESTNET

| Mode | Triggered by | Behavior |
|---|---|---|
| MOCK | network `hardhat` / `localhost` + the `task:cofhe-mocks:deploy` task run in the fixture | Encryption short-circuits to stand-in handles; decrypt resolves synchronously inside the block; ACL enforced via mock contracts |
| TESTNET | network matching a known testnet (e.g. arb-sepolia) | Real SDK calls; real threshold network; real latency |

The SDK auto-detects mock addresses at known fixed locations and switches behaviour accordingly.

## What the plugin exposes (typical)

- **`hre.cofhe.createClientWithBatteries(signer)`** — construct an SDK client bound to a Hardhat signer; the canonical entrypoint in current tests.
- **`hre.run("task:cofhe-mocks:deploy")`** — the task that deploys / re-deploys the mock contracts.
- The `cofhe` config block in `HardhatUserConfig` (`logMocks`, `gasWarning`).

Run `npx hardhat --help` after installation to enumerate the plugin's task surface — it evolves.

## Canonical example

- **cofhe-hardhat-starter** — the reference test setup.
  https://github.com/FhenixProtocol/cofhe-hardhat-starter
  → `test/Counter.test.ts` shows the full pattern (fixture, mock-deploy, `createClientWithBatteries`, encrypt → tx → decrypt → assert).

## Gotchas

- **Don't use the legacy `cofhejs_initializeWithHardhatSigner(signer)` form.** It's older API surface; current plugin uses `hre.cofhe.createClientWithBatteries(signer)`.
- **Run `task:cofhe-mocks:deploy` in every test fixture**, not just once at file scope — fixtures may run on a fresh fork.
- **TESTNET runs are slow.** Real threshold network = real latency. Don't run TESTNET tests in tight feedback loops.
- **Network-name sensitivity.** The plugin keys mode off the network name and the existence of mock contracts at fixed addresses; misnamed networks land in the wrong mode silently.
- **Use your project's package manager.** Fhenix repos lean on pnpm but the plugin works with npm/yarn/pnpm all the same.
