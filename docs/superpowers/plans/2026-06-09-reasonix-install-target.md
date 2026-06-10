# Reasonix Install Target Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a project-local `reasonix` install target so `./install.sh --target reasonix ...` installs ECC commands, agents, skills, flattened rules, and a managed `REASONIX.md` into `./.reasonix/`.

**Architecture:** Mirror the existing `zed` project-harness adapter. A new `reasonix-project` adapter is built with the `createInstallTargetAdapter` factory and registered in the install-targets registry. The target is added to both JSON schemas, `SUPPORTED_INSTALL_TARGETS`, the platform-path-owner map, the manifest module `targets` arrays (everywhere `zed` appears), the legacy-compat table, the installer help text, and the harness-adapter-compliance scorecard.

**Tech Stack:** Node.js (CommonJS), plain `.js`, JSON manifests/schemas, custom test harness (`node tests/run-all.js`).

**Spec:** `docs/superpowers/specs/2026-06-09-reasonix-install-target-design.md`

**Branch:** `feat/reasonix-install-target` (already created; the spec is committed there).

---

## File Structure

**Create:**
- `scripts/lib/install-targets/reasonix-project.js` — the adapter (clone of `zed-project.js`).
- `.reasonix/REASONIX.md` — managed instruction/memory file bundled in the repo and synced into the target.

**Modify:**
- `scripts/lib/install-targets/registry.js` — register the adapter.
- `scripts/lib/install-targets/helpers.js` — add `.reasonix` to `PLATFORM_SOURCE_PATH_OWNERS`.
- `scripts/lib/install-manifests.js` — add `reasonix` to `SUPPORTED_INSTALL_TARGETS` and `LEGACY_COMPAT_BASE_MODULE_IDS_BY_TARGET`.
- `schemas/ecc-install-config.schema.json` — add `reasonix` to `properties.target.enum`.
- `schemas/install-modules.schema.json` — add `reasonix` to the module `targets` enum.
- `manifests/install-modules.json` — add `reasonix` to every module `targets` array that lists `zed`; add `.reasonix` to the `platform-configs` module `paths`.
- `scripts/install-apply.js` — add the `reasonix` line to the help text.
- `scripts/lib/harness-adapter-compliance.js` — add the `reasonix` adapter record.
- `docs/architecture/harness-adapter-compliance.md` — regenerate the matrix block.

**Test:**
- `tests/lib/install-targets.test.js` — adapter resolution, lookup, and planning.
- `tests/scripts/install-apply.test.js` — end-to-end install of the target.

---

## Task 1: Register the `reasonix` project adapter

This task makes `getInstallTargetAdapter('reasonix')` resolve and keeps the schema/SUPPORTED regression guards green. All of these files must change together because `tests/lib/install-targets.test.js` has guards asserting every adapter target appears in both `SUPPORTED_INSTALL_TARGETS` and `schemas/ecc-install-config.schema.json`.

**Files:**
- Create: `scripts/lib/install-targets/reasonix-project.js`
- Modify: `scripts/lib/install-targets/registry.js`
- Modify: `scripts/lib/install-targets/helpers.js`
- Modify: `scripts/lib/install-manifests.js:7`
- Modify: `schemas/ecc-install-config.schema.json:20-32`
- Modify: `schemas/install-modules.schema.json:50-62`
- Test: `tests/lib/install-targets.test.js`

- [ ] **Step 1: Write the failing tests**

In `tests/lib/install-targets.test.js`, add `reasonix` to the existing `lists supported target adapters` test. Find this line (inside that test, after the `zed` assertion at ~line 51):

```javascript
    assert.ok(targets.includes('zed'), 'Should include zed target');
```

Add immediately after it:

```javascript
    assert.ok(targets.includes('reasonix'), 'Should include reasonix target');
```

Then add three new test blocks. Insert them immediately **before** the final `console.log(`\nResults: Passed: ${passed}, Failed: ${failed}`);` line (currently ~line 1091):

