# 004: Native Analysis of findstr.exe with Ghidra

**Target:** `C:\Windows\System32\findstr.exe`, Windows 11 PE x86_64, analysed
as shipped.
**Tool:** REA v6.4.0 composed workflow over the Ghidra provider v12.1.4,
operation `analyze`.
**Date:** 13 Oct 2026.

## Why this target

Follow up to the notepad.exe run, but with a much smaller binary: a stripped
command-line tool with a 24 KB code section. The point is to compare the same
workflow across two Microsoft PE binaries and see how the evidence scales
with the target.

## Method

```powershell
$env:GHIDRA_INSTALL_DIR = "C:\Users\vergio\Dev\tools\ghidra_12.1.4_PUBLIC"
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot"
npx -y rea-agents@latest analyze "C:\Windows\System32\findstr.exe" --json > rea_findstr.json
```

## Result

SHA-256 of the target: `867816f8a714a87f5f573ebbe3a526931fbe787cc03b4ec7ac8a970c9d8c0264`.

| Measure | Value |
|---|---|
| Segments | 10 |
| Procedures | 180 |
| Strings | 161 |

Segment layout:

| Segment | Range | Size |
|---|---|---:|
| Headers | 0x140000000 - 0x140001000 | 4,096 |
| .text | 0x140001000 - 0x140007000 | 24,576 |
| fothk | 0x140007000 - 0x140008000 | 4,096 |
| .rdata | 0x140008000 - 0x14000b000 | 12,288 |
| .data | 0x14000b000 - 0x14008c860 | 530,528 |
| .pdata | 0x14008d000 - 0x14008e000 | 4,096 |
| .didat | 0x14008e000 - 0x14008f000 | 4,096 |
| .rsrc | 0x14008f000 - 0x140090000 | 4,096 |
| .reloc | 0x140090000 - 0x140091000 | 4,096 |
| tdb | 0xff00000000 - 0xff00001850 | 6,224 |

## Observations

Two things stand out against the notepad.exe run:

- **Scale follows the target.** findstr's `.text` is 24 KB against notepad's
  160 KB, and the procedure count drops from 853 to 180. The workflow's
  output sizes track the actual executable surface rather than a fixed
  budget.
- **The 530 KB `.data` section is the interesting anomaly.** A command-line
  grep equivalent has no business carrying half a megabyte of writable data.
  The overview does not explain it, which is exactly the kind of open
  question a per-function pass should chase next (`rea decompile` /
  `rea function` on the routines that touch that range).

## Limitations

- Overview only: procedure and string counts, no per-function decompilation
  in this run.
- Windows Ghidra support remains experimental (`-windows-p0`), with
  no-return repair disabled as documented in post 003.
- The workflow limitation stands as stated in the bundle: *"Derived by an REA
  composed workflow."*

## Evidence reference

- Evidence id: `ev_9e498d2d479986f616b033242b46a96a80412d3afc903c07b167995648d2c2ea`
- Target SHA-256: `867816f8a714a87f5f573ebbe3a526931fbe787cc03b4ec7ac8a970c9d8c0264`
- Ghidra provider: 12.1.4

The full JSON bundle is not committed; digests above make the run
reproducible and verifiable.
