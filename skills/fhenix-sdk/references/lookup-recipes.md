# Lookup recipes — fhenix-sdk

This file teaches Claude how to find current information about `@cofhe/sdk`. **Prefer the user's installed types first** — they match the pinned version exactly.

## Find @cofhe/sdk method signatures

### Option A: User has @cofhe/sdk installed (preferred)

```
find node_modules/@cofhe/sdk -name '*.d.ts' | head
```

Read `node_modules/@cofhe/sdk/dist/index.d.ts` (or the equivalent main entry) for top-level exports. Subpath modules live alongside:

```
node_modules/@cofhe/sdk/dist/permits/
node_modules/@cofhe/sdk/dist/web/
node_modules/@cofhe/sdk/dist/node/
node_modules/@cofhe/sdk/dist/adapters/
```

### Option B: Fetch from the public repo

```
WebFetch https://raw.githubusercontent.com/FhenixProtocol/cofhesdk/main/packages/sdk/src/index.ts
```

The repo is a monorepo with subpackages under `packages/`. Browse via:

```
gh api repos/FhenixProtocol/cofhesdk/contents/packages -H "Accept: application/vnd.github+json"
```

## Find CofheError codes

```
grep -rn "CofheError" node_modules/@cofhe/sdk/dist/
```

The error class is typed with discriminated codes. Read the union to enumerate.

Remote alternative:

```
gh api repos/FhenixProtocol/cofhesdk/contents/packages/sdk/src/errors.ts -H "Accept: application/vnd.github.raw"
```

## Find the installed @cofhe/sdk version

```
npm list @cofhe/sdk
# or
cat node_modules/@cofhe/sdk/package.json | grep '"version"'
```

For the latest published version:

```
npm view @cofhe/sdk version
```

## Find Encryptable types

```
grep -rn "Encryptable" node_modules/@cofhe/sdk/dist/   # local
# or via grep on the fetched index.ts
```

## Find example usage in real apps

| Repo | What it shows |
|---|---|
| [poc-sealed-bid-auction](https://github.com/FhenixProtocol/poc-sealed-bid-auction) | `decryptForTx().withoutPermit()` after `FHE.allowPublic`; full settle loop |
| [miniapp-equle](https://github.com/FhenixProtocol/miniapp-equle) | Per-game permit naming; `decryptForTx` for claim-victory; clean hook decomposition |
| [encrypted-secret-santa](https://github.com/FhenixProtocol/encrypted-secret-santa) | Both `decryptForView` AND `decryptForTx().withPermit()` in one app |
| [poc-shielded-stablecoin](https://github.com/FhenixProtocol/poc-shielded-stablecoin) | SSR-safe Proxy singleton; minimal hooks |
| [selective-disclosure-demo](https://github.com/FhenixProtocol/selective-disclosure-demo) | ACP (Access Control Permit) scoping for selective disclosure |

Use `WebFetch` for raw file pulls; grep by hook name (`useCofhe`, `usePermit`, etc).

## Find canonical hook patterns

```
https://raw.githubusercontent.com/FhenixProtocol/poc-shielded-stablecoin/main/packages/nextjs/hooks/useCofhe.ts
https://raw.githubusercontent.com/FhenixProtocol/poc-shielded-stablecoin/main/packages/nextjs/hooks/usePermit.ts
https://raw.githubusercontent.com/FhenixProtocol/miniapp-equle/main/packages/cofhe-nextjs/src/app/hooks/useCofhe.ts
```

## Docs

- https://cofhe-docs.fhenix.zone/client-sdk/ (conceptual)
- https://fhenix.mintlify.app/client-sdk/ (more current API references)
- https://www.fhenix.io/blog/decryption-in-cofhe-evolved (authoritative narrative for the view vs tx split)

For API specifics, prefer the SDK source over the docs.
