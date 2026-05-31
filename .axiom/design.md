# Design Document: jup

Date: 2026-05-31
Version: 1.0.0 (ratified 2026-05-31, Full tier)
Type: Design Document
Governance: currency edits = agent-autonomous; new design decisions = collaborative; subordinate to `.axiom/intentspec.md`.

> **Ratified 2026-05-31 (owner).** First Design Document for jup, drafted by intent-alignment Phase 4.5. Full tier (112 tracked files: 69 py + 29 md; published PyPI tool). All 8 ADRs accepted.

## 1. Goals and Intent Link

jup is the multi-harness skill installer/syncer (manifests Objective + O1 harness fidelity, O2 lockfile reproducibility, O3 cross-platform reliability, O4 code-quality discipline, O5 CLI discoverability).

## 2. Constraints

| Constraint | Design obligation |
|---|---|
| C1-C7 AGENTS mandates | enforced through CI + pre-commit + `just qa` + branch-protection conventions |
| S1-S3 dev workflow | subagents for independent steps; adversarial testing; doc updates with every change |

## 3. Context and Scope (ASCII)

```
   GitHub repos exposing skills/ or .claude/skills/
        |
        | `jup install <repo>` -- clone + lockfile entry
        v
   ~/.jup/                                jup CLI surface
   ├── skills/        cache              │ jup install <source>
   ├── lockfile.json  state              │ jup sync (-> per-harness dirs)
   └── config         user config        │ jup list / config show / etc.
        |
        | `jup sync` -- copy or symlink into per-harness dirs
        v
   ~/.gemini/agents/
   ~/.github-copilot/...
   ~/.claude/skills/
   ~/.codex/agents/      (whatever the harness expects)
```

## 4. Solution Strategy

| Choice | Role | Serves | Note |
|---|---|---|---|
| Python 3.12+ | runtime | O3 | targets modern syntax features |
| `typer` | CLI framework | O5 | declarative subcommand registration |
| `pydantic` v2 | config + models | O4 | type-checked config + lockfile schema |
| `rich` | terminal output | O5 | discoverable help + status |
| `uv` | dependency mgmt + tool install | O3 | `uv tool install jup` is the preferred install path |
| `portalocker` (replaces `fcntl`) | cross-platform locking | O3, current branch | fix/windows-fcntl-compat lands the move |
| `~/.jup/` storage root | per-user state | O2, O3 | cache + lockfile + config |
| Lockfile (JSON, pydantic schema) | reproducibility | O2 | source of truth for installed set |
| GitHub Actions: `cd.yml`, `docs.yml`, `publish.yml` | CI/CD | O3, O4 | cross-platform test matrix + docs + PyPI publish |
| `justfile` (commands: `qa`, etc.) | task runner | C3 | `just qa` is the verification entry point |
| `pre-commit-config.yaml` | pre-commit hooks | C1, C3 | conventional-commit + lint + format gate |

## 5. Code-Artifact Map

