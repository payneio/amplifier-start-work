# amplifier-start: Initial Build Plan

> Created: 2026-02-20
> Source: Distro friction analysis (91 sessions), going-forward.md design

---

## What This Is

An Amplifier bundle (`start:` namespace) that delivers the 6 objectives from
the distro friction analysis as composable, ecosystem-native artifacts. The
value is in the opinions and policies, not infrastructure.

## The Problem (data-driven)

Analysis of 91 user-facing sessions (Jan 23 - Feb 6, 2026) found ~45% of
Amplifier time spent on friction rather than work. Six friction categories
ranked by severity:

| # | Friction | Severity |
|---|----------|----------|
| 1 | Session corruption & repair | Critical |
| 2 | Silent bundle config failures | Critical |
| 3 | Agent competence / trust failures | High |
| 4 | Cross-session context loss | High |
| 5 | Build recovery marathons | High |
| 6 | Environment & sandbox friction | Medium |

## Six Objectives

| Objective | Friction Addressed |
|-----------|--------------------|
| 1. Eliminate silent config failures | #2, #6 |
| 2. Eliminate cross-session context loss | #4, #1 |
| 3. Interchangeable interface viewports | #6, #4 |
| 4. One-time setup, then invisible | #2, #6 |
| 5. Self-healing with diagnostics | #1, #2, #5 |
| 6. Self-improving friction detection | All |

## Target Structure

```
amplifier-start/
  bundle.md                          # Thin manifest
  behaviors/
    start.yaml                       # Main behavior (composes everything)
  context/
    start-conventions.md             # 11 opinions as session context
    start-awareness.md               # Thin pointer for composing bundles
  agents/
    health-checker.md                # In-session diagnostics
    friction-detector.md             # Friction signal analysis
    session-handoff.md               # Session continuity
  recipes/
    morning-brief.yaml               # Daily intelligence brief
    friction-report.yaml             # Weekly friction analysis
  modules/
    hooks-preflight/                 # Pre-flight checks (Python hook)
      pyproject.toml
      amplifier_hooks_preflight/
        __init__.py
    hooks-handoff/                   # Session-end handoff generation (Python hook)
      pyproject.toml
      amplifier_hooks_handoff/
        __init__.py
  planning/                          # Permanent reference
    distro-one-pager.md
    going-forward.md
    bundle-drift.md
    OPINIONS.md
    initial-plan.md                  # This file
  working/
    decisions.md                     # Design decisions log
  docs/
    ARCHITECTURE.md                  # Ring 1/2/3 model
  README.md
```

---

## Build Phases

### Phase 1: Bundle Skeleton (no dependencies)

| Artifact | Purpose |
|----------|---------|
| `bundle.md` | Thin manifest: includes foundation + `start:behaviors/start` |
| `behaviors/start.yaml` | Main behavior: agents + context + hooks |
| `context/start-conventions.md` | 11 opinions ported from OPINIONS.md as session context |
| `context/start-awareness.md` | ~30 line thin pointer for composing bundles |
| `README.md` | What this is, how to use it |
| `docs/ARCHITECTURE.md` | Ring 1/2/3 attention allocation model |

**Deliverable**: Usable bundle. `includes: start:behaviors/start` gives any
session awareness of the environment conventions.

### Phase 2: Agents (no dependencies)

| Agent | Purpose |
|-------|---------|
| `health-checker.md` | In-session diagnostics ("check my env") |
| `friction-detector.md` | Friction signal analysis from session transcripts |
| `session-handoff.md` | Session continuity management |

Each follows the context sink pattern: heavy docs load only when spawned.

### Phase 3: Recipes (depends on Phase 2)

| Recipe | Purpose |
|--------|---------|
| `morning-brief.yaml` | Daily intelligence brief |
| `friction-report.yaml` | Weekly friction analysis |

### Phase 4: Hook Modules

| Hook | Event | Dependency |
|------|-------|------------|
| `hooks-handoff` | `SESSION_END` | None (event exists in core) |
| `hooks-preflight` | `SESSION_PREFLIGHT` | Needs CLI PR (C2) |

---

## Ecosystem Changes

### To amplifier-foundation (branch: papayne/start)

| PR | What | Why |
|----|------|-----|
| F1 | `docs/DIRECTORY_CONTRACT.md` | Formalize `~/.amplifier/` layout as contract |
| F2 | Cache TTL + auto-refresh | Extend cache with TTL, atomic replacement, auto-invalidation |
| F3 | Handoff injection on session start | Detect and inject `handoff.md` as system context |

### To amplifier-app-cli (branch: papayne/start)

| PR | What | Why |
|----|------|-----|
| C1 | `amplifier doctor` command | 13+ diagnostic checks with `--fix` and `--json` |
| C2 | Pre-flight hook point | `SESSION_PREFLIGHT` event between bundle load and prepare |
| C3 | Extend `amplifier init` | Workspace, identity, cache TTL, pre-flight defaults |

### Dependency Map

```
Phase 1 (skeleton)          ── no deps ──────────> usable immediately
Phase 2 (agents)            ── no deps ──────────> usable immediately
Phase 3 (recipes)           ── Phase 2 ──────────> after agents exist
Phase 4a (hooks-handoff)    ── no deps ──────────> SESSION_END exists in core
Phase 4b (hooks-preflight)  ── C2 ───────────────> needs CLI PR first

F3 (handoff injection)      ── no deps ──────────> makes handoff.md auto-load
C1 (doctor)                 ── no deps ──────────> health-checker can reference it
```

---

## Key Design Decisions

See `working/decisions.md` for rationale on each.

| Decision | Choice |
|----------|--------|
| Namespace | `start` |
| Bundle shape | Thin manifest + self-referencing behavior |
| Context strategy | Two files: full conventions + thin awareness |
| Config file | `settings.yaml` (not `distro.yaml`) |
| Secrets | Environment variables (not `keys.yaml`) |
| Handoff format | YAML frontmatter + markdown body |
| Handoff model | Configurable, default haiku-class |
| Health-checker | Self-contained agent, can delegate to `amplifier doctor` when available |
| Behavior split | Single `start.yaml` for now, split later if needed |

---

## What We're NOT Building

- No FastAPI server
- No plugin system
- No bridge abstraction
- No daemon/process management
- No Slack/voice/web chat (standalone experiences, separate concern)
- No second config system (`distro.yaml` / `keys.yaml`)

## Execution Order

1. Copy planning docs (4 files)
2. Write plan + decisions docs
3. bundle.md + behaviors/start.yaml + context files + README + docs
4. All 3 agents
5. hooks-handoff module
6. hooks-preflight module
7. Both recipes
8. Foundation changes (F1-F3)
9. CLI changes (C1-C3)
10. Commit everything

## Source Documents

| Document | Location |
|----------|----------|
| Friction analysis | `planning/distro-one-pager.md` |
| Restructuring design | `planning/going-forward.md` |
| Drift analysis | `planning/bundle-drift.md` |
| 11 conventions | `planning/OPINIONS.md` |