```javascript
  if (test('resolves reasonix adapter root and install-state path from project root', () => {
    const adapter = getInstallTargetAdapter('reasonix');
    const projectRoot = '/workspace/app';
    const root = adapter.resolveRoot({ projectRoot });
    const statePath = adapter.getInstallStatePath({ projectRoot });

    assert.strictEqual(adapter.id, 'reasonix-project');
    assert.strictEqual(adapter.target, 'reasonix');
    assert.strictEqual(adapter.kind, 'project');
    assert.strictEqual(root, path.join(projectRoot, '.reasonix'));
    assert.strictEqual(statePath, path.join(projectRoot, '.reasonix', 'ecc-install-state.json'));
  })) passed++; else failed++;

  if (test('reasonix adapter supports lookup by target and adapter id', () => {
    const byTarget = getInstallTargetAdapter('reasonix');
    const byId = getInstallTargetAdapter('reasonix-project');

    assert.strictEqual(byTarget.id, 'reasonix-project');
    assert.strictEqual(byId.id, 'reasonix-project');
    assert.ok(byTarget.supports('reasonix'));
    assert.ok(byTarget.supports('reasonix-project'));
  })) passed++; else failed++;

  if (test('plans reasonix memory, commands, agents, skills, and flattened rules', () => {
    const repoRoot = path.join(__dirname, '..', '..');
    const projectRoot = '/workspace/app';

    const plan = planInstallTargetScaffold({
      target: 'reasonix',
      repoRoot,
      projectRoot,
      modules: [
        {
          id: 'rules-core',
          paths: ['rules'],
        },
        {
          id: 'agents-core',
          paths: ['agents'],
        },
        {
          id: 'commands-core',
          paths: ['commands'],
        },
        {
          id: 'platform-configs',
          paths: ['.reasonix', '.gemini', 'mcp-configs'],
        },
        {
          id: 'workflow-quality',
          paths: ['skills/tdd-workflow'],
        },
      ],
    });

    assert.strictEqual(plan.adapter.id, 'reasonix-project');
    assert.strictEqual(plan.targetRoot, path.join(projectRoot, '.reasonix'));
    assert.strictEqual(plan.installStatePath, path.join(projectRoot, '.reasonix', 'ecc-install-state.json'));
    assert.ok(
      plan.operations.some(operation => (
        normalizedRelativePath(operation.sourceRelativePath) === '.reasonix'
        && operation.destinationPath === path.join(projectRoot, '.reasonix')
        && operation.strategy === 'sync-root-children'
      )),
      'Should sync the bundled .reasonix memory dir into .reasonix'
    );
    assert.ok(
      plan.operations.some(operation => (
        normalizedRelativePath(operation.sourceRelativePath) === 'rules/common/coding-style.md'
        && operation.destinationPath === path.join(projectRoot, '.reasonix', 'rules', 'common-coding-style.md')
      )),
      'Should flatten common rules into namespaced files for reasonix'
    );
    assert.ok(
      plan.operations.some(operation => (
        normalizedRelativePath(operation.sourceRelativePath) === 'agents'
        && operation.destinationPath === path.join(projectRoot, '.reasonix', 'agents')
      )),
      'Should install agents under .reasonix/agents'
    );
    assert.ok(
      plan.operations.some(operation => (
        normalizedRelativePath(operation.sourceRelativePath) === 'commands'
        && operation.destinationPath === path.join(projectRoot, '.reasonix', 'commands')
      )),
      'Should install commands under .reasonix/commands'
    );
    assert.ok(
      plan.operations.some(operation => (
        normalizedRelativePath(operation.sourceRelativePath) === 'skills/tdd-workflow'
        && operation.destinationPath === path.join(projectRoot, '.reasonix', 'skills', 'tdd-workflow')
      )),
      'Should install skills under .reasonix/skills'
    );
    assert.ok(
      !plan.operations.some(operation => normalizedRelativePath(operation.sourceRelativePath) === '.gemini'),
      'Should skip foreign Gemini platform config paths'
    );
  })) passed++; else failed++;
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `node tests/lib/install-targets.test.js`
Expected: FAIL — `getInstallTargetAdapter('reasonix')` throws `Unknown install target adapter: reasonix` (the adapter does not exist yet), and the `lists supported target adapters` assertion fails.

- [ ] **Step 3: Create the adapter**

Create `scripts/lib/install-targets/reasonix-project.js` with this exact content (a clone of `zed-project.js` with reasonix identifiers):

```javascript
const path = require('path');