| Cluster | Files | Serves | Disposition |
|---|---|---|---|
| C01 CLI entry + state | `src/jup/main.py`, `src/jup/context.py`, `src/jup/config.py`, `src/jup/models.py` | O5, O2 | in-scope-core |
| C02 Core business logic | `src/jup/core/{sync,filesystem,lock}.py` | O1, O2, O3 | in-scope-core |
| C03 Per-command modules | `src/jup/commands/` (decoupled from business logic) | O5 | in-scope-core |
| C04 Tests | `tests/` (units + integration) | O1, O4 | in-scope-supporting |
| C05 Repro fixtures | `repro/race_skills/skill[1-10]/SKILL.md` | O3 (race-condition repro) | in-scope-supporting |
| C06 Docs | `docs/{index, reference, lockfile-spec, jup-vs-npx-skills}.md` + `docs/images/*.svg` | O5 | in-scope-supporting |
| C07 CI/CD + tooling | `.github/workflows/{cd,docs,publish}.yml`, `pyproject.toml`, `justfile`, `packaging-guide.sh`, `.pre-commit-config.yaml`, `.python-version` | O3, O4 | in-scope-supporting |
| C08 Governance / scope | `README.md`, `AGENTS.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `backlog/roadmap.md`, `.axiom/{intentspec,design}.md`, `.gitignore` | governance | in-scope-supporting |
| C09 Per-harness fixtures | `.gemini/agents/{advocate-con,advocate-pro,debate-judge}.md` | adversarial testing infra | in-scope-supporting (S2) |
| C10 Skills | `skills/jup/SKILL.md` | self-described skill | in-scope-supporting |

## 6. Decisions (ADR log)

### ADR-001: typer-based CLI with declarative subcommand registration   Status: accepted   2026-05-31
Context: O5. CLI surface must be discoverable and self-documenting.
Decision: `typer` for the CLI; subcommands live in `src/jup/commands/` (decoupled from business logic in `src/jup/core/`); main entry registers subcommands declaratively in `src/jup/main.py`.
Consequences: `jup --help` and `jup <cmd> --help` auto-generated; adding a subcommand is mechanical; business logic stays testable independent of CLI shape.

### ADR-002: Lockfile as the source of truth   Status: accepted   2026-05-31
Context: O2 reproducibility.
Decision: A pydantic-typed JSON lockfile (`~/.jup/lockfile.json`, spec in `docs/lockfile-spec.md`) is the canonical state for what is installed. `install` mutates the lockfile + cache; `sync` reads from lockfile + writes per-harness dirs.
Consequences: any operation that bypasses the lockfile is a bug (e.g., manual edits to `~/.jup/skills/` without a lockfile entry will be removed by next `sync`).

### ADR-003: `~/.jup/` as the per-user storage root   Status: accepted   2026-05-31
Context: O2, O3. Per-user state needs a stable location.
Decision: All state under `~/.jup/`: `skills/` (cache), `lockfile.json` (state), `config` (user config). Per-harness directories are the *output*, not the storage.
Consequences: uninstall = `rm -rf ~/.jup/` + `jup sync` on each harness (which then sees an empty lockfile and removes synced skills). Clean separation between jup-state and harness-state.

### ADR-004: `portalocker` over `fcntl` for cross-platform locking   Status: accepted (in flight on `fix/windows-fcntl-compat`)   2026-05-31
Context: O3 cross-platform. `fcntl` is Unix-only; Windows users hit an ImportError.
Decision: Replace `fcntl`-based locking in `src/jup/core/lock.py` with `portalocker`, which abstracts both Unix (`fcntl`) and Windows (`msvcrt`) underlying APIs.
Consequences: Windows users no longer hit the ImportError; the test matrix can include Windows reliably. Branch `fix/windows-fcntl-compat` lands this.

### ADR-005: GitHub Actions matrix CI (Linux/macOS/Windows)   Status: accepted   2026-05-31
Context: O3. Without cross-platform CI, the cross-platform claim is hopeful.
Decision: `.github/workflows/cd.yml` runs the test matrix across Linux, macOS, and Windows on every PR + main push. `publish.yml` publishes to PyPI from tags. `docs.yml` builds + publishes docs.
Consequences: PR review surface includes cross-platform pass; releases gate on green matrix.

### ADR-006: `just` (justfile) + pre-commit hooks as the quality gate   Status: accepted   2026-05-31
Context: C3 `just qa`, C1 Conventional Commits, code-quality discipline.
Decision: `just qa` is the canonical pre-commit verification command. The `justfile` defines `qa` (lint + type-check + tests) and other developer tasks. `.pre-commit-config.yaml` enforces conventional-commit + lint + format at commit time.
Consequences: a developer who runs `just qa` and `git commit` (with `cz commit` or a conventional message) is automatically compliant with C1, C2, C3.

### ADR-007: Patch-over-minor version bump default   Status: accepted   2026-05-31
Context: C7 (lifted from AGENTS.md mandate).
Decision: When a change qualifies for either patch or minor, bump patch (`0.x.y+1`). Minor bumps are deliberate, owner-collaborative decisions reserved for new public-API surfaces.
Consequences: minor version preserves its semver meaning (new functionality, backward-compatible); patch volume rises but conveys "fix / internal improvement" honestly.

### ADR-008: Adversarial testing as a workflow step (S2)   Status: accepted   2026-05-31
Context: AGENTS.md "Adversarial Phase" step in the dev workflow.
Decision: For significant features / fixes, launch 3 parallel adversarial subagents to find edge cases / implementation flaws / bugs before merge. The `.gemini/agents/{advocate-con,advocate-pro,debate-judge}.md` files are the fixture set.
Consequences: significant changes carry an adversarial-pass record; the cost is per-significant-change agent runs (acceptable for a tool whose correctness across harnesses matters).

## 7. Risks, Tech Debt, and Open Questions

- **Windows compatibility (in flight):** `fix/windows-fcntl-compat` branch carries the `portalocker` migration; merge pending.
- **Per-harness directory contract drift:** harnesses occasionally change their skill-directory format; the `DEFAULT_HARNESSES` registry must keep pace (ADR-001 makes adding/updating mechanical; the alertness is a maintenance commitment).
- **anima_shell boundary:** jup and anima_shell both touch skills; clarity on which is the operator's primary depends on operator-vs-publishable-tool framing (IntentSpec Strategic Context names this).
- **Adversarial-test infra (`.gemini/agents/`):** Gemini-specific fixture currently; cross-harness adversarial fixtures would broaden coverage.
- **Ratified 2026-05-31** (owner). IntentSpec v1.0.0 + design.md v1.0.0 are the committed governance of record. Bootstrap currently lives on `fix/windows-fcntl-compat`; cherry-picked to main 2026-05-31 so the governance layer is on both branches.

## References

- `.axiom/intentspec.md` -- the intent this document manifests.
- `README.md` + `AGENTS.md` -- canonical scope + mandates.
- `docs/{index, reference, lockfile-spec, jup-vs-npx-skills}.md` -- public docs.
- `axiom_framework/specs/design-document-guide-v1.0.0.md` -- the spec this document conforms to.
