# Hockney — Final PR Review for v0.8.21 Release

> Date: 2026-03-07
> Reviewer: Hockney (Tester)
> Requested by: Brady
> Context: Final assessment of 2 open PRs before v0.8.21 ships

---

## PR #189 — feat: Squad Workstreams — horizontal scaling via Codespaces

**Author:** tamirdresher (community) | **Base:** main | **State:** open, dirty (merge conflicts)
**Size:** 26 files, +1,867 / -14, 9 commits

### Files Changed

| Category | Files | Lines |
|----------|-------|-------|
| SDK (streams module) | 6 new files in `packages/squad-sdk/src/streams/` | +299 |
| CLI (workstreams command) | 1 new, 1 modified in `packages/squad-cli/` | +192 |
| Config (init support) | 1 modified | +35 |
| Tests | 1 new (`test/streams.test.ts`) | +592 |
| Docs | 5 new (blog, features, scenarios, PRD) | +709 |
| Acceptance tests | 4 modified (version bumps) | +6/-6 |
| Package versions | 4 modified | +4/-4 |
| Coordinator template | 1 modified | +13 |

### CI Status

- **Check runs:** None. No CI pipeline ran on this PR.
- **Mergeable state:** `dirty` — merge conflicts against `main`.

### Review Status

- **Copilot PR reviewer:** 9 automated review threads (3 responded to by author, 6 still unresolved)
- **Human approvals:** Zero.

### Code Quality Assessment

**Positives:**
- Clean module structure — types, resolver, filter, barrel export pattern matches existing SDK layout
- Uses `spawnSync` with args array (not `execSync` with string interpolation) — follows project norms
- Strict config validation with per-entry shape checking and invalid-entry filtering
- Backward compatibility — `streams` alias preserved, repos without workstreams.json unaffected
- Good type definitions with JSDoc
- Resolution order (env → file → auto-select → null) is well-documented and tested

**Issues found:**

1. **`process.exit(1)` in streams.ts** — Lines 42 and 48 of `packages/squad-cli/src/cli/commands/streams.ts` call `process.exit(1)` directly. This violates team decision: *"Only `cli-entry.ts` may call `process.exit()`. Library functions throw `SquadError`."* Should throw and let cli-entry handle it.

2. **Version bump conflicts** — PR bumps to `0.8.21-preview.3`, but `main` is at `0.8.21-preview.9`. Will conflict on merge.

3. **No CLI command tests** — Copilot reviewer flagged this: `runWorkstreams` subcommands (list/status/activate) have no tests. Only SDK-level config/resolver tests exist in `test/streams.test.ts`. The CLI surface — output formatting, error paths, `.squad-workstream` file creation via activate — is untested.

