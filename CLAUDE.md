# Workpads Standard — Claude Code

Normative specification repo. Implementations must conform here; kaios insights flow back via SUI register.

## Start here (mandatory)

1. **[`../workpadskaios/system/project-process.md`](../workpadskaios/system/project-process.md)** — cross-repo bottleneck (authority, sync order, project state).
2. [`README.md`](README.md) — spec index.
3. [`codec.md`](codec.md) — §5 pads-v1 wire format (`#1pa/`).

## When editing the standard

- Register design advances in [`STANDARD-UPDATES.md`](STANDARD-UPDATES.md) (SUI rows).
- Codec changes: update [`codec-sync.md`](codec-sync.md), then `workpads-codec`, then kaios inline codec.
- Mirror deviation summaries in [`implementation-notes.md`](implementation-notes.md); live register: `workpadskaios/system/dev_daily/DEVIATIONS.md`.

## Ecosystem

| Repo | Role |
|------|------|
| `workpads-standard/` (here) | Normative spec |
| `workpads-codec/` | Canonical JS |
| `workpads-cli/` | CLI harness |
| `workpadskaios/` | Primary implementation lab |
