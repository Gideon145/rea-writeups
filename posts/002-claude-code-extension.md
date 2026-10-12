# 002: Mapping the Claude Code VS Code Extension

**Target:** `anthropic.claude-code` v2.1.296, the extension installed at
`~\.vscode\extensions`, analysed as shipped.
**Tool:** REA v6.4.0, `analyze_javascript_application`, integrity policy
`fail`.
**Date:** 12 Oct 2026.

## Why this target

Claude Code is an agentic coding tool, and its VS Code extension is a thin
shell around a bundled CLI. The interesting question is what the shell does:
which parts live in JavaScript, what the webview persists, and what it imports
from the host.

## Method

```powershell
npx -y rea-agents@latest analyze-javascript-application "C:\Users\vergio\.vscode\extensions\anthropic.claude-code-2.1.296-win32-x64" --json > rea_claude.json
```

## Inventory and parse

| Measure | Value |
|---|---|
| Files inventoried | 5 |
| JavaScript files parsed | 2 |
| Text bytes read | 9.3 MB |
| AST nodes visited | 2,262,148 |
| Findings | 581 |
| Parse failures | 0 |
| Root artifact SHA-256 | `f6b9863132a2ee3165c3dccf21054297ba2be01c5082ef7b07e98ae95aba2171` |

Two parsed modules: `extension.js` (the extension host entry point) and
`webview/index.js` (the chat panel). Everything else in the folder is the
platform CLI and resources.

## Application graph

125 nodes, 591 edges.

| Relation | Count |
|---|---|
| imports | 453 |
| exposes | 51 |
| calls | 60 |
| persists_to | 15 |
| contains | 8 |
| loads | 3 |
| maps_to | 1 |

The 453 import edges include unresolved `require` references to
`node:process`, `node:path` and friends, each marked
`confidence: low` with the honest limitation *"The static reference could not
be resolved within the inventoried artifact."* That is correct behaviour: the
runtime is the bundled CLI binary, which sits outside the inventoried
JavaScript surface. The graph records the boundary instead of pretending to
cross it.

## Persistence surface

Every `persists_to` edge is a `localStorage` call in `webview/index.js`,
with exact source ranges, for example:

```
persists_to: webview/index.js line 2247, col 4527
mechanism: call:globalThis.localStorage.setItem
```

Fifteen of them: `setItem`, `getItem` and `removeItem` interleaved. So the
webview keeps its UI state in `localStorage` rather than in the extension
host's workspace state. From a security review angle that is a useful fact on
its own: whatever the chat panel remembers (conversation drafts, UI state) is
webview-scoped browser storage, not `vscode.workspaceState`.

## Security summary

Same as Pylance: every Electron IPC counter is zero. No `ipcMain`, no
`ipcRenderer`, no preload, no context bridge. The extension talks to the
extension host through the VS Code API, and the webview talks to the
extension through VS Code webview message passing, neither of which is raw
Electron IPC. REA measuring zero here is the correct, evidence-backed answer.

## What this tells you

- The extension is a genuine thin client: 2 JavaScript modules, everything
  else shipped as a binary.
- The chat webview manages its own persistence in `localStorage`.
- The import boundary is explicit about what could not be resolved, which is
  how a static tool should behave at the edge of its inventory.

## Limitations

- Static only; no runtime confirmation.
- The bundled CLI binary was out of scope for the JavaScript provider. A
  native pass with Ghidra would be the follow up.
- `webview/index.js` is minified; source ranges are exact but symbol names
  are the minifier's.

## Evidence reference

- Evidence id: `ev_dc6e3bba8bc63bb1463648460bf4440fa3449f2343d1f15a7dc22965e9732219`
- Root artifact SHA-256: `f6b9863132a2ee3165c3dccf21054297ba2be01c5082ef7b07e98ae95aba2171`

The full JSON bundle is not committed; digests above keep the run
reproducible and verifiable.
