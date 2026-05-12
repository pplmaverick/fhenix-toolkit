# Permits — lifecycle, scoping, expiration

## What

A permit is a signed authorization that lets the SDK ask the threshold network for plaintext on behalf of a user. Permits replace the legacy `cofhejs` auto-permit; they are now explicit, scoped, and time-bounded.

## Permit operations

```
// Create a fresh self-permit
const permit = await cofheClient.permits.createSelf({
  issuer: address,
  name: 'myapp',
  expiration: Math.floor(Date.now() / 1000) + 30 * 24 * 3600,  // SECONDS!
});

// Get-or-create (idempotent — most apps use this)
const permit = await cofheClient.permits.getOrCreateSelfPermit({...});

// Read the cached active permit
const permit = cofheClient.permits.getActivePermit();

// Remove a permit
cofheClient.permits.removePermit(permitId);
```

## Permit scoping — name strategy

The `name` field scopes the permit. Two strategies:

**App-wide** (state never cycles):
```
name: 'myapp'
```
Best for: persistent balances, identity, anything the user keeps for the lifetime of the dApp.

**Per-cycle** (state cycles between rounds):
```
name: `myapp-${gameId}`
```
Best for: per-game state, per-auction state. Avoids stale-permit failures when state rotates.

## Permit expiration — Unix seconds, not milliseconds

`Date.now()` returns ms. The SDK expects seconds.

```
expiration: Math.floor(Date.now() / 1000) + 30 * 24 * 3600   // 30 days from now
```

Forgetting the `/1000` gives you an instantly-expired permit and an opaque rejection on the first decrypt.

## `permitVersion` re-render trick

Permits live in the client. Hooks that depend on "is there a valid permit?" need a way to re-run when the permit changes. Pattern:

```
// in a Zustand store
permitVersion: 0,
bumpPermitVersion: () => set(s => ({ permitVersion: s.permitVersion + 1 })),

// in a hook
useEffect(() => {
  void permitVersion;   // referenced for the dep list
  // ... re-read permit state
}, [permitVersion]);
```

Bump `permitVersion` after `createSelf` / `removePermit`.

## Access Control Permits (ACPs) — selective disclosure

The SDK supports scoped permits naming a specific recipient. The recipient holds the permit and can decrypt on the issuer's behalf; no one else can.

```
const acp = await cofheClient.permits.createPermit({
  issuer: holder,
  recipient: verifierAddress,
  name: 'compliance-disclosure',
  expiration: ...,
});
// Holder shares the ACP JSON with the verifier off-chain.
// Verifier uses it to decrypt the holder's value — chain has no record.
```

This is the selective-disclosure pattern: prove a fact to one party without revealing it on-chain.

## Canonical examples

- **Equle — per-game permit naming.**
  https://github.com/FhenixProtocol/miniapp-equle
  → `packages/cofhe-nextjs/src/app/hooks/usePermit.ts`. `getOrCreateSelfPermit({..., name: \`equle${gameId}\`})`.

- **Secret Santa — app-wide permit.**
  https://github.com/FhenixProtocol/encrypted-secret-santa
  → `packages/nextjs/hooks/useSecretSanta.ts`. `getOrCreateSelfPermit({..., name: 'Secret Santa'})`.

- **Selective disclosure — ACP scoping.**
  https://github.com/FhenixProtocol/selective-disclosure-demo
  → ACP creation + JSON share for verifier to decrypt.

## Gotchas

- **No auto-permit.** Calling `decryptForView` without a permit fails. Always `getOrCreateSelfPermit` first.
- **`expiration` is absolute (Unix seconds), not duration.** Use `Math.floor(Date.now() / 1000) + DURATION_SECONDS`.
- **Permits are per-client-instance.** A user reloading the page may lose them. Use SDK persistence (if any) or accept re-creating each visit.
- **ACPs cross trust boundaries.** Treat the JSON as a capability token. Don't post it publicly and expect it to remain useful.
- **`name` is part of the permit's identity** — changing `name` invalidates lookup via `getActivePermit`.
