# IntentSpec: jup

**Last updated:** 2026-05-31 (RATIFIED by owner 2026-05-31)
**Lifecycle stage:** Build-Ship (released to PyPI; active iteration; current branch `fix/windows-fcntl-compat`)
**Priority:** Medium (publishable open-source tool; complements anima_shell)
**Ceremony tier:** Full

---

## Objective

`jup` is a **small command-line tool for installing and syncing agent skills across the local skill directories used by supported AI assistants** (Gemini, Copilot, Claude, Codex). The name evokes the *Matrix* Jump Program -- the foundational training environment for moving between contexts. The tool installs skills from GitHub repositories (or local sources), keeps them organized via a lockfile, and copies or links them into the per-harness directories. **Scope discipline:** jup is **lightweight, multi-harness, local-first, and intentionally smaller than `anima_shell`** -- anima manages the full identity/memory/persona substrate; jup manages skills specifically.

---

## Desired Outcomes

1. **Multi-harness fidelity.** Maximize the share of supported harnesses (Gemini, Copilot, Claude, Codex) for which `jup sync` produces a correct, reproducible skill set in the expected per-harness directory -- subject to never modifying harness-side state outside the documented per-harness skill directories. (Anchor: harness-agnostic skill distribution; the tool is the *bridge*, not a harness wrapper.)

2. **Lockfile-driven reproducibility.** Maximize the determinism of `jup install` / `jup sync` outcomes from the lockfile -- subject to lockfile changes being explicit and reviewable. The lockfile is the source of truth for what skills are installed; any `sync` operation reads from it. (Anchor: package-manager lockfile semantics -- npm/cargo/uv-style; reproducibility-by-default.)

