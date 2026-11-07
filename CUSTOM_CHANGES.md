# Custom pi-agent Changes

This file tracks all local modifications to the pi-agent codebase beyond upstream `upstream/main`.
Use it to guide rebases, merges, and conflict resolution.

## Pi documentation prompt policy

The system prompt directs Pi-topic work to the relevant API documentation and the smallest applicable example. Consult additional references only when needed to resolve uncertain contracts or directly relevant requirements; do not blanket-read all Markdown files or follow every link. Documentation asset roots are resolved from the package configuration, not assumed relative to the working directory.

**2026-11-06**: the local change had also added a "Relevant starting points:" line duplicating the prompt's existing "When asked about:" mapping. Its ~150 extra tokens pushed upstream's small-window (2600-token) compaction characterization test past the overflow edge (`reason: "overflow"` instead of `"threshold"`; test passes on pure upstream). The duplicate was removed in `bdb4c5bbc`; the "smallest applicable example" scoping line is kept.

---

## Other Local Deltas (undocumented elsewhere)

| File | Change | Notes |
|---|---|---|
| root `package.json` / `package-lock.json` | `@types/semver@7.7.1` added as devDependency | **Removed 2026-10 (v0.99.2 merge)**: resolved to upstream, which removed it from the root. `packages/coding-agent` still declares it. No local delta remains here. |

---

## Shift+Enter → Newline (Extension-based)

| Field | Value |
|---|---|
| **Date** | 2026-06-16 |
| **Why** | Terminals send different byte sequences for Shift+Enter (JetBrains: `ESC+CR`, VS Code: bare `\n`, Kitty: CSI-u). Without proper handling, these sequences fall through to submit or follow-up instead of inserting a newline. |

### Architecture

**Primary**: `$PI_CODING_AGENT_DIR/extensions/autosuggestion/index.ts` (default: `~/.pi/agent/extensions/autosuggestion/index.ts`) — integrated into `GhostEditor` class (extends `CustomEditor`)
- `GhostEditor` overrides `handleInput()`
- Intercepts 6 known Shift+Enter sequences BEFORE any keybinding checks
- Calls `this.addNewLine()` (inherited, `protected`)
- All other input passes through to `super.handleInput(data)`
- Part of the autosuggestion extension (registered on `session_start` like other autosuggestion logic)

Handled sequences:
| Sequence | Source |
|---|---|
| `\n` | VS Code / bare LF |
| `\x1b\r` | JetBrains ESC+CR |
| `\x1b\n` | JetBrains ESC+LF |
| `\x1b[13;2u` | Kitty CSI-u |
| `\x1b[13;2:1u` | Kitty CSI-u (press) |
| `\x1b[13;2~` | xterm function-key |

### Core Changes (preserved)

| File | Change |
|---|---|
| `packages/tui/src/components/editor.ts` | `ESC+LF` (`\x1b\n`) in newline detection; `addNewLine()` is `protected` |
| `packages/tui/test/editor.test.ts` | Tests for all Shift+Enter variants + bare LF |

The core `custom-editor.ts` has NO hardcoded Shift+Enter handling — this was moved to the extension.

### Rebase/Merge Notes

- **Upstream risk**: Minimal. The core `custom-editor.ts` has no local modifications for Shift+Enter.
- **Watch for**: If upstream changes `addNewLine()` from `protected` to `private` in `editor.ts`. If upstream removes `ESC+LF` from editor newline detection.
- **Conflict strategy**: The extension is the primary fix and lives outside the repo. On merge, preserve `protected` on `addNewLine()` and the editor's newline sequence list. No custom-editor.ts conflicts expected.

### Merge Log

