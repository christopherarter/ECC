# Design: Reasonix Install Target

- **Date:** 2026-06-09
- **Status:** Approved design, pending spec review
- **Author:** ECC maintainers (chris@arter.dev)

## Summary

Add support for **Reasonix 1.0** (esengine/DeepSeek-Reasonix) — DeepSeek's terminal
coding agent — as a new ECC install target named `reasonix`. The target is
**project-local** (`kind: project`, root `./.reasonix/`) and installs commands,
agents, skills, and flattened rules into `./.reasonix/`, plus a managed
`REASONIX.md` instruction file. It mirrors the existing `zed` project-harness
adapter; almost all the work is following established patterns rather than
inventing new ones.

MCP and hooks integration are explicitly **deferred** — Reasonix's docs confirm
those subsystems exist (`/mcp`, `reasonix mcp install`, `/hooks reload`) but do
not publish their on-disk config paths, so wiring them now would mean guessing.

## Background

### What Reasonix is

Reasonix is a DeepSeek-native, config- and plugin-driven coding agent that runs
in the terminal. Version 1.0 is a ground-up Go rewrite (single static binary,
`npm i -g reasonix`). Confirmed from its v1 CLI reference:

- Project memory/instruction file: **`REASONIX.md`** (plus `~/.reasonix/memory`),
  seeded by `/init`.
- Global config: `~/.reasonix/config.json` (holds the API key — **Reasonix-managed,
  ECC must not write it**).
- Subsystems that exist but whose on-disk paths are **not documented**: skills
  (`/skill list|new`), MCP hub (`/mcp`, `reasonix mcp install`), hooks
  (`/hooks reload`), subagents, custom slash commands.

### How ECC adds a harness target (existing mechanism)

Install targets are adapter objects registered in
`scripts/lib/install-targets/registry.js`. Each adapter is built with the
`createInstallTargetAdapter` factory in `scripts/lib/install-targets/helpers.js`
and declares `id`, `target`, `kind` (`home` | `project`), `rootSegments`, and
`nativeRootRelativePath`. Project harnesses that flatten rules (e.g.
`zed-project.js`, `codebuddy-project.js`) supply a custom `planOperations` that
routes the `rules` source path through `createFlatRuleOperations` and scaffolds
all other module paths.

The set of artifacts a target receives is driven by `manifests/install-modules.json`
(each module lists its `targets` and `paths`), validated against
`schemas/install-modules.schema.json` (which enumerates allowed target ids).
`scripts/lib/install-manifests.js` holds `SUPPORTED_INSTALL_TARGETS` and the
per-target tables (`LEGACY_COMPAT_BASE_MODULE_IDS_BY_TARGET`, etc.).
`scripts/lib/install-targets/helpers.js` owns `PLATFORM_SOURCE_PATH_OWNERS`, which
maps a platform source dir (e.g. `.gemini`) to the target that owns it so other
targets filter it out.

## Goals

- A working `reasonix` install target reachable via
  `./install.sh --target reasonix --profile <name>` and the legacy-language path
  `./install.sh --target reasonix <language>`.
- Installed surface matches the `zed` target: commands, agents, skills, flattened
  rules, plus a managed `REASONIX.md`.
- Full parity with sibling adapters in tests, docs, schema, and the harness
  compliance matrix.

## Non-Goals (Deferred)

- MCP config installation (`.mcp.json` / `reasonix mcp`). Revisit once Reasonix
  documents the on-disk MCP config path.
- Hooks integration (`/hooks`). Revisit once Reasonix documents hook definition
  paths.
- Writing or merging `~/.reasonix/config.json` (Reasonix-managed; out of bounds).
- Home-scoped (`~/.reasonix/`) or `reasonix-project` dual targets. Single
  project-local target only.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Install scope | Project-local `./.reasonix/`, single `reasonix` target | User decision; matches `gemini`/`zed` pattern. |
| Integration depth | Mirror existing project harness (`zed`) | User decision; lowest risk, consistent with repo. |
| Instruction file | Managed `./.reasonix/REASONIX.md`, `GEMINI.md`-style content | Matches sibling convention (managed, install-state-tracked, uninstallable). |
| Legacy-compat | Add `reasonix` to `LEGACY_COMPAT_BASE_MODULE_IDS_BY_TARGET` mirroring `zed` | User decision; bare-language installs work, full `zed` parity. |
| Module set | Every manifest module currently tagged for `zed` | Mirror `zed` exactly so the installed surface (incl. skills) matches by construction, rather than hardcoding a list that could drift. As of writing: `rules-core`, `agents-core`, `commands-core`, `platform-configs`, `workflow-quality`. |

## Architecture / Changes

### New files

1. **`scripts/lib/install-targets/reasonix-project.js`** — adapter built on
   `createInstallTargetAdapter`, modeled directly on `zed-project.js`:
   - `id: 'reasonix-project'`, `target: 'reasonix'`, `kind: 'project'`
   - `rootSegments: ['.reasonix']`, `nativeRootRelativePath: '.reasonix'`
   - `installStatePathSegments: ['ecc-install-state.json']`
   - `planOperations`: flatten `rules` → `./.reasonix/rules/` via
     `createFlatRuleOperations`; scaffold all other paths via
     `createScaffoldOperation`; skip foreign-platform paths via
     `isForeignPlatformPath`.

