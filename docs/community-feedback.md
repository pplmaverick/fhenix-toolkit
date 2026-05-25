# Community feedback process

This document defines how community-reported issues, suggestions, and observations flow back into the skill content (SKILL.md activations, references, gotcha catalog).

Tracking issue: [#6 — Review community feedback report and adjust skills](https://github.com/FhenixProtocol/fhenix-toolkit/issues/6).

## Goals

- Community devs find that their reports produce visible changes within a predictable window.
- Skill drift (stale claims, missing rules, fabricated APIs) is detected and fixed faster than annual releases.
- Maintainers have a clear, low-overhead routine for processing the feedback stream.

## Sources of feedback

- **GitHub issues** opened against this repo. Primary funnel.
- **Discord / Telegram** — community channels where Fhenix devs hang out. Triage Maintainer translates relevant threads into GitHub issues so they enter the funnel.
- **Direct devrel touchpoints** — workshops, dev calls, 1:1 questions. Same translation rule.
- **Drift-detection workflows** (`link-check`, `lookup-recipe-smoke` in `.github/workflows/`) — automated; produces issues directly when they fail.
- **Self-review** of recent PR-review-cycles in dependent repos (`cofhesdk`, `cofhe-contracts`, example apps).

## Review cadence

| Cadence | What | Owner |
|---|---|---|
| **Daily (async)** | Triage incoming GitHub issues — label, prioritise, close obvious dupes. | Triage rotation (DevRel) |
| **Weekly (15 min)** | Skim the open-issue list; close stale ones, escalate hot ones. | Triage rotation |
| **Monthly (60 min)** | Roll-up review: which skills attracted the most issues? What classes of feedback recurred? Decide on substantive adjustments. | DevRel lead + 1 SDK rep + 1 contracts rep |
| **Per major SDK / contracts release** | Run the audit checklist below against the affected skill(s). Open follow-up PRs for any drift. | Whichever rep owns the release |

## Monthly review checklist

For each skill (`fhenix-contracts`, `fhenix-sdk`, `fhenix-review`, `fhenix-tests`):

- [ ] List issues opened/closed since last review. Note recurring themes.
- [ ] Verify each `references/lookup-recipes.md` URL still resolves and returns relevant content.
- [ ] Pick 3 random concept files; spot-check that the linked canonical examples (in real public repos) still exist with the named targets and that the patterns shown still match.
- [ ] Re-check the `references/hard-rules.md` against current `FHE.sol` / `@cofhe/sdk` source. Any rules that contradict current source? Any new rules that should be added (e.g., new gotcha class from a recent fix)?
- [ ] For `fhenix-review` specifically: walk the gotcha catalog. Any entries that no longer reproduce? Any new gotchas observed in recent PR reviews?
- [ ] For `fhenix-tests`: confirm `@cofhe/hardhat-plugin` and `cofhe-mock-contracts` APIs still match what the skill teaches.

Output: a checklist of changes needed → one PR per skill if substantive, OR one bundled PR if minor.

## Per-release audit (triggered by upstream major releases)

When `cofhesdk` or `cofhe-contracts` ships a major:

1. The cross-repo `repository_dispatch` workflow (per `docs/SPEC.md` §9) opens a bump PR in this repo. (Currently this workflow is unimplemented; see #7.)
2. The PR author runs `compatibility.json` against the new version.
3. Maintainer reviews drift, applies fixes, merges.

## Pattern recognition (what to look for)

Across feedback sources, watch for:

- **"Skill said X but Y is real"** — direct contradiction between curated content and live source. Highest priority. Fix in-place.
- **"Skill is silent on Z"** — gap. Decide if Z is in scope (add to references / gotcha catalog) or out of scope (close with reasoning).
- **"Skill activates when it shouldn't" / "skill doesn't activate when it should"** — trigger mistuning. Adjust SKILL.md `description:`.
- **Repeated questions in community channels** — signal that documentation is missing or hard to find. Add a concept file or sharpen activation triggers.
- **Real-world bugs traced back to skill guidance** — Critical. Treat as a security finding; reproduce, fix, document.

## Severity → action mapping

| Severity | Median time to fix | Action |
|---|---|---|
| Critical (active misguidance leading to real bugs) | 24-48h | Open PR same day; merge with abbreviated review |
| High (factual error, but not immediately exploitable) | 1 week | Open issue → PR within the week |
| Medium (gap, awkward phrasing, missing nuance) | Monthly review cycle | Batched in the monthly roll-up PR |
| Low (style, taste, typo) | Best-effort | Welcome PRs; don't prioritise own time |

## Metrics worth tracking (best-effort)

These don't need a fancy dashboard — a quarterly numbers tally in a tracking issue is enough:

- Open issues count, trend over time.
- Time from issue open → first reply.
- Time from issue open → resolution (closed or PR merged).
- Number of "skill said X but Y is real" reports (signal of curated-content quality).
- Number of skill activations reported as helpful vs noise (rough survey at devrel touchpoints).

## When the process itself needs adjustment

Cadence and checklists are starting points. If a routine produces no findings for 3 review cycles in a row, drop or compress it. If a routine consistently overruns its time budget, split it. Document changes here so the process stays self-consistent.

Periodic process retro: quarterly, 15 minutes, two maintainers; output is edits to this file.
