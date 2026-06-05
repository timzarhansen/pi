# Custom pi-agent Changes

This file tracks all local modifications to the pi-agent codebase beyond upstream `upstream/main`.
Use it to guide rebases, merges, and conflict resolution.

---

## JetBrains Shift+Enter Fix

| Field | Value |
|---|---|
| **Commit** | `c1543690789ca28cf03216e36522d341b02113f1` |
| **Date** | 2026-06-02 |
| **Why** | JetBrains IDEs 2025.3.3+ send `ESC+CR` (`\x1b\r`) or `ESC+LF` (`\x1b\n`) for Shift+Enter. Without Kitty keyboard protocol, the keybinding system interprets `\x1b\r` as Alt+Enter, triggering `app.message.followUp` instead of inserting a newline. |

### Files Changed

| File | Change |
|---|---|
| `packages/coding-agent/src/modes/interactive/components/custom-editor.ts` | Detect `ESC+CR`, `ESC+LF`, `CSI-u [13;2u`, `CSI-u [13;2:1u` at top of `handleInput()` before keybinding checks → calls `addNewLine()` |
| `packages/tui/src/components/editor.ts` | Added `ESC+LF` (`\x1b\n`) to newline detection; changed `addNewLine()` from `private` to `protected` |
| `packages/tui/test/editor.test.ts` | 8 new tests covering all JetBrains Shift+Enter variants |

### Rebase/Merge Notes

- **Upstream risk**: Low. The upstream has not touched these files since this commit.
- **Watch for**: If upstream modifies `handleInput()` in `custom-editor.ts` or `addNewLine()` visibility/signature in `editor.ts`, merge conflicts are likely.
- **Conflict strategy**: Apply the JetBrains escape-sequence detection block before any upstream-added keybinding checks. Preserve `protected` modifier on `addNewLine()`.
