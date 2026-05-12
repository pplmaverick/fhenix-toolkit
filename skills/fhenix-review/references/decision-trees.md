# Decision trees — fhenix-review

## When to escalate to the deep-audit subagent

```
Is the diff small (< 100 LOC of FHE-related code) and the function-by-function flow obvious?
├── Yes  →  Run review inline. Apply the security checklist directly.
└── No   →  Is this security-sensitive (pre-launch, handling real value, complex composition)?
            ├── Yes  →  Invoke fhe-reviewer subagent for a thorough pass.
            └── No   →  Run inline review; flag explicitly that this wasn't a deep audit.
```

## Severity ratings

| Severity | When |
|---|---|
| **Critical** | Funds at risk, private data leakable to public, or contract becomes uncallable due to ACL mistake |
| **High** | A user can be tricked into revealing private data, or operation fails silently in a way users won't notice |
| **Medium** | Inefficient (wastes gas), brittle (works today, breaks on next SDK bump), or violates a hard rule without an immediate exploit path |
| **Low / Note** | Style, documentation, optimization opportunities, future-tense concerns |

## When a gotcha is "not actually a bug"

```
Could the gotcha be exploited or cause user-visible failure in this codebase?
├── No, it's defensively prevented elsewhere  →  Note it; don't escalate.
├── Maybe, depending on caller behavior        →  High severity.
└── Yes, on the happy path                     →  Critical severity.
```

Even when not exploited, repeated rule violations (e.g. missing `allowThis` in 10 places) signal a maintainer who didn't internalize the model — flag the pattern, not just the line.

## "Is this leaking via side channels?"

```
Does the on-chain footprint differ when the encrypted value differs?
├── Yes — gas usage differs measurably     →  Side-channel leakage (Medium-High)
├── Yes — different events fire             →  Event-fact leakage (Medium-High)
├── Yes — different functions are called    →  Tx-graph leakage (depends on threat model)
└── No — externally indistinguishable       →  Privacy holds at this layer.
```
