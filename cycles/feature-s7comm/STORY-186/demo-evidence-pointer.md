---
document_type: demo-evidence-pointer
level: ops
version: "1.0"
status: final
producer: state-manager
timestamp: 2026-09-07T23:49:00Z
traces_to: "STORY-186"
inputs: []
input-hash: "d41d8cd"
---

# Demo Evidence Pointer — STORY-186

Demo evidence for STORY-186 lives on the `develop` branch (not in the `.factory` worktree) at:

```
docs/demo-evidence/STORY-186/
```

Verified present at `develop` commit `294174f5` (PR #470 merge commit). Coverage: **12/12
acceptance criteria**, recorded as paired `.tape`/`.gif`/`.webm` VHS terminal recordings:

| Evidence File Group | AC Coverage |
|---------------------|-------------|
| `AC-001-003-carry-reassembly.{tape,gif,webm}` | AC-186-001..003 — carry-buffer reassembly across TCP segment boundaries |
| `AC-004-006-defense-in-depth.{tape,gif,webm}` | AC-186-004..006 — carry-overflow defense-in-depth guard (post BC-2.20.014 v1.1 reconciliation) |
| `AC-007-009-resync.{tape,gif,webm}` | AC-186-007..009 — 1-byte resync (bad-version-byte + post-overflow) |
| `AC-010-011-module-boundary.{tape,gif,webm}` | AC-186-010..011 — frozen SS-20/SS-21 module boundary static guards |
| `AC-012-flow-close.{tape,gif,webm}` | AC-186-012 — `on_flow_close` carry-byte teardown |
| `AC-ALL-18-green.{tape,gif,webm}` | Full 18/18 STORY-186 test suite green, post-implementation |
| `VP-050-proptests.{tape,gif,...}` | VP-050 proptest evidence |

Scrub gate: per `CLAUDE.md` / `.factory/maintenance/demo-evidence-scrub-gate.md`, this evidence
set is subject to the standard path-scrub gate before any further redistribution; it was already
run as part of the STORY-186 PR #470 delivery pipeline prior to merge.

This pointer file exists so the delivery close-out artifact set under
`cycles/feature-s7comm/STORY-186/` is self-contained without duplicating binary evidence into
the `.factory` worktree (which does not carry `docs/demo-evidence/` — that path lives only on
`develop`).
