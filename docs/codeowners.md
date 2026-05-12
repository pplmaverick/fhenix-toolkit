# CODEOWNERS — current state and TODO

The repo ships a CODEOWNERS file at `.github/CODEOWNERS`. As of this writing it is a **placeholder**: every line shows the intended ownership *shape* but the actual `@handle` mentions are commented out.

Tracking issue: [#8 — Add CODEOWNERS](https://github.com/FhenixProtocol/fhenix-toolkit/issues/8).

## Why a placeholder

We don't yet have a clear roster of named owners across DevRel, SDK, contracts, and security. The placeholder makes the intent visible so contributors and reviewers know which area maps to which team — when the handles are filled in, GitHub will start auto-requesting reviews.

## To activate

1. Decide which GitHub users / teams own each area. Create the GitHub teams (`@FhenixProtocol/devrel`, `@FhenixProtocol/contracts-team`, `@FhenixProtocol/sdk-team`, `@FhenixProtocol/security`) in the org if they don't exist.
2. Add the corresponding users to each team.
3. In `.github/CODEOWNERS`, **uncomment** the `@FhenixProtocol/<team>` mentions on each line (remove the leading `#`).
4. In the repo's branch-protection settings on `main`: require pull-request review and tick "Require review from Code Owners."
5. Open a small test PR touching one file per area to confirm the right team(s) get auto-requested.

## Validation

After activating, a contributor opening a PR that touches `skills/fhenix-contracts/SKILL.md` should see `@FhenixProtocol/contracts-team` auto-added as a reviewer. If they don't, check:

- Was the user added to the team? (Team membership is required.)
- Does the team have write access to the repo?
- Is the CODEOWNERS path pattern correct? Test with `https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners#example-of-a-codeowners-file`.

## Why this structure

Skills map to the team that owns the underlying technology:

- `fhenix-contracts` → contracts team (knows FHE.sol surface, on-chain ACL semantics)
- `fhenix-sdk` → SDK team (knows `@cofhe/sdk` API, permits, decrypt modes)
- `fhenix-review` → security + contracts (audit-flavoured; security has veto)
- `agents/fhe-reviewer.md` → security alone (subagent that performs deep audits)
- `fhenix-tests` → SDK + contracts (straddles both surfaces)
- Docs / spec / infra → devrel (lightest review touch)

This isn't perfect — cross-cutting changes will hit multiple owners, and the boundary between "skill content" and "tooling" can be fuzzy. Refine as the team grows.

## When to revisit

- A new skill ships → add a CODEOWNERS line for it.
- A team renames or splits → update the team mention.
- A reviewer-rotation pattern emerges (e.g. one person always covers gotcha-catalog PRs) → add a more granular rule above the team rule.
