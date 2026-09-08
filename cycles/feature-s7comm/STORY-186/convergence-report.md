---
document_type: per-story-convergence-report
level: ops
version: "1.0"
status: complete
producer: state-manager
timestamp: 2026-09-07T23:49:00Z
phase: step-4.5-per-story-adversarial
inputs: []
input-hash: "[live-state]"
traces_to: STATE.md
story: STORY-186
cycle: feature-s7comm
passes_total: 5
verdict: CONVERGED
criterion: BC-5.39.001
clean_streak: [P3, P5-pending-classification, "see note"]
final_head: "PR #470 merge commit 294174f5"
base: "e0ea30ce"
story_version: "1.1"
---

# Convergence Report — STORY-186 (compact)

## Pipeline Run: 2026-09-07
## Product: wirerust — STORY-186 S7comm ISO-on-TCP Carry-Buffer Reassembly, Walk-First Frame Extraction, Resync, and the Frozen SS-20/SS-21 Module Boundary (wave 89)
## Iterations: 5 (P1, P1b, P2, P3, P5)

---

## Verdict: CONVERGED — BC-5.39.001 SATISFIED (final passes clean/nitpick-only: P3/P5, with P1b as an interstitial re-verification pass)

## Trajectory

`2 MAJOR + 2 MINOR + 3 NIT (P1) → [human ruling Option B + spec reconciliation] → re-verify (P1b) → 1 MINOR (P2, fixed) → CLEAN/NITPICK_ONLY (P3) → CLEAN/NITPICK_ONLY (P5) → CONVERGED`

| Pass | Verdict | MAJOR | MINOR | NIT | Notes |
|------|---------|-------|-------|-----|-------|
| P1 | MAJOR findings | 2 | 2 | 3 | F-01 CHANGELOG gap (MAJOR); F-02 dead-code T0814 overflow guard / BC-2.20.013-vs-BC-2.20.014 spec contradiction (MAJOR) — triggered a mid-story human ruling |
| P1b | re-verification | 0 | 0 | — | Re-verifies the human-ruling-driven spec reconciliation (BC-2.20.013/014 v1.0→v1.1, ADR-0014 note, STORY-186 v1.0→v1.1) before continuing the pass sequence |
| P2 | MINOR (fixed) | 0 | 1 | — | Doc-tense MINOR finding, fixed same pass |
| P3 | CLEAN / NITPICK_ONLY | 0 | 0 | 0 | BC-conformance axis clean |
| P5 | CLEAN / NITPICK_ONLY | 0 | 0 | 0 | Security/evasion + test-fidelity axes clean (pass P4 folded into the same review round as P5 per the dispatch record) |

---

## Headline Narrative

**Red Gate PASSED.** 14 behavioral tests written against `todo!()` stub bodies all failed
(RED) pre-implementation; 2 static module-boundary regression guards (zero `StreamAnalyzer`
impls in `iso_on_tcp.rs`; no `IsoOnTcpFlowState` type anywhere in the tree) were green from the
start, confirming the frozen SS-20/SS-21 boundary was not violated by the stub scaffold itself.

**TDD implementation** added `S7commAnalyzer` (`src/analyzer/s7comm.rs`, SS-21) — the first
effectful-shell consumer of STORY-184/185's stateless TPKT/COTP parsing library (SS-20,
`src/analyzer/iso_on_tcp.rs`): directional carry-buffer TPKT reassembly across TCP segment
boundaries via a walk-first, residual-bound frame-extraction loop (no aggregate
`carry.len() + data.len()` pre-check); a shared 1-byte resync sub-routine reused verbatim for
both bad-version-byte and post-overflow conditions; a 65,535-byte carry bound; and flow-close
teardown (`on_flow_close`) that discards carry bytes with no finding.

**Pass 1 (2 MAJOR + 2 MINOR + 3 NIT):**

- **F-01** (MAJOR) — CHANGELOG `[Unreleased]` entry gap (CLAUDE.md AC-158-001/PG-W71-CHANGELOG
  obligation for a `src/` change). Fixed same burst.
- **F-02** (MAJOR) — the carry-overflow guard (BC-2.20.014, as originally specified) is
  literally unreachable dead code under the walk-first resync design: BC-2.20.013 specifies
  that resync CONSUMES the offending garbage byte-by-byte, so the carry buffer never actually
  accumulates unbounded garbage the way BC-2.20.014's guard assumed. This is a genuine
  BC-2.20.013-vs-BC-2.20.014 spec contradiction, not an implementation bug — the implementer
  had faithfully coded to both (mutually inconsistent) contracts. **Human ruling: Option B
  (defense-in-depth)** — retain the overflow guard as a proven-unreachable-under-current-traffic
  defense-in-depth measure rather than delete it as dead code, since a future change to the
  resync strategy could make the guard reachable again. Drove a same-burst spec reconciliation:
  `BC-2.20.013` v1.0→v1.1, `BC-2.20.014` v1.0→v1.1 (reclassified as defense-in-depth), an
  ADR-0014 note documenting the reconciliation, and `STORY-186` v1.0→v1.1 (overflow ACs
  reframed as defense-in-depth + an AC-186-007 length-typo fix caught in the same pass).
