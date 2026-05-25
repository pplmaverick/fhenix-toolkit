# Decision trees — fhenix-tests

## Foundry mocks or Hardhat plugin?

```
What layer are you testing?
├── Solidity logic only (branchless updates, allow* paths, encrypted state)
│   →  Foundry mocks (fast, no Node.js, clean assertions)
│
├── Solidity + SDK (encrypt → tx → decrypt → display)
│   →  Hardhat plugin in MOCK mode
│
├── Solidity + SDK against a real testnet (integration)
│   →  Hardhat plugin in TESTNET mode (arb-sepolia, etc.)
│
└── Both unit + E2E
    →  Use both in one repo:
       - Foundry for fast unit tests
       - Hardhat for E2E flows
```

## Mock vs testnet — when to escalate?

```
Are you testing gas behavior?
├── Yes  →  Testnet (mocks don't model real gas accurately)
└── No   →  Mocks first (faster iteration, then promote to testnet)

Are you testing async decrypt latency?
├── Yes  →  Testnet (mocks resolve fast; latency only realistic on testnet)
└── No   →  Mocks fine

Are you testing the threshold network's permit/ACL enforcement?
├── Yes  →  Testnet (mocks may simplify these checks)
└── No   →  Mocks fine
```

## Test fixture for InEuintXX inputs

```
Do you have the SDK installed and a wallet client available?
├── Yes  →  Use the SDK directly:
│           client.encryptInputs([Encryptable.uintN(value)]).execute()
│
└── No   →  Use the test framework's helper:
            - Foundry mocks ship helpers to fabricate fake InEuintXX
            - Hardhat plugin exposes mock-only encryption taskhelpers
```

## When a test passes locally but fails on CI

Common causes for FHE-test flakiness:

```
- Non-deterministic randomness (FHE.random with seed=0)            → seed deterministically
- Async decrypt timing (poll loop too short)                         → increase timeout, or use a fixed mock-tick
- Test isolation (shared state between tests)                        → re-deploy in beforeEach / setUp
- SDK version drift between local and CI                              → pin in package.json, use npm ci
- WASM not initialized in CI                                         → ensure the test runner triggers .execute() at least once before timing-sensitive assertions
```
