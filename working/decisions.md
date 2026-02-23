# Design Decisions Log

> Decisions made during implementation. Each captures the choice, alternatives
> considered, and rationale.

---

## D1: Namespace is `start`

**Choice**: Use `start` as the bundle namespace (not `distro`, `env`, `environment`).

**Rationale**: "start" signals "start here" for new users. Clean, short,
no legacy baggage from the distro project. Confirmed by project owner.

---

## D2: Config uses `settings.yaml`, not `distro.yaml`

**Choice**: All configuration goes in `~/.amplifier/settings.yaml`. No
separate `distro.yaml` or `keys.yaml`.

**Rationale**: The going-forward.md design explicitly eliminates the second
config system. One config file for everything. Environment variables for
secrets (standard Amplifier practice). The 11 opinions become recommended
defaults that `amplifier init` writes to `settings.yaml`.

---

## D3: Handoff format is YAML frontmatter + markdown body

**Choice**: `handoff.md` files use YAML frontmatter for structured metadata
(session ID, timestamp, project, files changed) followed by a markdown body
for the human-readable summary.

**Alternatives**: Pure markdown (less parseable), pure YAML (less readable),
JSON (not grep-able).

**Rationale**: YAML frontmatter is the Amplifier convention for structured
metadata in markdown files (bundles, agents all use it). Markdown body is
human-readable and can be injected directly as session context. Best of both.

---

## D4: Handoff model defaults to haiku-class

**Choice**: The hooks-handoff module defaults to a haiku-class model
(cheapest/fastest that can summarize) but is configurable via behavior config.

**Rationale**: Handoff generation runs on every session end. It needs to be
fast (<2 seconds) and cheap. A haiku-class model is sufficient for
summarization. Users who want higher quality can configure a stronger model.

**Config path**: `hooks.handoff.model` in behavior config, falls back to
the session's configured provider with smallest available model.

---

## D5: Health-checker agent is self-contained

**Choice**: The health-checker agent performs diagnostic checks itself using
available tools (bash, filesystem), rather than requiring `amplifier doctor`
to exist first.

**Alternatives**: Agent delegates to `amplifier doctor` CLI command.

**Rationale**: The agent needs to work immediately, before the CLI PR lands.
When `amplifier doctor` exists, the agent can optionally use it, but isn't
dependent on it. The 13 diagnostic checks are encoded as agent knowledge,
not as a hard dependency on an external command.

---

## D6: Single behavior file for now

**Choice**: One `behaviors/start.yaml` that includes everything, rather than
splitting into `health-checks.yaml`, `friction-detection.yaml`, etc.

**Alternatives**: Multiple behavior files for granular composition.

**Rationale**: Start simple. If users want to compose only handoff without
health checks, we can split later. Over-splitting before we have real usage
data creates unnecessary complexity. The behavior can always be decomposed
without breaking existing consumers.

---

## D7: Preflight hook registers for session lifecycle events

**Choice**: The preflight hook module registers for an early session event
to run checks. Initially it can use `tool:pre` on the first tool call as
a proxy until a dedicated `SESSION_PREFLIGHT` event exists in the CLI.

**Alternatives**: Wait for CLI PR to add `SESSION_PREFLIGHT` event.

**Rationale**: We want the hook to be functional immediately. Using the first
`tool:pre` event as a trigger means checks run before the first real work
happens. When the dedicated event lands, we switch to that. The hook checks
a flag to ensure it only runs once per session.

---

## D8: Context conventions adapted from OPINIONS.md for Amplifier-native use

**Choice**: The 11 opinions are reformatted for session context. References
to `distro.yaml` are updated to `settings.yaml`. References to `amp distro`
commands are updated to `amplifier` commands. The distro-specific framing
is generalized.

**Rationale**: The opinions are the core value. But they reference a config
system (`distro.yaml`, `keys.yaml`) and CLI (`amp distro`) that we're
deliberately not building. The conventions must reflect the actual
ecosystem-native approach.

---

## D9: Hook modules use subdirectory source URIs

**Choice**: Hook modules live in `modules/hooks-handoff/` and
`modules/hooks-preflight/` within the bundle repo. They're referenced via
git+URL with `#subdirectory=` fragment.

**Rationale**: Keeps everything in one repo. The canonical pattern for
bundles that ship custom modules (amplifier-bundle-recipes does this for
tool-recipes). Each module is a proper Python package with its own
pyproject.toml.

---

## D10: Recipes delegate to agents, not direct tool calls

**Choice**: The morning-brief and friction-report recipes delegate work to
the bundle's agents (session-handoff, friction-detector) rather than making
direct tool calls.

**Rationale**: Agents carry the domain knowledge and heavy context. Recipes
orchestrate the workflow. This separation means the agents can be used
independently (in-session delegation) or as part of a recipe workflow.

---

## D11: No `workspace_root` in settings.yaml

**Choice**: Removed `workspace_root` from `amplifier init` and from the
conventions. Project identity is derived from the working directory name
(`Path(cwd).name`), not from a configured workspace root.

**Alternatives**: Configure `workspace_root` (e.g. `~/dev/`) in settings.yaml
so tools can enumerate all projects and reverse-map slugs to code paths.

**Rationale**: OPINIONS.md Opinion #1 conflated several concepts:
- **Project identity** (which project am I in?) — already solved by deriving
  slug from `Path(cwd).name`. No config needed.
- **Project state** (where does Amplifier store data?) — already solved by
  `~/.amplifier/projects/<slug>/`. No config needed.
- **Code location** (where are my git repos?) — the only thing `workspace_root`
  would add. But nothing in the current scope consumes this.
- **Project enumeration** (list all projects) — can scan
  `~/.amplifier/projects/` for projects with sessions. Scanning a code
  directory for projects without sessions is a multi-interface feature
  not in scope.

The convention now recommends organizing repos under a single root as
guidance, but doesn't configure it in Amplifier. When a feature genuinely
needs to enumerate code directories (project discovery, multi-interface
navigation), it should be added then — with a clearer name like `code_root`
rather than the ambiguous `workspace_root`.

---

## D12: Handoffs are per-session, not per-project

**Choice**: Each session gets its own `handoff.md` inside its session
directory (`~/.amplifier/projects/<slug>/sessions/<session-id>/handoff.md`),
not one shared file at the project level.

**Original design**: One `handoff.md` per project, overwritten by each
session.

**Rationale**: Per-project handoffs have three problems:
1. **Parallel sessions clobber**: Two concurrent sessions in the same
   project overwrite each other's handoff.
2. **History loss**: Only the latest session's handoff survives. Can't
   answer "what happened in my last 3 sessions?"
3. **No selective resume**: Can't pick up from session A if session B
   wrote last.

Per-session handoffs solve all three. The most recent handoff is found by
scanning session directories and picking the latest timestamp. Multiple
handoffs can be surveyed for morning briefs and friction reports.

---

## D13: No handoff utilities in amplifier-foundation

**Choice**: Removed `handoff.py` from amplifier-foundation. The hook writes
handoff files, and the session-handoff agent reads them using standard file
tools. No foundation-level API for handoffs.

**Alternatives**: Foundation provides `find_handoff()`, `read_handoff()`,
`format_handoff_context()`, `inject_handoff_if_available()` as a shared
library.

**Rationale**: The handoff pattern is unproven. We haven't validated that
the hook-generated summaries are useful, that the format is right, or that
other bundles want this capability. Adding it to foundation promotes it to
a shared contract before we have usage data. If the pattern proves valuable,
we can promote the utilities then. YAGNI.