const {
  createFlatRuleOperations,
  createInstallTargetAdapter,
  isForeignPlatformPath,
} = require('./helpers');

module.exports = createInstallTargetAdapter({
  id: 'reasonix-project',
  target: 'reasonix',
  kind: 'project',
  rootSegments: ['.reasonix'],
  installStatePathSegments: ['ecc-install-state.json'],
  nativeRootRelativePath: '.reasonix',
  planOperations(input, adapter) {
    const modules = Array.isArray(input.modules)
      ? input.modules
      : (input.module ? [input.module] : []);
    const {
      repoRoot,
      projectRoot,
      homeDir,
    } = input;
    const planningInput = {
      repoRoot,
      projectRoot,
      homeDir,
    };
    const targetRoot = adapter.resolveRoot(planningInput);

    return modules.flatMap(module => {
      const paths = Array.isArray(module.paths) ? module.paths : [];
      return paths
        .filter(p => !isForeignPlatformPath(p, adapter.target))
        .flatMap(sourceRelativePath => {
          if (sourceRelativePath === 'rules') {
            return createFlatRuleOperations({
              moduleId: module.id,
              repoRoot,
              sourceRelativePath,
              destinationDir: path.join(targetRoot, 'rules'),
            });
          }

          return [adapter.createScaffoldOperation(module.id, sourceRelativePath, planningInput)];
        });
    });
  },
});
```

- [ ] **Step 4: Register the adapter in the registry**

In `scripts/lib/install-targets/registry.js`, add the require near the other requires (after the `qwenHome`/`zedProject` requires):

Find:

```javascript
const zedProject = require('./zed-project');
```

Add immediately after:

```javascript
const reasonixProject = require('./reasonix-project');
```

Then add it to the `ADAPTERS` array. Find:

```javascript
  qwenHome,
  zedProject,
]);
```

Replace with:

```javascript
  qwenHome,
  zedProject,
  reasonixProject,
]);
```

- [ ] **Step 5: Add `reasonix` to the platform-path-owner map**

In `scripts/lib/install-targets/helpers.js`, find the `PLATFORM_SOURCE_PATH_OWNERS` object. Find:

```javascript
  '.qwen': 'qwen',
  '.zed': 'zed',
});
```

Replace with:

```javascript
  '.qwen': 'qwen',
  '.reasonix': 'reasonix',
  '.zed': 'zed',
});
```

- [ ] **Step 6: Add `reasonix` to `SUPPORTED_INSTALL_TARGETS`**

In `scripts/lib/install-manifests.js:7`, find:

```javascript
const SUPPORTED_INSTALL_TARGETS = ['claude', 'claude-project', 'cursor', 'antigravity', 'codex', 'gemini', 'opencode', 'codebuddy', 'joycode', 'qwen', 'zed'];
```

Replace with:

```javascript
const SUPPORTED_INSTALL_TARGETS = ['claude', 'claude-project', 'cursor', 'antigravity', 'codex', 'gemini', 'opencode', 'codebuddy', 'joycode', 'qwen', 'zed', 'reasonix'];
```

- [ ] **Step 7: Add `reasonix` to `schemas/ecc-install-config.schema.json`**

In `schemas/ecc-install-config.schema.json`, find the `target` enum:

```json
        "qwen",
        "zed"
      ]
    },
    "profile": {
```

Replace with:

```json
        "qwen",
        "zed",
        "reasonix"
      ]
    },
    "profile": {
```

- [ ] **Step 8: Add `reasonix` to `schemas/install-modules.schema.json`**

In `schemas/install-modules.schema.json`, find the module `targets` enum:

```json
                "qwen",
                "zed"
              ]
