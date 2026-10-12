# 003: Native Analysis of notepad.exe with Ghidra

**Target:** `C:\Windows\System32\notepad.exe`, Windows 11 PE x86_64, analysed
as shipped.
**Tool:** REA v6.4.0 composed workflow over the Ghidra provider v12.1.4,
operation `analyze`.
**Date:** 12 Oct 2026.

## Why this target

First native run: a signed Microsoft binary on the machine where it ships.
The point is to exercise the whole native path (Ghidra headless import,
decompiler probe, segment and procedure recovery) and to show what the
evidence looks like when the target is not source we control.

## Setup

```powershell
winget install EclipseAdoptium.Temurin.21.JDK
# downloaded ghidra_12.1.4_PUBLIC_20260921.zip from the NSA release and extracted it
$env:GHIDRA_INSTALL_DIR = "C:\Users\vergio\Dev\tools\ghidra_12.1.4_PUBLIC"
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot"
npx -y rea-agents@latest analyze "C:\Windows\System32\notepad.exe" --json > rea_notepad.json
```

`rea doctor` reports every Ghidra sub-check healthy: installation, version,
headless analyzer, and the native decompiler probe (`decompile.exe` reporting
`pe x86_64` against `win32/x64`).

## Result

The workflow identified a PE x86_64 executable with a SHA-256 of
`468ffe129c395abf6b21a09efdf261910a95fb98aa982ead73caa7b2b684577e`, and
recovered:

| Measure | Value |
|---|---|
| Segments | 10 |
| Procedures | 853 |
| Strings | 638 |

Segment layout:

| Segment | Range | Size |
|---|---|---:|
| Headers | 0x140000000 - 0x140001000 | 4,096 |
| .text | 0x140001000 - 0x140028000 | 159,744 |
| fothk | 0x140028000 - 0x140029000 | 4,096 |
| .rdata | 0x140029000 - 0x140034000 | 45,056 |
| .data | 0x140034000 - 0x140036740 | 10,048 |
| .pdata | 0x140037000 - 0x140039000 | 8,192 |
| .didat | 0x140039000 - 0x14003a000 | 4,096 |
| .rsrc | 0x14003a000 - 0x140059000 | 126,976 |
| .reloc | 0x140059000 - 0x14005a000 | 4,096 |
| tdb | 0xff00000000 - 0xff00001850 | 6,224 |

## What the profile says about how it was analysed

The analysis profile is explicit about its own limits, which is worth
quoting:

- `import_mode: ephemeral-source-immutable` - the import is a throwaway,
  read-only project, not a persistent modification of the target
- `analyzer_preset: ghidra-default`
- `no_return_repair: disabled-windows-p0` - on Windows the no-return repair
  pass is deliberately off, which is the documented P0 behaviour for the
  experimental Windows Ghidra support

The target itself was never modified and never executed. Ghidra imported the
bytes read-only.

## Limitations

- The result is an overview: segment map plus procedure and string counts.
  Per-function decompilation and reference recovery are separate REA
  operations and were not run for this post.
- Windows Ghidra support is experimental (`-windows-p0`), and no-return
  repair is disabled as above, so some control flow recovery is knowingly
  weaker than on Linux.
- The workflow limitation is stated in the bundle itself: *"Derived by an REA
  composed workflow."*

## Evidence reference

- Evidence id: `ev_e8cfde73452a816bb69e88e3ea04c346c36aef2830e95b77a4318d5b4f1f9867`
- Target SHA-256: `468ffe129c395abf6b21a09efdf261910a95fb98aa982ead73caa7b2b684577e`
- Ghidra provider: 12.1.4

The full JSON bundle is not committed; digests above make the run
reproducible and verifiable.
