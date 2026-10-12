# 001: Mapping the Pylance VS Code Extension

**Target:** `ms-python.vscode-pylance` v2026.4.1, the extension installed at
`~\.vscode\extensions`, analysed as shipped. No source checkout, no rebuild.
**Tool:** REA v6.4.0, provider `rea-javascript-application` v1, operation
`analyze_javascript_application`, integrity policy `fail`.
**Date:** 12 Oct 2026.

## Why this target

Pylance is the language intelligence behind the Python experience in VS Code.
It ships as a directory of minified bundles plus a local index cache of Python
library stubs. A real, widely used artifact, complex enough to produce a
meaningful graph, small enough to run end to end in one pass.

## Method

```powershell
npx -y rea-agents@latest analyze-javascript-application "C:\Users\vergio\.vscode\extensions\ms-python.vscode-pylance-2026.4.1" --json > rea_pylance.json
```

REA inventories the tree, parses the JavaScript statically and projects two
graphs: an application graph (artifacts, modules, chunks and their relations)
and a semantic relation graph (variables, functions, data flow). Nothing is
executed.

## Inventory and parse

| Measure | Value |
|---|---|
| Files inventoried | 250 |
| JavaScript files parsed | 18 |
| Text bytes read | 46.3 MB |
| AST nodes visited | 5,638,718 |
| Modules identified | 26 |
| Parse failures | 0 |
| Root artifact SHA-256 | `0456a589d3a372bd623864ee71ee8448861df94c2b70564744` |

## Application graph

605 nodes, 709 edges. Node kinds found: `artifact`, `javascript-module`,
`javascript-chunk` and `endpoint`.

Edge relations by count:

| Relation | Count |
|---|---|
| contains | 303 |
| imports | 231 |
| calls | 159 |
| loads | 9 |
| exposes | 5 |
| maps_to | 2 |

An example `calls` edge, quoted from the evidence bundle:

```
source:  dist/extension.bundle.js  line 29, col 45650 -> 45669
authority: static-relationship-inference
confidence: high
limitation: "Static syntax does not prove that this relationship executes at runtime."
```

Every edge in the bundle carries that same shape: the artifact digest, the
exact source range, the extractor identity and version, the confidence level
and the limitation note. The graph does not let you forget how each fact was
arrived at.

## Semantic graph

13,368 nodes and 22,558 relations reconstructed from the bundle internals.

Relation types by count:

| Relation | Count |
|---|---|
| aliases | 12,501 |
| reads | 4,065 |
| captures | 1,783 |
| defines | 1,523 |
| reads-property | 1,317 |
| argument-to-parameter | 426 |
| writes | 300 |
| returns-to-call | 260 |
| calls | 252 |
| writes-property | 123 |
| parses | 3 |
| coerces | 5 |

362 function fingerprints were extracted and 4,885 unknowns were recorded
rather than guessed away. That split between confident relations and explicit
unknowns is the part I find most useful in practice: the graph tells you where
it stops knowing.

## Security surface

REA's Electron security summary for this target is a set of zeroes:

```
operations: 0
literal_channels: 0
dynamic_channel_operations: 0
renderer_transmissions: 0
renderer_listeners: 0
main_handlers: 0
paired_renderer_transmissions: 0
ambiguous_renderer_transmissions: 0
unpaired_literal_renderer_transmissions: 0
```

Zero IPC surface is the correct answer here, and that is the interesting
result. Pylance is not an Electron app, it is a VS Code extension, so it never
touches `ipcMain` or `ipcRenderer`. Its communication goes through the VS Code
extension host API instead. A scanner that flagged this artifact for Electron
IPC risk would be measuring the wrong thing; REA measures the right thing and
reports zero, with one sender-validation observation worth a follow up
inspection.

## What this tells you

The application graph is a map of how the extension is assembled: which
bundles contain which chunks, what imports what, which entry points are
exposed. The semantic graph is a map of what the code does inside those
bundles: 12k alias edges and 4k read edges show a heavily namespaced,
configuration-heavy codebase, which matches what Pylance is (a language server
client with deep per-library indexing logic).

## Limitations

- Static only. No edge is proof of runtime behaviour, and REA says so on
  every inferred relation.
- The shipped index cache (`dist/.cache/local_indices/...`, hundreds of Python
  library stub records) dominates the inventory and inflates raw file counts;
  the 18 parsed JavaScript files are the executable surface.
- The pyright engine internals inside the bundles were surfaced structurally,
  not semantically audited by hand.
- No native analysis engine was involved; this run used only the JavaScript
  provider.

## Reproduce

1. Node 22+, then the command above against your own Pylance install.
2. Output is a canonical Evidence JSON bundle (~32 MB) with an `evidence_id`
   and per-artifact SHA-256 digests.
3. Verify integrity: `npx -y rea-agents@latest evidence-import <bundle>`.

## Evidence reference

- Evidence id: `ev_617e374092f616077e5ced1387ede854eaa70d35b257b3c`
- Application graph id: `jag_6388d5e4341c45bee7f99803c99552062fe2bca49ab967`
- Semantic graph id: `jsrg_6d0bba5cf03e94eab7cbf74c35b8d55db879168d1c07e`

The full JSON is not committed; the digests above make the run reproducible
and verifiable.
