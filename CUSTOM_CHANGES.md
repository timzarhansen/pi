# Custom pi-agent Changes

This file tracks all local modifications to the pi-agent codebase beyond upstream `upstream/main`.
Use it to guide rebases, merges, and conflict resolution.

---

## Shift+Enter → Newline (Extension-based)

| Field | Value |
|---|---|
| **Date** | 2026-06-16 |
| **Why** | Terminals send different byte sequences for Shift+Enter (JetBrains: `ESC+CR`, VS Code: bare `\n`, Kitty: CSI-u). Without proper handling, these sequences fall through to submit or follow-up instead of inserting a newline. |

### Architecture

**Primary**: Extension at `~/.pi/agent/extensions/shift-enter-fix/index.ts`
- Subclasses `CustomEditor`, overrides `handleInput()`
- Intercepts 6 known Shift+Enter sequences BEFORE any keybinding checks
- Calls `this.addNewLine()` (inherited, `protected`)
- All other input passes through to `super.handleInput(data)`
- Registered via `ctx.ui.setEditorComponent()` on `session_start`

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