```

Replace with:

```json
                "qwen",
                "zed",
                "reasonix"
              ]
```

- [ ] **Step 9: Run the tests to verify they pass**

Run: `node tests/lib/install-targets.test.js`
Expected: PASS — all three new tests pass, the `lists supported target adapters` test passes, and the schema/SUPPORTED regression guards stay green.

- [ ] **Step 10: Commit**

```bash
git add scripts/lib/install-targets/reasonix-project.js scripts/lib/install-targets/registry.js scripts/lib/install-targets/helpers.js scripts/lib/install-manifests.js schemas/ecc-install-config.schema.json schemas/install-modules.schema.json tests/lib/install-targets.test.js
git commit -m "feat(install): register reasonix project install-target adapter"
```

---

## Task 2: Bundle `REASONIX.md`, wire the manifest, help text, and legacy-compat

This task makes the target installable end-to-end: the manifest modules now include `reasonix`, the bundled `.reasonix/REASONIX.md` syncs into the install, legacy-language installs resolve a sensible module set, and the installer help text documents the target.

**Files:**
- Create: `.reasonix/REASONIX.md`
- Modify: `manifests/install-modules.json`
- Modify: `scripts/lib/install-manifests.js` (`LEGACY_COMPAT_BASE_MODULE_IDS_BY_TARGET`)
- Modify: `scripts/install-apply.js` (help text)
- Test: `tests/scripts/install-apply.test.js`

- [ ] **Step 1: Write the failing end-to-end test**

In `tests/scripts/install-apply.test.js`, add a new test block immediately **after** the `installs Qwen profile through managed home install-state` test (it ends at ~line 302 with `})) passed++; else failed++;`). Insert:

```javascript
  if (test('installs the reasonix project target with bundled memory and assets', () => {
    const homeDir = createTempDir('install-apply-home-');
    const projectDir = createTempDir('install-apply-project-');

    try {
      const result = run(['--target', 'reasonix', '--profile', 'minimal'], { cwd: projectDir, homeDir });
      assert.strictEqual(result.code, 0, result.stderr);

      assert.ok(fs.existsSync(path.join(projectDir, '.reasonix', 'REASONIX.md')));
      assert.ok(fs.existsSync(path.join(projectDir, '.reasonix', 'rules', 'common-coding-style.md')));
      assert.ok(!fs.existsSync(path.join(projectDir, '.reasonix', 'rules', 'common', 'coding-style.md')));
      assert.ok(fs.existsSync(path.join(projectDir, '.reasonix', 'agents', 'architect.md')));
      assert.ok(fs.existsSync(path.join(projectDir, '.reasonix', 'commands', 'plan.md')));
      assert.ok(fs.existsSync(path.join(projectDir, '.reasonix', 'skills', 'tdd-workflow', 'SKILL.md')));
      assert.ok(!fs.existsSync(path.join(projectDir, '.reasonix', 'hooks')));

      const statePath = path.join(projectDir, '.reasonix', 'ecc-install-state.json');
      const state = readJson(statePath);
      assert.strictEqual(state.target.id, 'reasonix-project');
      assert.strictEqual(state.request.profile, 'minimal');
      assert.ok(state.resolution.selectedModules.includes('workflow-quality'));
      assert.ok(
        state.operations.some(operation => (
          operation.destinationPath.endsWith(path.join('.reasonix', 'skills', 'tdd-workflow', 'SKILL.md'))
        )),
        'Should record reasonix skill file operation'
      );
    } finally {
      cleanup(homeDir);
      cleanup(projectDir);
    }
  })) passed++; else failed++;
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `node tests/scripts/install-apply.test.js`
Expected: FAIL — the `reasonix` profile resolves no modules (no manifest module lists `reasonix` yet), so `.reasonix/REASONIX.md` and the asset files do not exist.

- [ ] **Step 3: Create the bundled `REASONIX.md`**