- 2 MINOR + 3 NIT — absorbed into the same remediation burst; no separate commits tracked in
  this compact report.

**Pass 1b:** Re-verification pass confirming the P1 spec-reconciliation + fixes landed
correctly and did not introduce new inconsistencies. 0 findings.

**Pass 2 (1 MINOR, fixed):** A doc-tense finding (RED/GREEN tense precision in a code comment
or test header) — fixed same pass.

**Pass 3 (CLEAN/NITPICK_ONLY):** BC-conformance axis — all STORY-186 ACs verified against the
reconciled `BC-2.20.013`/`BC-2.20.014` v1.1 text. Zero findings.

**Pass 5 (CLEAN/NITPICK_ONLY):** Security/evasion + test-fidelity axes. Zero findings.

**Security verification:** 2 independent security-reviewer passes, 0 CRIT/HIGH findings both
times — APPROVE.

---

## Non-Blocking Residuals (for gate ratification)

Three accepted-residual NITs carried to `cycles/feature-s7comm/lessons.md` (items 5-7):

1. "ADR-014" vs the repo's 4-digit "ADR-0014" naming convention in `s7comm.rs` doc-comments +
   CHANGELOG — cosmetic.
2. `BC-2.20.014`'s stale "OPEN ITEM (2026-09-07)" forward-reference — the ADR-0014
   reconciliation note it requested WAS added (`develop` `294174f5`), but the OPEN ITEM marker
   itself was not cleared.
3. `BC-2.20.014`'s canonical-vector "At-bound, legitimate" row is self-inconsistent (a
   65,535-byte "still incomplete" residual is unrealizable under the `u16` length cap) — the
   test correctly sidesteps this; spec-vector imprecision only.

All three are non-blocking, deliberately deferred to the STORY-187 spec pass or a future
maintenance sweep to avoid a canonical-input-hash rehash cascade on the just-merged wave.

---

## Final Verification Evidence

| Check | Result |
|-------|--------|
| Red Gate (pre-impl) | PASSED — 14/14 behavioral tests RED against `todo!()` stubs; 2/2 static guards GREEN |
| Post-implementation test suite | 18/18 green (new STORY-186 tests) |
| Full CI | 13/13 green |
| pr-reviewer | APPROVE (0 blocking) |
| security-reviewer | CLEAN ×2 independent reviews (0 CRIT/HIGH) |
| Demo evidence | 12/12 ACs (`docs/demo-evidence/STORY-186/` on develop) |
| BC-5.39.001 convergence | CONVERGED across P1(MAJOR)→P1b(re-verify)→P2(MINOR,fixed)→P3(CLEAN)→P5(CLEAN) |
| Story version | v1.1 |
| Merge | PR #470 squash-merged to develop as `294174f5`, 2026-09-07T23:49Z |

---

## Process Gaps Noted

- **F-02 root cause** — see `cycles/feature-s7comm/lessons.md` item 8 / STATE.md Decisions Log
  D-565 / `cycles/feature-s7comm/drift-items-and-carry-forwards.md`
  (`DRIFT-F2-CROSS-BC-CONSISTENCY-CHECK`): the BC-2.20.013-vs-BC-2.20.014 contradiction was
  inherited from F2 spec-evolution and escaped both the F2 consistency audit and F3 story
  decomposition, only surfacing here at per-story implementation review.
- `PG-CANONICAL-HOLDOUT-NOT-AC-ENFORCED` (2 prior occurrences, STORY-184/185) did **NOT**
  recur a 3rd time on STORY-186 — watch closed for this epic pending a future recurrence.

---

## Traceability

- Story: `.factory/stories/STORY-186.md` (v1.1)
- Red Gate log: `.factory/cycles/feature-s7comm/STORY-186/implementation/red-gate-log.md`
- Demo evidence pointer: `.factory/cycles/feature-s7comm/STORY-186/demo-evidence-pointer.md`
- BC-2.20.013 / BC-2.20.014 (v1.1): `.factory/specs/behavioral-contracts/ss-20/`
- BC-INDEX: `.factory/specs/behavioral-contracts/BC-INDEX.md` (v2.38.1)
- Lessons: `.factory/cycles/feature-s7comm/lessons.md` (items 5-8, infra items 1-3)
- Drift items: `.factory/cycles/feature-s7comm/drift-items-and-carry-forwards.md`
- PR: #470, merge commit `294174f5` (2026-09-07T23:49Z)
- Note: this cycle does not maintain a machine-readable per-pass `adversary-convergence-state.json`
  for feature-s7comm F4 stories (unlike the wave-084/085/086 per-story convergence artifacts);
  this compact narrative report is the authoritative pass-by-pass record for STORY-186.
