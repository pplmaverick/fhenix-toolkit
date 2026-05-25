# CI workflows

This repo ships two CI workflows under `.github/workflows/`:

## `link-check.yml`

Runs Lychee against every markdown file. Catches dead links to public Fhenix repos, docs pages, blog posts, and external references.

**Triggers:**
- Daily cron at 02:00 UTC
- Manual dispatch (`workflow_dispatch`)
- Pull requests touching `skills/**`, `docs/**`, `agents/**`, `README.md`, `.lycheeignore`, or the workflow file itself

**Tuning:**
- Concurrency capped at 4 to avoid hitting GitHub's rate limit
- 20-second timeout per URL
- 2-second retry wait on transient failures

**False-positive suppression:** `.lycheeignore` at repo root holds the allowlist of URLs to skip. Add entries one per line; lychee accepts plain strings or anchored regex.

**Auth:** uses `secrets.GITHUB_TOKEN` for `api.github.com` and `raw.githubusercontent.com` requests against the FhenixProtocol org. No additional secrets required.

## `lookup-recipe-smoke.yml`

Verifies that every URL referenced from `skills/*/references/lookup-recipes.md` still resolves. The intent is "if Claude follows a recipe, the recipe's URLs actually work today."

**Triggers:**
- Daily cron at 03:00 UTC
- Manual dispatch
- Pull requests touching any `lookup-recipes.md` or the workflow file

**What it does:**
1. Four hand-picked smoke URLs (FHE.sol, cofhesdk core, mock-contracts CoFheTest, docs site) — fast failure on the most-load-bearing recipes.
2. Programmatic extraction of all `https?://(raw.githubusercontent|api.github|cofhe-docs.fhenix|github.com|fhenix.io|npmjs.com|fhenix.mintlify).*` URLs from every skill's `lookup-recipes.md`. Each is HEAD-checked with a 20-second timeout. Failures are listed and the job exits non-zero.

**Deduplication:** URLs are deduped within a run; the same URL referenced from multiple skills is only checked once.

## How to validate manually

```
# From a clone or the GitHub UI: Actions → workflow → "Run workflow"
gh workflow run link-check.yml
gh workflow run lookup-recipe-smoke.yml

# Tail the latest run
gh run watch
```

## Adding new workflows

If you add a workflow:

1. Place it in `.github/workflows/<name>.yml`.
2. Document it here.
3. Add a `paths:` filter on pull-request triggers so the workflow only fires when relevant files change.
4. Use `secrets.GITHUB_TOKEN` for authenticated GitHub API calls; avoid adding new secrets.

## Future work

- Cross-repo `repository_dispatch` from `cofhesdk` and `cofhe-contracts` on major releases — opens an auto-PR in this repo bumping `compatibility.json`. Tracked in `docs/SPEC.md` §9; deferred until v1.0 release.
- If pattern snippets ever land in `references/patterns/`, add a `pattern-compile` workflow (Foundry + tsc). Currently the design philosophy is to link to canonical examples instead of vendoring patterns; this workflow is therefore unneeded.
