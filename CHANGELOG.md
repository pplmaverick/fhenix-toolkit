# Changelog

All notable changes to `fhenix-toolkit` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Repository scaffolding: README, LICENSE, CHANGELOG, plugin manifest, marketplace manifest, compatibility matrix.
- `link-check` and `lookup-recipe-smoke` CI workflows.
- Design spec at `docs/SPEC.md`.
- `skills/fhenix-review/` — SKILL.md, lookup-recipes, gotcha catalog (30+ items), security checklist, decision-trees, and four concept files (confidentiality-vs-anonymity, pattern-leakage, proof-of-plaintext-input, reveal-labels).
- `agents/fhe-reviewer.md` — companion deep-audit subagent that loads the full catalog and produces structured prioritized reports.

### Changed

- v1 scope reduced from five to four skills; `fhenix-migrate` deferred to v1.5 (spec retained at `docs/SPEC.md` §5.3).

### Added (continued)

- `skills/fhenix-contracts/` — SKILL.md, lookup-recipes, hard-rules, decision-trees, and seven concept files (branchless-update, allow-cascade, encrypted-input, bit-shift-ratio, operator-pattern, randomness-via-entropy, confidential-token-standards).