Create `.reasonix/REASONIX.md` with this exact content (modeled on `.gemini/GEMINI.md`, adapted to Reasonix, and clearly marked ECC-managed):

```markdown
# ECC for Reasonix

> **ECC-managed file.** Installed by Everything Claude Code via
> `./install.sh --target reasonix`. Edits here are overwritten on reinstall.
> Keep your own project memory in a separate `REASONIX.md` if you run
> `reasonix /init`.

This file provides Reasonix with the baseline ECC workflow, review standards, and security checks for repositories that install the Reasonix target.

## Overview

Everything Claude Code (ECC) is a cross-harness coding system of specialized agents, skills, and commands. The Reasonix target installs ECC's commands, agents, skills, and flattened rules into `./.reasonix/` so they are available inside DeepSeek Reasonix sessions.

## Core Workflow

1. Plan before editing large features.
2. Prefer test-first changes for bug fixes and new functionality.
3. Review for security before shipping.
4. Keep changes self-contained, readable, and easy to revert.

## Coding Standards

- Prefer immutable updates over in-place mutation.
- Keep functions small and files focused.
- Validate user input at boundaries.
- Never hardcode secrets.
- Fail loudly with clear error messages instead of silently swallowing problems.

## Security Checklist

Before any commit:

- No hardcoded API keys, passwords, or tokens
- All external input validated
- Parameterized queries for database writes
- Sanitized HTML output where applicable
- Authz/authn checked for sensitive paths
- Error messages scrubbed of sensitive internals

## Delivery Standards

- Use conventional commits: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`
- Run targeted verification for touched areas before shipping
- Prefer contained local implementations over adding new third-party runtime dependencies

## ECC Areas To Reuse

- `.reasonix/rules/` for repo-wide operating rules
- `.reasonix/skills/` for deep workflow guidance
- `.reasonix/commands/` for slash-command patterns worth adapting into prompts/macros
- `.reasonix/agents/` for specialized subagent definitions
```

- [ ] **Step 4: Add `reasonix` to every manifest module that targets `zed`**

In `manifests/install-modules.json`, every module `targets` array that ends with `"zed"` lists it as the final element (8-space indent, no trailing comma, followed by a 6-space-indented `],`). Add `reasonix` after each. Use a single replace-all edit.

Replace all occurrences of this exact block:

```json
        "zed"
      ],
```

with:

```json
        "zed",
        "reasonix"
      ],
```

Expected: 21 occurrences replaced (one per module that targets zed: rules-core, agents-core, commands-core, platform-configs, framework-language, database, workflow-quality, optimization-workflows, security, research-apis, business-content, operator-workflows, prediction-market-skills, social-distribution, media-generation, swift-apple, agentic-patterns, devops-infra, machine-learning, supply-chain-domain, document-processing).

- [ ] **Step 5: Add `.reasonix` to the `platform-configs` module paths**

In `manifests/install-modules.json`, inside the `platform-configs` module's `paths` array, find:

```json
        ".qwen",
        ".zed",
```

Replace with:

```json
        ".qwen",
        ".reasonix",
        ".zed",
```

- [ ] **Step 6: Verify the manifest is still valid JSON with the expected edits**

Run: `node -e "const m=require('./manifests/install-modules.json'); const n=m.modules.filter(x=>x.targets.includes('reasonix')).length; const z=m.modules.filter(x=>x.targets.includes('zed')).length; const pc=m.modules.find(x=>x.id==='platform-configs'); if(n!==z) throw new Error('reasonix count '+n+' != zed count '+z); if(!pc.paths.includes('.reasonix')) throw new Error('.reasonix missing from platform-configs paths'); console.log('OK reasonix='+n+' zed='+z);"`
Expected: prints `OK reasonix=21 zed=21` (valid JSON, reasonix mirrors zed exactly, `.reasonix` path present).

- [ ] **Step 7: Add the legacy-compat base module list for `reasonix`**

In `scripts/lib/install-manifests.js`, in `LEGACY_COMPAT_BASE_MODULE_IDS_BY_TARGET`, find the `zed` entry:

```javascript
  zed: [
    'rules-core',
    'agents-core',
    'commands-core',
    'platform-configs',
    'workflow-quality',
  ],
});
```

Replace with:

```javascript
  zed: [
    'rules-core',
    'agents-core',
    'commands-core',
    'platform-configs',
    'workflow-quality',
  ],
  reasonix: [
    'rules-core',
    'agents-core',
    'commands-core',
    'platform-configs',
    'workflow-quality',
  ],
});
```

- [ ] **Step 8: Add the `reasonix` line to the installer help text**

In `scripts/install-apply.js`, in `getHelpText()`, find the `zed` line:

```javascript
  zed          - Install project settings, commands, agents, skills, and flattened rules into ./.zed/
