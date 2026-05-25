# Reveal labels — categorizing every decrypt

## What

Every place the code decrypts something falls into one of three categories. Mis-labeling produces silent failures, leaked plaintexts, or broken UX. A reviewer should explicitly label each decrypt callsite as one of:

- **UI-view** — purely off-chain display for the user's own eyes. Contract never sees the plaintext.
- **Protocol-reveal-permit** — user proves a private value to the contract via a permit + verify flow.
- **Protocol-reveal-public** — the protocol intends everyone to see the plaintext at this point.

## The labeling test

For each decrypt callsite, ask:

```
Will the contract use the plaintext to compute / decide / settle?
├── Yes  →  PROTOCOL REVEAL
│           ├── Is this a per-user "I proved I know X" check?
│           │   └──→  protocol-reveal-permit
│           │         (FHE.allow(ct, user) on-chain + decryptForTx().withPermit() off-chain
│           │         + FHE.verifyDecryptResult on-chain)
│           │
│           └── Is this a "everyone sees the settle result" reveal?
│              └──→  protocol-reveal-public
│                    (FHE.allowPublic(ct) on-chain + decryptForTx().withoutPermit() off-chain
│                    + FHE.verifyDecryptResult on-chain)
│
└── No  →  UI-VIEW
          (FHE.allow(ct, user) on-chain + decryptForView(ct, type) off-chain
          Plaintext stays client-side, never round-trips)
```

## Common mislabels

| Mislabel | Symptom | Fix |
|---|---|---|
| UI-view used where protocol-reveal needed | Contract's follow-up call fails (no signature passed); user sees the value but can't claim/settle | Switch to `decryptForTx`; pass `signature` to the contract |
| Protocol-reveal-public used where permit-based was right | Value visible to everyone, even though only the user needed to know | Use `FHE.allow(ct, user)` + `decryptForTx().withPermit()` |
| Protocol-reveal-permit used where UI-view was right | Extra on-chain tx for no reason; UX hit | Use `decryptForView` only |
| UI-view used where permit needed | Permit not created → `decryptForView` fails | Call `getOrCreateSelfPermit` first |

## Review hook

While reading SDK callsites, write the inferred label next to each `decryptFor*` call. If a label is ambiguous, that's a finding: the code's intent isn't clear and someone migrating from cofhejs almost certainly classified it wrong.

## Source

The labeling discipline comes from Fhenix's "Decryption in CoFHE, Evolved" post: https://www.fhenix.io/blog/decryption-in-cofhe-evolved