| Date | Upstream | Result |
|---|---|---|
| 2026-07-02 | v0.85.0 (589 commits) | Clean merge, no conflicts. Upstream made `addNewLine()` `private` and removed `ESC+LF` — git auto-resolved preserving local `protected` + `\x1b\n` (line 882) + tests. All 2102 tests pass. |
| 2026-09-07 | v0.85.1  (20 commits) | Clean merge, no conflicts. Custom changes intact: `addNewLine()` `protected` (line 1317), `\x1b\n` handling (line 882), Shift+Enter tests. All 2161 tests pass. |
| 2026-09-10 | upstream/main @ `6b94ae2ec` (35 commits) | Clean merge, no conflicts. Custom changes intact: `addNewLine()` `protected` (line 1317), `\x1b\n` handling (line 882), Shift-Enter tests. One upstream test failure: `packages/ai/test/anthropic-adaptive-thinking-models.test.ts` — upstream commit `6b94ae2ec` marks 9 Fireworks Messages models as `forceAdaptiveThinking` (by design), but this older invariant test still filters to Claude/kimi IDs only. Unrelated to local changes; needs an upstream test fix or a local test patch. |
| 2026-10-01 | v0.99.2 (320 commits) | Conflicts only in root `package.json` + `package-lock.json` (`@types/semver` / `@typescript/native-preview`); resolved by taking upstream. Custom changes intact: `addNewLine()` `protected` (line 1328), `\x1b\n` handling (line 894), Shift-Enter tests. Note: upstream now has its own native Shift-Enter support (Kitty CSI-u via `keys.ts`, `\x1b\r`/`\n` when Kitty active), but still lacks `\x1b\n` (JetBrains ESC+LF) and `\x1b[13;2:1u` (Kitty press event) — local delta still needed. Merge commit `beab4dda5`. Full `npm run check` + `./test.sh` pass (all suites, exit 0). |
| 2026-10-01 | v1.0.0 (31 commits) | Clean auto-merge, no conflicts and no lockfile changes this time. Upstream v1.0.0 bumps all packages to 1.0.0 and adds the `durable`, `client`, `server`, `mcp`, `telemetry` packages. Custom changes intact: `addNewLine()` `protected` (line 1328), `\x1b\n` handling (line 894), Shift-Enter tests. `npm install --ignore-scripts` left `package-lock.json` unchanged; `npm run check` exit 0; `./test.sh` exit 0 (5548 tests, 12 suites). `pi-test.sh --version` reports 1.0.0. Merge commit `a5550d4e4`. |
| 2026-10-08 | earendil-works/pi `upstream/main` @ `ce950d78f` (146 commits; v1.0.1–v1.1.0) | Auto-merged without conflicts. Local Shift+Enter changes remain intact: `addNewLine()` is `protected`, ESC+LF remains in newline detection, and the local tests remain. `npm run build:offline`, `npm run check`, and the TUI editor test (200 tests) pass. `npm ci --ignore-scripts` warned that Gondolin requires Node >=23.6 (bundled Node is 22.19) and reported 2 high-severity audit findings. |
| 2026-11-06 | earendil-works/pi `upstream/main` @ `d448c51e2` (197 commits; v1.5.0 →) | Conflicts: `nix/model-catalog.json` (took upstream — the local pin was the same routine pin-update chore, two days stale) and `packages/ai/CHANGELOG.md` + `packages/codemode/CHANGELOG.md` (union of entries; local codemode entry preserved). Pre-commit `tsc` failed until `npm run generate:models` was run — provider `data/*.json` files are gitignored and generated; the new Qwen Token Plan Individual CN provider needs regeneration after any merge. Lockfile verified byte-identical to upstream before the `PI_ALLOW_LOCKFILE_CHANGE` override. Customs intact: `addNewLine()` `protected` (line 1339), `\x1b\n` (line 905), Shift+Enter tests, codemode prelude, docs-policy prompt line. Residual surface drift vs upstream: the `github-copilot.ts` type cast (harmless, type-checks). Merge commit `71265a514`. Post-merge findings: (1) the duplicated Pi-docs prompt line (local) broke the compaction characterization test — see the policy section above; fixed in `bdb4c5bbc`. (2) 34 `pi-env` failures were a stale Rust daemon binary (machine had no cargo) — installed Rust 1.99, `npm run build:daemon` in `packages/env`, env suite then 116/0. (3) Two chord failures (files summary tie, watcher `set`) are full-suite-load flakes — both pass 4/4 in isolation. Final `./test.sh`: ~22,000 passed, 1 failed (known chord watcher flake), build + check green. `pi-test.sh --version` reports 1.5.0. |
