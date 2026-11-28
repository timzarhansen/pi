# Custom pi-agent Changes

## Shift+Enter → Newline

`packages/tui/src/components/editor.ts` keeps `addNewLine()` protected for the autosuggestion extension and recognizes JetBrains ESC+LF. `packages/tui/test/editor.test.ts` covers the Shift+Enter input variants. The primary key-sequence handling lives in `$PI_CODING_AGENT_DIR/extensions/autosuggestion/index.ts`, outside this repository.