```

Replace with:

```javascript
  zed          - Install project settings, commands, agents, skills, and flattened rules into ./.zed/
  reasonix     - Install commands, agents, skills, and flattened rules into ./.reasonix/
```

- [ ] **Step 9: Run the end-to-end test to verify it passes**

Run: `node tests/scripts/install-apply.test.js`
Expected: PASS — the new `installs the reasonix project target with bundled memory and assets` test passes and no existing test regresses.

- [ ] **Step 10: Commit**

```bash
git add .reasonix/REASONIX.md manifests/install-modules.json scripts/lib/install-manifests.js scripts/install-apply.js tests/scripts/install-apply.test.js
git commit -m "feat(install): wire reasonix manifest modules, bundled REASONIX.md, and help text"
```

---

## Task 3: Add the harness-adapter-compliance record and regenerate the matrix

The compliance CLI is validate-only; the guard test (`tests/docs/harness-adapter-compliance.test.js`) asserts the doc's matrix block equals `renderMarkdownTable()`. Adding an adapter record requires regenerating that block in the doc.

**Files:**
- Modify: `scripts/lib/harness-adapter-compliance.js` (`ADAPTER_RECORDS`)
- Modify: `docs/architecture/harness-adapter-compliance.md` (matrix block)
- Test: `tests/docs/harness-adapter-compliance.test.js`

- [ ] **Step 1: Run the compliance check to confirm the current baseline passes**

Run: `node scripts/harness-adapter-compliance.js --check`
Expected: `Harness Adapter Compliance: PASS` (baseline is in sync before edits).

- [ ] **Step 2: Add the `reasonix` adapter record**

In `scripts/lib/harness-adapter-compliance.js`, find the end of the `zed` record (it closes just before the `dmux` record begins):

```javascript
    source_docs: [
      '.zed/settings.json',
      'scripts/lib/install-targets/zed-project.js',
      'docs/architecture/cross-harness.md',
      'tests/lib/install-targets.test.js',
    ],
  },
  {
    id: 'dmux',
```

Replace with:

```javascript
    source_docs: [
      '.zed/settings.json',
      'scripts/lib/install-targets/zed-project.js',
      'docs/architecture/cross-harness.md',
      'tests/lib/install-targets.test.js',
    ],
  },
  {
    id: 'reasonix',
    harness: 'Reasonix',
    state: 'Adapter-backed',
    supported_assets: [
      'managed REASONIX.md memory',
      'flattened project rules',
      'shared skills',
      'commands',
      'agents',
    ],
    unsupported_surfaces: [
      'MCP and hooks integration deferred until Reasonix documents their on-disk config paths',
    ],
    install_or_onramp: ['`./install.sh --profile minimal --target reasonix`'],
    verification_commands: [
      '`node tests/lib/install-targets.test.js`',
      '`node tests/scripts/install-apply.test.js`',
    ],
    risk_notes: ['Do not write the Reasonix-managed `~/.reasonix/config.json`; it holds the API key.'],
    last_verified_at: '2026-06-09',
    owner: 'ECC maintainers',
    source_docs: [
      '.reasonix/REASONIX.md',
      'scripts/lib/install-targets/reasonix-project.js',
      'tests/lib/install-targets.test.js',
    ],
  },
  {
    id: 'dmux',
```

- [ ] **Step 3: Confirm the record is valid but the doc is now out of sync**

Run: `node scripts/harness-adapter-compliance.js --check`
Expected: FAIL with a non-zero exit and an error about the matrix block in `docs/architecture/harness-adapter-compliance.md` not being generated from adapter records (the record is valid, but the committed doc table lacks the Reasonix row).

- [ ] **Step 4: Print the regenerated matrix table**

Run: `node scripts/harness-adapter-compliance.js --format=markdown`
Expected: prints the full Markdown table including a new `Reasonix | Adapter-backed | ...` row. Copy this entire table output.

- [ ] **Step 5: Replace the matrix block in the doc**

In `docs/architecture/harness-adapter-compliance.md`, replace everything **between** the markers `<!-- harness-adapter-compliance:matrix-start -->` and `<!-- harness-adapter-compliance:matrix-end -->` with the exact table printed in Step 4 (keep the marker comment lines themselves). Preserve a blank line before/after the table if the original had them.

- [ ] **Step 6: Verify the compliance check passes**

Run: `node scripts/harness-adapter-compliance.js --check`
Expected: `Harness Adapter Compliance: PASS` and `Adapters: <N>` (the count incremented by one).

- [ ] **Step 7: Run the compliance guard test**

Run: `node tests/docs/harness-adapter-compliance.test.js`
Expected: PASS — `adapter compliance matrix is generated from source data` and `adapter compliance CLI check passes against the committed doc` both pass.

- [ ] **Step 8: Commit**

```bash
git add scripts/lib/harness-adapter-compliance.js docs/architecture/harness-adapter-compliance.md
git commit -m "docs(install): record reasonix in the harness adapter compliance matrix"
```

---

## Task 4: Full verification and finalize

**Files:** none (verification only).

- [ ] **Step 1: Run the full test suite**

Run: `node tests/run-all.js`
Expected: the suite passes with no failures. If a docs/enumeration test fails because it asserts a target list that now omits `reasonix`, read the failure, add `reasonix` to that assertion/list in the same style as the surrounding `zed` entry, and re-run. (Do not suppress — fix the assertion to include the new target.)

- [ ] **Step 2: Lint the new and changed Markdown**

Run: `npx markdownlint-cli '.reasonix/REASONIX.md' 'docs/architecture/harness-adapter-compliance.md' 'docs/superpowers/**/*.md' --ignore node_modules`
Expected: no lint errors. Fix any reported issues in the files you created/edited.

- [ ] **Step 3: Smoke-test a dry-run plan for the new target**

Run: `node scripts/install-apply.js --target reasonix --profile minimal --dry-run`
Expected: exit 0; output shows `Target: reasonix`, `Adapter: reasonix-project`, an install root ending in `.reasonix`, and planned operations including `.reasonix -> .../.reasonix` and flattened `rules/...` entries.

- [ ] **Step 4: Confirm clean working tree**

Run: `git status`
Expected: working tree clean (all changes committed across Tasks 1–3).

---

## Self-Review Notes

- **Spec coverage:** New target (Task 1) ✓; mirror-zed module set via `targets`-array edit (Task 2 Step 4) ✓; managed `REASONIX.md` at `.reasonix/REASONIX.md` (Task 2 Step 3) ✓; legacy-compat parity (Task 2 Step 7) ✓; help text (Task 2 Step 8) ✓; both schemas (Task 1 Steps 7–8) ✓; compliance matrix (Task 3) ✓; tests (Tasks 1–2) and full suite (Task 4) ✓. Deferred MCP/hooks are recorded in the compliance record's `unsupported_surfaces` (Task 3 Step 2) ✓.
- **No placeholders:** every code/edit step shows literal content and exact commands with expected output.
- **Type/name consistency:** adapter id `reasonix-project` and target `reasonix` are used identically across the adapter, registry, schemas, manifest, tests, and compliance record.