3. **Cross-platform reliability.** Maximize correct behavior on Linux, macOS, and Windows -- subject to platform-specific idioms not leaking into the cross-platform interface. (Anchor: Python's `portalocker` over `fcntl` for cross-platform locking; per the current `fix/windows-fcntl-compat` branch.)

4. **Code-quality discipline.** Maximize codebase health (small clean commits, conventional commit messages, type-checked, lint-clean, tests pass) -- subject to the AGENTS.md mandates (`just qa` before every commit; no `git add .`; no force pushes without explicit instruction; patch-over-minor version bumps). (Anchor: the AGENTS.md "Mandates & Core Workflows" section -- this IS the project's quality contract.)

5. **Self-documenting CLI.** Maximize the share of behavior discoverable through `jup --help` / `jup <subcommand> --help` and the README's Quick Start path -- subject to no documentation duplication across CHANGELOG / README / AGENTS / docs/.

---

## Health Metrics

- **Harness coverage** (O1 vs O2): every supported harness has an integration test exercising the sync path. *Measurement:* `tests/` includes per-harness coverage; `DEFAULT_HARNESSES` registry matches tested harnesses.
- **Lockfile integrity** (O2 vs O1): lockfile state matches the on-disk skill set after `sync`. *Measurement:* a `jup verify` (or equivalent) check returns clean; lockfile schema-version field is monotonic.
- **Cross-platform CI** (O3 vs O1): CI runs on Linux + macOS + Windows. *Measurement:* `.github/workflows/cd.yml` exercises all three; failures surface before release.
- **`just qa` discipline** (O4 vs all): every commit follows the `just qa` (lint + type-check + tests) precondition. *Measurement:* CI gates; pre-commit hook present.
- **CLI discoverability** (O5 vs O4): every public subcommand has `--help` text + a doc snippet; the README Quick Start path works from scratch. *Measurement:* `docs/reference.md` enumerates all subcommands; image-asset captures of help output stay current (`docs/images/help.svg` etc.).

---

## Constraints

### Hard (non-negotiable, lifted from AGENTS.md "Mandates")

- **C1. Conventional Commits.** All commit messages follow the Conventional Commits spec (`feat:`, `fix:`, `chore:`, etc.). Use `cz commit` when possible. *Prevents:* unstructured commit history that breaks automated changelog + version-bump logic.
- **C2. Small, clean commits -- one concern per commit.** Each commit addresses exactly one concern. Git history quality is a priority. *Prevents:* mixed commits that are hard to bisect, revert, or review.
- **C3. Verification before action (`just qa`).** Run `just qa` (lint + type-check + tests) before each commit. *Prevents:* regressions reaching main.
- **C4. Precise git staging.** Never use `git add .` or `git add -A`. Manually add specific files / hunks per commit. *Prevents:* accidental inclusion of secrets / scratch / unrelated changes.
- **C5. Confirm before pushing.** Always seek user confirmation before pushing to remote branches. *Prevents:* premature publication.
- **C6. No force pushes.** Force pushing is prohibited unless explicitly requested by the user. *Prevents:* lost-work classes of failure.
- **C7. Patch over minor.** Prefer patch-version bumps (`0.x.y+1`) over minor bumps (`0.x+1.0`) when the change allows it. *Prevents:* premature minor inflation; preserves semver meaning.

### Steering (overridable with justification)

- **S1. Subagents for independent steps.** ALWAYS use subagents for independent steps (per AGENTS.md). *Override:* trivially-coupled steps that cost more in coordination than they save.
- **S2. Adversarial testing for significant changes.** For every significant feature or fix, launch 3 parallel adversarial subagents to find edge cases. *Override:* trivial changes (typo fixes, docstring polish) skip this.
- **S3. Documentation updates with every change.** Update AGENTS.md, docs/, and skills/<skill>/SKILL.md if the change affects them. *Override:* internal-only refactors with no surface impact.
- **S4. ASCII typography + no emojis in code-tracked artifacts.** README + AGENTS use emojis as flair (existing convention); code, tests, and config stay emoji-free.

---

## Decision Authority

| Decision | Level | Notes |
|---|---|---|
| Implement a fix or feature within approved scope | Autonomous | Pass `just qa` + new tests + conventional commit (C1-C3) |
| Add a new harness adapter | Collaborative | Scope decision (new entry in `DEFAULT_HARNESSES`) |
| Bump version (patch) | Autonomous | C7 default; release notes via CHANGELOG |
| Bump version (minor or major) | Collaborative | Owner decides on semver intent |
| Publish to PyPI | Human-only | C5; via `cd.yml` triggered after owner OK |
| Push to remote (any branch) | Collaborative | C5 |
| Force-push or rewrite history | Prohibited | C6 |
| Fix stale docs / cross-references / version pointers | Autonomous | Currency; non-semantic |
| Modify CI workflows (`.github/workflows/`) | Collaborative | Cross-cutting; affects release process |

---

## Stop Rules

- **Complete when** (per release): all supported harnesses pass per-harness tests; lockfile integrity verified on test fixtures; cross-platform CI green; CHANGELOG entry exists; docs current.
- **Escalate if:** a supported harness changes its skill-directory contract; lockfile schema needs migration; a force-push is requested; a release is requested without `just qa` clean; an adversarial-subagent pass surfaces a class of bugs unaddressed by the current test set.
- **Abandon/pivot if:** the multi-harness ecosystem consolidates such that a single skill spec emerges (the tool's value drops); maintainer bandwidth no longer supports cross-platform CI.
- **Review cadence:** per-PR review; per-release pre-publication gate (the C5 confirm-before-push pattern).

---

## Strategic Context

- **Depends on:** Python 3.12+ (target); `typer`, `pydantic` v2, `rich`, `uv`; `portalocker` for cross-platform locking; per-harness skill-directory conventions (Gemini, Copilot, Claude, Codex).
- **Enables:** any user who wants to install/sync agent skills across multiple AI assistants from a single tool; complements `anima_shell` (which handles full identity/memory; jup is skill-specific + multi-harness-broader).
- **Boundary vs anima_shell:** anima is the operator's personal harness-substrate (identity, memory, skills, hooks, security) -- single-operator, opinionated, integrated. jup is a lighter, publishable, multi-harness skill-only tool intended for general use. Both can coexist; jup may be a sub-component anima leans on for the skill-sync arm in the future.
- **Priority:** Medium; publishable open-source tool with active iteration.
- **Current -> Next stage:** land `fix/windows-fcntl-compat` (cross-platform locking via `portalocker`); maintain cross-harness fidelity as the per-harness ecosystem evolves.

---

## Change Log

| Date | Version | Changes |
|---|---|---|
| 2026-05-31 | 0.1.0 | BOOTSTRAP -- IntentSpec synthesized from existing README + AGENTS.md ("Mandates & Core Workflows" section) by intent-alignment Phase 4. 5 outcomes anchored on multi-harness fidelity, lockfile-driven reproducibility, cross-platform reliability, code-quality discipline, self-documenting CLI. 7 Hard constraints lifted verbatim from AGENTS.md mandates. |
| 2026-05-31 | 1.0.0 | **Ratified by owner.** Bootstrap caveat removed; this is the committed intent of record. |
