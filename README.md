# REA Writeups

Reverse engineering case studies produced with
[REA](https://github.com/morluto/rea) (Reverse Engineer Anything).

Each post follows the same discipline: a target, a method, the evidence REA
produced, and the honest limitations. Every analysis runs locally and every
claim is backed by an artifact digest.

## Posts

1. [001: Pylance VS Code extension](posts/001-pylance-extension.md) — what a
   static JavaScript application graph of a real, shipped extension looks like,
   from IPC observations down to per-edge source ranges.

## Method

REA runs from the terminal with no cloud dependency:

    npx -y rea-agents@latest analyze-javascript-application "<target path>" --json

Native targets (binaries, APKs, firmware) additionally use Hopper, Ghidra or
IDA where available. See `posts/` for per-target commands.

## Why evidence first

Every graph edge REA returns carries a SHA-256 digest of the artifact it came
from, the source range, the extractor version, a confidence level and a
limitation note. The point of these writeups is to show that reasoning, not
just results.