2. **`.reasonix/REASONIX.md`** — managed instruction file modeled on
   `.gemini/GEMINI.md`: ECC workflow, coding standards, security checklist, and
   pointers to the installed `rules/`, `agents/`, `commands/`, `skills/`. Header
   clearly marks it ECC-managed so it is distinguishable from a user/`/init`
   generated file.

3. **Tests** (see Testing section).

### Edited files (additions to existing lists/tables)

- **`scripts/lib/install-manifests.js`**
  - Add `'reasonix'` to `SUPPORTED_INSTALL_TARGETS`.
  - Add a `reasonix:` entry to `LEGACY_COMPAT_BASE_MODULE_IDS_BY_TARGET` mirroring
    `zed`'s module list (`rules-core`, `agents-core`, `commands-core`,
    `platform-configs`, `workflow-quality`).
- **`scripts/lib/install-targets/registry.js`** — `require('./reasonix-project')`
  and add it to the `ADAPTERS` array.
- **`scripts/lib/install-targets/helpers.js`** — add `'.reasonix': 'reasonix'` to
  `PLATFORM_SOURCE_PATH_OWNERS`.
- **`scripts/install-apply.js`** — add the `reasonix` line to the Targets help text.
- **`schemas/install-modules.schema.json`** — add `"reasonix"` to the `targets`
  enum.
- **`manifests/install-modules.json`** — add `reasonix` to the `targets` array of
  **every module that currently lists `zed`** (so the surface matches `zed` by
  construction, including skills); add `.reasonix` to the `platform-configs`
  module's `paths` so the bundled `.reasonix/` dir syncs.
- **`scripts/lib/harness-adapter-compliance.js`** — add a `reasonix` adapter
  record (state: `Adapter-backed`) with `supported_assets`,
  `install_or_onramp`, `verification_commands`, `source_docs`
  (`.reasonix/REASONIX.md`, `scripts/lib/install-targets/reasonix-project.js`),
  and regenerate the compliance matrix markdown block it feeds.

## Data flow

1. `./install.sh --target reasonix ...` → `scripts/install-apply.js` parses args.
2. `normalizeInstallRequest` resolves `target: 'reasonix'` (validated against
   `SUPPORTED_INSTALL_TARGETS`).
3. `createInstallPlanFromRequest` → `getInstallTargetAdapter('reasonix')` resolves
   the new `reasonix-project` adapter via `supports()`.
4. Adapter `planOperations` emits managed copy operations: rules flattened into
   `./.reasonix/rules/`, other module paths scaffolded under `./.reasonix/`, and
   the `.reasonix/` source dir (REASONIX.md) synced into the target root.
5. `applyInstallPlan` executes operations and writes
   `./.reasonix/ecc-install-state.json` recording ECC-managed files.

## Error handling

- Reuses the factory's `defaultValidateAdapterInput`: project targets require
  `projectRoot`/`repoRoot`, else a `missing-project-root` validation error.
- Foreign-platform source paths (`.gemini`, `.qwen`, `.zed`, ...) are filtered by
  `isForeignPlatformPath` so they never land in `./.reasonix/`.
- No network calls; no writes to `~/.reasonix/config.json`.

## Testing

- **`tests/lib/install-targets.test.js`** — add cases: `getInstallTargetAdapter('reasonix')`
  resolves the adapter; `kind === 'project'`; root resolves to `<project>/.reasonix`;
  `rules` flattens into `.reasonix/rules/`; foreign-platform paths excluded;
  install-state path is `.reasonix/ecc-install-state.json`.
- **`tests/lib/install-manifests.test.js`** — assert `reasonix` ∈
  `SUPPORTED_INSTALL_TARGETS` and has a `LEGACY_COMPAT_BASE_MODULE_IDS_BY_TARGET`
  entry.
- **`tests/scripts/install-apply.test.js`** — assert help text lists `reasonix`;
  a `--target reasonix --dry-run --json` plan resolves to the `reasonix-project`
  adapter with expected operations.
- Update any compliance/doc enumeration tests
  (`tests/docs/install-identifiers.test.js`,
  `tests/docs/configure-ecc-install-paths.test.js`,
  `tests/scripts/install-readme-clarity.test.js`) that assert the target list.
- Run the full suite: `node tests/run-all.js`.

## Open questions / verification items

- **REASONIX.md load path:** Reasonix docs are ambiguous on whether the project
  memory file is read from repo root or `./.reasonix/`. We place ours at
  `./.reasonix/REASONIX.md` (consistent with `.gemini/GEMINI.md` living in the
  target root). Verify against a real Reasonix install; if it only reads repo-root
  `REASONIX.md`, follow up with a placement adjustment.
- **`reasonix.toml` vs `config.json`:** 1.0 release notes mention `reasonix.toml`
  while the v1 CLI ref cites `~/.reasonix/config.json`. ECC writes neither, so this
  does not block the target, but confirm before any future config-bundling work.

## Rollout

Single PR. Conventional commit: `feat(install): add reasonix project install target`.
No migration; purely additive.
