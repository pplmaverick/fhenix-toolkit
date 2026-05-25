# Changelog

All notable changes to `fhenix-toolkit` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- `assets/fhenix-logo.svg` — official Fhenix logo for README header.
- `CLAUDE.md` at repo root — primer Claude reads when the plugin is loaded; documents activation map, layering, and hard rules.
- `docs/architecture.md` — runtime model, skill anatomy, lookup-driven philosophy, file map.
- `docs/known-flaws.md` — honest coverage-gap catalog grouped by skill + infra.
- `docs/release-process.md` — version bump rules, manual compatibility-bump flow, pre-1.0 caveats.
- Marketing-grade README with three-stat headline, problem→solution table, status, and `@FhenixProtocol` install form.
- `marketplace.json` metadata — `keywords`, `category`, full author block, per-plugin `homepage` / `repository` / `license`.

### Changed

- **Repo layout:** plugin moved to `plugins/fhenix-toolkit/` subdirectory (cc10x-style monorepo pattern). Marketplace `source` is now `git-subdir` pointing at the new path. Frees up the root for a future second plugin without restructuring.
- CI workflows (`link-check.yml`, `lookup-recipe-smoke.yml`) — `paths:` filters and the smoke loop updated to match the new subdir layout.

### Added (original — pre-restructure)

- Repository scaffolding: README, LICENSE, CHANGELOG, plugin manifest, marketplace manifest, compatibility matrix.
- `link-check` and `lookup-recipe-smoke` CI workflows.
- Design spec at `docs/SPEC.md`.
- `skills/fhenix-tests/` — SKILL.md, lookup-recipes, hard-rules, decision-trees, and six concept files (foundry-mocks-setup, hardhat-plugin-setup, testing-encrypted-input, testing-decrypt-flows, testing-multi-permit, mock-vs-prod-divergence).
- `skills/fhenix-review/` — SKILL.md, lookup-recipes, gotcha catalog (30+ items), security checklist, decision-trees, and four concept files (confidentiality-vs-anonymity, pattern-leakage, proof-of-plaintext-input, reveal-labels).
- `agents/fhe-reviewer.md` — companion deep-audit subagent that loads the full catalog and produces structured prioritized reports.
- `skills/fhenix-sdk/` — SKILL.md, lookup-recipes, hard-rules, decision-trees, and six concept files (init-singleton, encrypt-input, decrypt-view-vs-tx, permits, error-handling, hooks-pattern).

### Changed

- v1 scope reduced from five to four skills; `fhenix-migrate` deferred to v1.5 (spec retained at `docs/SPEC.md` §5.3).

### Added (continued)

- `skills/fhenix-contracts/` — SKILL.md, lookup-recipes, hard-rules, decision-trees, and seven concept files (branchless-update, allow-cascade, encrypted-input, bit-shift-ratio, operator-pattern, randomness-via-entropy, confidential-token-standards).