4. **6 unresolved review threads** — Including docstring accuracy (resolver comment claims config-based activation that doesn't exist), missing CLI flag wiring for `--stream` in init, and `filterIssuesByStream` described as "used by Ralph" but never wired into triage.

5. **`.squad/.first-run` in gitignore** — Added but not referenced or explained anywhere in the PR.

### Test Coverage

- **44 tests** in `test/streams.test.ts` — covers config loading, stream resolution, issue filtering, init integration
- Good: edge cases for malformed config, missing files, case-insensitive filtering
- Gap: No CLI command invocation tests (streams list/status/activate subcommands)
- Gap: No tests for backward-compat `streams` alias routing

### Risk Assessment

- **Blast radius:** Low for existing functionality. New module is additive with clean barrel exports. Repos without `workstreams.json` are unaffected.
- **What could break:** The `process.exit()` calls in streams.ts could cause problems if the workstreams module is ever imported in library context. Version conflicts will definitely block merge.
- **Dependency risk:** None — no new dependencies added.

### Base Branch

Both `main` and `dev` branches exist. This PR targets `main`. Per project conventions, feature work should land on `dev` first.

### Recommendation

**Hold for v0.8.22 — rebase to `dev`**

**Rationale:** Solid feature architecture and good SDK-level tests, but merge conflicts, zero CI runs, no human approval, `process.exit()` violations, and missing CLI command tests make this not ship-ready for v0.8.21. Rebase to `dev`, fix the process.exit() calls, add CLI-level tests, resolve remaining review threads, then merge for v0.8.22.

---

## PR #191 — feat: Azure DevOps platform adapter — Squad for enterprise

**Author:** tamirdresher (community) | **Base:** main | **State:** open, dirty (merge conflicts)
**Size:** 28 files, +2,732 / -45, 14 commits

### Files Changed

| Category | Files | Lines |
|----------|-------|-------|
| SDK (platform module) | 7 new files in `packages/squad-sdk/src/platform/` | +1,212 |
| CLI (help text, routing) | 1 modified | +38/-18 |
| Config (init support) | 1 modified | +66/-12 |
| Tests | 1 new (`test/platform-adapter.test.ts`) | +746 |
| Docs | 3 new (blog, features, PRD) | +472 |
| Coordinator template | 2 modified | +172/-2 |
| Acceptance tests | 4 modified (version bumps) | +6/-6 |
| Package versions | 3 modified | +3/-3 |
| Consult test | 1 modified | +2/-2 |

### CI Status

- **Check runs:** None. No CI pipeline ran on this PR.
- **Mergeable state:** `dirty` — merge conflicts against `main`.

### Review Status

- **Formal reviews:** Zero. No reviewer approvals.
- **Community review:** @wiisaacs performed a rigorous 5-model code review with test validation (4 comments). Two rounds of critical findings, author addressed both rounds.
- **Author responses:** All critical/high findings addressed in follow-up commits (shell injection → execFileSync, WIQL escaping, JSON.parse error handling, token exposure mitigation).

### Code Quality Assessment

**Positives:**
- Clean adapter interface design — `PlatformAdapter` with consistent `listWorkItems`, `createPR`, `mergePR`, `addTag` API
- Factory pattern with auto-detection from git remote URL — elegant UX
- GitHub adapter uses `execFileSync('gh', [...args])` consistently — safe from shell injection
- ADO adapter has `escapeWiql()` helper for query-language injection protection
- `parseJson()` helper with raw output in error messages — good DX
- ADO remote URL parsing covers 4 formats (HTTPS, SSH, user-prefix, legacy visualstudio.com)
- Planner adapter is a creative addition for enterprise teams using Microsoft 365

**Issues found:**

1. **`execSync` still present** — `azure-devops.ts` imports `execSync` and uses it in `assertAzCliAvailable()` (line 20: `execSync('az devops -h', ...)`). `detect.ts` uses `execSync` for `git remote get-url origin`. While these don't interpolate user input, the import of `execSync` in `azure-devops.ts` is unnecessary — `execFileSync('az', ['devops', '-h'], ...)` is safer and consistent with the rest of the adapter.

2. **Planner task ID hash collisions** — Community review confirmed 1 collision in 100K synthetic IDs using the simple hash function. More critically, the hash is non-reversible — once `listWorkItems` returns a numeric ID, you can't call `PATCH /planner/tasks/{originalStringId}` because the original Planner ID is lost. The `url` field workaround helps but is fragile.

3. **PlannerAdapter incomplete interface** — Missing `createPR`, `mergePR`, `createBranch` methods that `PlatformAdapter` likely requires. Community review flagged this as a runtime crash risk if used polymorphically. Consider splitting into `WorkItemAdapter` + `RepoAdapter`.

4. **Bearer token in process args** — Mitigated via `curl --config -` (stdin) per author's latest fix, but `curl` shelling is still unnecessary. Node's native `fetch` or `undici` would eliminate the process-arg exposure entirely and be more portable.

5. **Planner PATCH missing If-Match/ETag** — Per Graph API docs, Planner updates require concurrency headers. Acknowledged by author as deferred.

6. **N+1 queries in ADO listWorkItems** — 1 WIQL query + N individual `az boards work-item show` calls. Performance concern for large backlogs.

7. **Hardcoded "User Story" work item type** — Fails for Scrum ("PBI") and Basic ("Issue") ADO process templates.

8. **Version bumps conflict** — PR bumps to `0.8.21-preview.7`, main is at `0.8.21-preview.9`.

### Test Coverage

- **57 tests** in `test/platform-adapter.test.ts`
- Strong: Platform detection from URL (10 tests), GitHub remote parsing (8 tests), ADO remote parsing (8 tests)
- Strong: Ralph command generation, Planner task mapping, type coverage
- Gap: **No integration tests for adapter command construction** — tests validate parsing/detection but never mock `execFileSync` to verify command assembly
- Gap: No tests for `createPlatformAdapter` factory function
- Gap: No tests for `escapeWiql` with adversarial inputs (despite this being a critical security fix)

### Risk Assessment

- **Blast radius:** Medium. Modifies CLI help text, coordinator template, and init flow — all shared surfaces. New platform module is additive but the init changes could affect existing `squad init` behavior.
- **What could break:** `init.ts` changes (+66/-12) modify the initialization flow — needs careful regression testing. Coordinator template changes affect all agent sessions.
- **Security residual:** Planner adapter's hash collision and missing concurrency headers could cause data corruption in production use. WIQL escaping was added but has no test coverage.
- **Dependency risk:** Requires `az` CLI and optionally `curl` — runtime dependencies not declared or detected.

### Base Branch

Targets `main`. Should target `dev`.

### Recommendation

**Hold for v0.8.22 — rebase to `dev`**

**Rationale:** Architecturally strong adapter pattern with good community review process (the 5-model review with test validation was exceptional). However, merge conflicts, zero CI, no human approval, incomplete Planner adapter, untested security fixes (escapeWiql), and missing integration tests put this below the bar for v0.8.21. Rebase to `dev`, add escapeWiql tests, add adapter command construction tests, address the execSync remnants, then merge for v0.8.22.

---

## Summary

| PR | Recommendation | Key Blockers |
|----|---------------|--------------|
| #189 Workstreams | **Hold for v0.8.22** | Merge conflicts, no CI, `process.exit()` violation, no CLI tests, no human approval |
| #191 ADO Adapter | **Hold for v0.8.22** | Merge conflicts, no CI, incomplete Planner adapter, untested security fixes, no human approval |

**Neither PR should merge for v0.8.21.** Both are solid features with good architecture, but neither has passed CI, received human approval, or resolved merge conflicts. Ship v0.8.21 clean, then bring these in on `dev` for v0.8.22.

— Hockney
