# Lookup recipes — fhenix-tests

## Find current Foundry mock APIs

Source of truth:

```
https://github.com/FhenixProtocol/cofhe-foundry-mocks
```

Look at:
- `src/CofheMockSetup.sol` — the setup helper
- `src/MockTaskManager.sol` — the mock TaskManager
- `src/MockFheOS.sol` — the mock fheOS

Fetch raw:

```
WebFetch https://raw.githubusercontent.com/FhenixProtocol/cofhe-foundry-mocks/main/src/CofheMockSetup.sol
```

Browse via:

```
gh api repos/FhenixProtocol/cofhe-foundry-mocks/contents/src
```

## Find current Hardhat plugin tasks / API

```
https://github.com/FhenixProtocol/cofhe-hardhat-starter
https://www.npmjs.com/package/@cofhe/hardhat-plugin
```

Locally:

```
find node_modules/@cofhe/hardhat-plugin -name '*.ts' -o -name '*.d.ts' | head
cat node_modules/@cofhe/hardhat-plugin/package.json
```

For task list:

```
npx hardhat --help
```

## Find canonical test patterns

| Repo | What to look at |
|---|---|
| `cofhe-hardhat-starter` | Complete starter: deploy → encrypt input → tx → decrypt → assert |
| `poc-sealed-bid-auction/packages/hardhat/test/` | Async-decrypt-and-settle test pattern |
| `miniapp-equle/packages/hardhat/test/` | Multi-input encrypt + verify-on-claim pattern |
| `encrypted-secret-santa/packages/hardhat/test/` | User-contributed entropy + `decryptForView` pattern |
| This repo (cofhe) at `tests/contracts/` | Internal test contracts — encrypted I/O, public-decrypt, deep nesting |

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
```

## Reference docs

- https://cofhe-docs.fhenix.zone/get-started/ (local development setup section)
- https://github.com/FhenixProtocol/cofhe-hardhat-starter#readme
- https://github.com/FhenixProtocol/cofhe-foundry-mocks#readme
