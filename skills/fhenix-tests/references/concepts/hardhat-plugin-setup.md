# Hardhat plugin setup

## What

`@cofhe/hardhat-plugin` provides Hardhat-native integration for CoFHE: encryption tasks, MOCK / TESTNET environment auto-selection, signer helpers, and convenience methods that wrap the SDK for test contexts.

## Install

```
npm install --save-dev @cofhe/hardhat-plugin
```

In `hardhat.config.ts`:

```
import '@cofhe/hardhat-plugin';
import { HardhatUserConfig } from 'hardhat/config';

const config: HardhatUserConfig = {
  networks: {
    hardhat: { /* uses MOCK mode automatically */ },
    arbSepolia: {
      url: process.env.ARB_SEPOLIA_RPC,
      accounts: [process.env.PRIVATE_KEY!],
      /* uses TESTNET mode automatically */
    },
  },
  solidity: '0.8.25',
};
export default config;
```

## Use in tests

```
import { ethers } from 'hardhat';
import { cofhejs_initializeWithHardhatSigner, Encryptable, FheTypes } from '@cofhe/hardhat-plugin';

describe('MyContract', () => {
  before(async () => {
    const [signer] = await ethers.getSigners();
    await cofhejs_initializeWithHardhatSigner(signer);
    // plugin auto-detects hardhat network → MOCK
    // when network is arb-sepolia → TESTNET
  });

  it('encrypts and submits', async () => {
    const myContract = await ethers.deployContract('MyContract');
    const encrypted = await cofhe.encryptInputs([Encryptable.uint64(42n)]).execute();
    await myContract.submit(encrypted[0]);
    // ...
  });
});
```

Exact API may evolve — verify against `references/lookup-recipes.md`.

## MOCK vs TESTNET

| Mode | Triggered by | Behavior |
|---|---|---|
| MOCK | network name `hardhat` or `localhost` | Encryption short-circuits to stand-in handles; decrypt resolves fast; ACL enforced minimally |
| TESTNET | network name matching a known testnet (arb-sepolia, etc.) | Real SDK calls; real threshold network; real latency |

## What the plugin exposes (typical)

- `cofhejs_initializeWithHardhatSigner(signer)` — bind the SDK to a Hardhat signer.
- Helper tasks like `hardhat cofhe:*` — listing varies; run `npx hardhat --help` to enumerate.
- Re-exports of `Encryptable`, `FheTypes`, etc. so test files don't need a separate SDK import.

## Canonical example

- **cofhe-hardhat-starter** — the reference test setup.
  https://github.com/FhenixProtocol/cofhe-hardhat-starter
  → `test/` directory shows the full pattern.

## Gotchas

- **`cofhejs_*` prefix is legacy.** New SDK helpers may be renamed; verify in the installed `.d.ts`.
- **Network name matters.** The plugin keys mode off the network name; misnamed networks can land in the wrong mode.
- **Some tasks require `.env` vars** (private key, RPC URL). Document required env vars in the project README.
- **TESTNET runs are slow.** Real threshold network = real network latency. Don't run TESTNET tests in tight feedback loops.
