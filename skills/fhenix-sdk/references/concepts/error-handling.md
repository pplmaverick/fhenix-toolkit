# Error handling — CofheError

## What

`@cofhe/sdk` throws typed `CofheError` instances with discriminated codes. No more `Result<T>` tuple-pattern from legacy `cofhejs`.

## The shape

```
import { CofheError } from '@cofhe/sdk';

try {
  const result = await cofheClient.encryptInputs([...]).execute();
} catch (err) {
  if (err instanceof CofheError) {
    switch (err.code) {
      case 'KEYS_NOT_INITIALIZED': /* prompt re-init */ break;
      case 'PERMIT_REQUIRED':       /* call getOrCreateSelfPermit */ break;
      case 'ACL_DENIED':            /* on-chain allow* missing */ break;
      case 'NETWORK_ERROR':         /* retry */ break;
      default:                       throw err;
    }
  } else throw err;
}
```

The exact code list evolves — look it up via `references/lookup-recipes.md`. **Don't hard-code assumptions** about codes; check the typed surface.

## Recoverable vs fatal — rough taxonomy

| Type | Examples | Strategy |
|---|---|---|
| User action | `PERMIT_REQUIRED`, `WALLET_NOT_CONNECTED` | Trigger UI (create permit, reconnect wallet) |
| Network transient | `NETWORK_ERROR`, `TIMEOUT` | Retry with backoff |
| Configuration | `CHAIN_NOT_SUPPORTED`, `KEYS_NOT_INITIALIZED` | Probably a code bug — surface clearly |
| ACL | `ACL_DENIED` | Contract didn't `allow*` correctly — check the pairing matrix in `concepts/decrypt-view-vs-tx.md` |

## Gotchas

- **Errors are NOT Result-wrapped.** Use `try`/`catch`. Don't write `if (result.error)` — the API doesn't have that shape.
- **`err.message` is human-readable but not stable** for branching. Branch on `err.code`.
- **WASM/init errors can surface as generic `Error`** before CofheError is in scope. Always have a fallback `else throw err` branch.
- **Async errors propagate through promise chains** — wrap the outer `await` (the `execute()` call), not the builder chain.
- **Don't swallow errors silently.** Show the user something — even just a toast — when an SDK call fails. Silent failures lead to bug reports like "the button doesn't do anything."
