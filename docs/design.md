# jup implementation design

Date: 2026-09-14
Version: 2.0.0
Status: active implementation design

This repository owns jup's implementation contracts. Project purpose, outcomes,
constraints, and decision authority belong in the canonical
[Project Docs authority](https://github.com/andessen/project_docs/blob/main/projects/jup/INTENT.md).
The accepted decisions below are preserved from the retired governance record.

## Context and scope

jup is a lightweight, local-first command-line tool for installing and syncing
agent skills across supported AI harnesses. It is intentionally narrower than
anima_shell: jup distributes skills; it does not own identity, memory, persona,
or the wider operator substrate.

```
Git or local skill source
          |
          v
~/.jup/skills + lockfile.json + config
          |
          v
documented per-harness skill directories
```

The lockfile is authoritative for the managed skill set. Harness directories
are outputs. jup must not modify harness-side state outside documented managed
directories.

## Implementation strategy

| Choice | Responsibility |
|---|---|
| Python 3.12+ | Cross-platform runtime |
| Typer | Declarative CLI and generated help |
| Pydantic v2 | Configuration and lockfile contracts |
| Rich | Terminal presentation |
| uv | Dependency and tool installation |
| portalocker | Cross-platform state locking |
| `~/.jup/` | Per-user cache, lockfile, and configuration |
| GitHub Actions | Platform matrix, documentation, and release automation |
| `just qa` | Lint, type-check, and test entry point |

Core behavior lives in `src/jup/core/`; command adapters live in
`src/jup/commands/`; configuration and public models live in
`src/jup/config.py` and `src/jup/models.py`. User documentation belongs in
`README.md`, `CONTRIBUTING.md`, and `docs/`. The bundled skills describe
supported workflows but do not override the canonical project authority.

## Accepted decisions

### ADR-001: Typer CLI with separated command adapters

Public commands register declaratively in `src/jup/main.py`; business logic
remains independently testable under `src/jup/core/`.

### ADR-002: Lockfile is the managed-state authority

Install mutates the cache and lockfile; sync reads that state and writes only to
documented harness targets. Manual cache changes without a lockfile entry are
not durable managed state.

### ADR-003: `~/.jup/` is the per-user storage root

Skills, lockfile, and configuration live under one per-user root. Harness
directories remain delivery targets, not jup's primary store.

### ADR-004: Portalocker provides cross-platform locking

Use portalocker rather than Unix-only `fcntl` so state locking has a supported
Windows path as well as macOS and Linux paths.

### ADR-005: CI covers Linux, macOS, and Windows

The test matrix is the evidence surface for the cross-platform claim. Workflow
or harness changes require corresponding platform evidence.

### ADR-006: `just qa` is the contributor quality entry point

Lint, type checking, and tests run through one documented command. Pre-commit
hooks complement CI but do not replace it.

### ADR-007: Patch-over-minor is the default release choice

Use patch releases when a change does not create a new public API surface.
Minor and major release intent remains an owner decision.

### ADR-008: Significant changes receive adversarial review

For material installer, sync, filesystem, or harness-contract changes, exercise
edge cases and failure modes before merge. Existing Gemini agent files are
review aids, not independent assurance.

## Preserved failure modes

- Path traversal, normalization ambiguity, and Git argument injection must fail
  before filesystem or process effects.
- Concurrent or interrupted operations must not corrupt the lockfile or install
  partial managed state.
- A stale lock must have a bounded and observable recovery path.
- Network failure must not silently replace or truncate an installed skill.
- Harness contract drift must not cause writes outside the managed directory.
- Release automation must not publish without the project's human gate.

## Open implementation questions

- Keep the supported harness registry aligned with changing harness contracts.
- Decide whether cross-harness adversarial fixtures are warranted beyond the
  existing Gemini-focused review aids.
- Reassess the boundary with anima_shell if either tool expands its skill-sync
  responsibilities.
