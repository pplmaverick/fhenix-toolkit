# Lookup recipes — fhenix-tests

## Find current Foundry mock APIs

Source of truth: `FhenixProtocol/cofhe-mock-contracts` (the old `cofhe-foundry-mocks` repo is archived). Default branch is `master`.

```
https://github.com/FhenixProtocol/cofhe-mock-contracts
```

Key files:

- `contracts/CoFheTest.sol` — the abstract `Test` extension your test inherits. Look for `createInEuintXX`, `assertHashValue`, `etchFhenixMocks`, `mockStorage`.
- `contracts/MockTaskManager.sol` — mock TaskManager (ACL, ciphertext handle storage, op semantics).
- `contracts/MockZkVerifier.sol` — mock ZK input verification.
- `contracts/MockQueryDecrypter.sol` — mock decryption-query handler.
- `contracts/ACL.sol` — mock ACL.

Fetch raw:

```
WebFetch https://raw.githubusercontent.com/FhenixProtocol/cofhe-mock-contracts/master/contracts/CoFheTest.sol
```

Browse:

```
gh api repos/FhenixProtocol/cofhe-mock-contracts/contents/contracts
```

## Find current Hardhat plugin tasks / API

```
https://github.com/FhenixProtocol/cofhesdk/tree/master/packages/hardhat-plugin
https://www.npmjs.com/package/@cofhe/hardhat-plugin
```

Locally:

```
find node_modules/@cofhe/hardhat-plugin -name '*.d.ts' | head
cat node_modules/@cofhe/hardhat-plugin/package.json
```

For task list:

```
npx hardhat --help
```

Real entrypoints to look up:

- `hre.cofhe.createClientWithBatteries(signer)` — SDK client construction.
- `task:cofhe-mocks:deploy` — Hardhat task that deploys the mock contracts.
- `cofhe` config block in `HardhatUserConfig` (e.g. `logMocks`, `gasWarning`).

## Find canonical test patterns

| Repo | What to look at |
|---|---|
| `cofhe-hardhat-starter` | Complete starter: fixture → `task:cofhe-mocks:deploy` → `createClientWithBatteries` → encrypt → tx → decrypt → assert |
| `poc-sealed-bid-auction/packages/hardhat/test/` | Decrypt-and-settle test pattern (`allowPublic` + `decryptForTx().withoutPermit()` + `verifyDecryptResult`) |
| `miniapp-equle/packages/hardhat/test/` | Multi-input encrypt + verify-on-claim pattern |
| `encrypted-secret-santa/packages/hardhat/test/` | User-contributed entropy + `decryptForView` pattern |
| `cofhe-mock-contracts/test/` | The mock repo's own tests — minimal usage of `CoFheTest` |

Fetch:

```
WebFetch https://raw.githubusercontent.com/FhenixProtocol/cofhe-hardhat-starter/main/test/Counter.test.ts
```

## Find the installed plugin version

```
npm list @cofhe/hardhat-plugin
cat node_modules/@cofhe/hardhat-plugin/package.json | grep '"version"'
```

For the latest published:

```
npm view @cofhe/hardhat-plugin version
npm view @fhenixprotocol/cofhe-mock-contracts version
```

## Reference docs

- https://cofhe-docs.fhenix.zone/get-started/ (local development setup section)
- https://github.com/FhenixProtocol/cofhe-hardhat-starter#readme
- https://github.com/FhenixProtocol/cofhe-mock-contracts#readme (canonical README for `CoFheTest`)
