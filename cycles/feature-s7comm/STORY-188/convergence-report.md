---
document_type: per-story-convergence-report
level: ops
version: "1.0"
status: complete
producer: state-manager
timestamp: 2026-10-05T00:00:00Z
phase: step-4.5-per-story-adversarial
inputs: []
input-hash: "[live-state]"
traces_to: STATE.md
story: STORY-188
cycle: feature-s7comm
passes_total: 7
verdict: CONVERGED
criterion: BC-5.39.001
clean_streak: [P5, P6, P7]
final_head: "c6bd91e3 (feature/STORY-188-s7comm-function-code-classification, worktree .worktrees/STORY-188 — NOT YET MERGED, no PR)"
base: "develop 17b00031"
story_version: "1.7"
---

# Convergence Report — STORY-188

## Pipeline Run: 2026-10-04..05 (D-572..D-578)
## Product: wirerust — STORY-188 S7comm Job/Ack_Data Function-Code Classification (wave 91)
## Iterations: 7

---

## Verdict: CONVERGED — BC-5.39.001 SATISFIED (3 consecutive clean-class passes, last NITPICK_ONLY)

Closing trio: P5 NITPICK_ONLY, P6 CLEAN, P7 NITPICK_ONLY. `passes_clean` [5, 6, 7],
`last_classification` NITPICK_ONLY. Closure authority: BC-5.39.001 criterion met (no human
closure ruling needed, unlike STORY-187). No logic defects found since pass 2.

## Pass Table

| Pass | Classification | MAJOR | MINOR | NIT | Branch at review | Remediation commits |
|------|----------------|-------|-------|-----|------------------|---------------------|
| P1 | HAS_FINDINGS | 1 | 7 | 3 | pre-`9e2d34cd` | `9e2d34cd`, `ca78afda`, `f33b4337`, `1835c5b4` |
| P2 | HAS_FINDINGS | 0 | 6 | 4 | `1835c5b4` | `7fcf3a5b`, `e24c6f7d` |
| P3 | HAS_FINDINGS | 1 | 3 | 3 | `e24c6f7d` | `861c7f0b`, `cc8dbf22` (+ whole-BC-set sweep, 12 extra defects) |
| P4 | HAS_FINDINGS | 0 | 5 | 2 | `cc8dbf22` | `7b9c5ab1`, `539645f1` (+ 4 extra BC frontmatter YAML defects) |
| P5 | NITPICK_ONLY | 0 | 0 | 4 | `539645f1` | `360df9b5`, `731b3ec8` |
| P6 | CLEAN | 0 | 0 | 0 | `731b3ec8` | none |
| P7 | NITPICK_ONLY | 0 | 0 | 4 | `731b3ec8` | `436912f5`, `c6bd91e3` + spec edits (BC-2.21.008 v1.13, BC-INDEX v2.38.24) |

Trajectory: P1 1/7/3 -> P2 0/6/4 -> P3 1/3/3 -> P4 0/5/2 -> P5 0/0/4 -> P6 0/0/0 -> P7 0/0/4.
Every pass was a fresh-context review; P1-P5 had Phase A attestation valid. P6 and P7 were
independent full reviews (P7 given no prior-findings list).

## Findings per Pass and Dispositions

### P1 (1 MAJOR / 7 MINOR / 3 NIT) — all dispositioned
- F-01 MAJOR no canonical-frame tests (DF-CANONICAL-FRAME-HOLDOUT-001): FIXED. Canonical vectors
  in `research/s7comm-canonical-fc-vectors.md`; AC-188-011 added; test renamed to BC-2.21.016.
- F-02 Write Var length boundary unpinned: FIXED.
- F-03 classification result discarded: RESOLVED-BY-DECISION (orchestrator; classification-only
  placeholder consumed by STORY-191/192).
- F-04 stale stderr test comment: FIXED. F-05 stale story/BC text: FIXED (BC re-anchored).
- F-06 bounds-gated recording unspecified: FIXED (gated on BC-2.21.009 bounds pass; orchestrator).
- F-07 first-N record loses later errors: FIXED per human ruling R2.
- F-08 Kani harness mislabeled VP-052: FIXED (relabeled VP-051 third harness).
- F-09 Unrecognized(0xFF) shared placeholder: ACCEPTED-RESIDUAL. F-10, F-11 NITs: FIXED.

### P2 (0 / 6 / 4) — 9 fixed, 1 resolved-by-decision
P2-F-01..F-06 MINOR (BC-2.21.016 EC-001 contradiction, BC-2.21.017 disproved claim, VP-052
mis-anchors, stale VP-INDEX refs, Userdata non-recording untested [new test], story BC titles
not verbatim): FIXED. P2-F-07..F-09 NIT: FIXED. P2-F-10 BC-2.21.009 traceability omits
STORY-188: RESOLVED-BY-DECISION (orchestrator: cross-reference only, not added to `bcs`).

### P3 (1 / 3 / 3) — all fixed; all 10 P2 findings VERIFIED-FIXED
P3-F-02 MAJOR BC-2.21.013/014 denied VP while VP-INDEX VP-054 lists them (sibling miss of
P2-F-04): FIXED. P3-F-01, F-03, F-04 MINOR (anchors omit Userdata test, false "no service-string"
claim, Kani harness overclaim) and P3-F-05..F-07 NIT: FIXED. Orchestrator-mandated whole-BC-set
reverse-check sweep of BC-2.21.008..017 found and fixed 12 additional defects.

### P4 (0 / 5 / 2) — all fixed; all P3 findings VERIFIED-FIXED except STORY EC-012
P4-F-01 STORY EC-012 false Kani-coverage claim + untested defensive `NoParameterBlock` path (new
test `test_BC_2_21_017_unsliceable_parameter_block_returns_no_parameter_block`), P4-F-02 stale
VP-051 harness doc, P4-F-03 BC-2.21.009 invalid YAML frontmatter, P4-F-04 BC-2.21.008 EC test
citations mismatched, P4-F-05 research doc "10-byte Job/Ack header", P4-F-06/F-07 NIT: FIXED.
Product-owner full ss-21 frontmatter parse found 4 more invalid frontmatters
(BC-2.21.010/013/015/016): FIXED, 41/41 parse. No logic defects.

### P5 (0 / 0 / 4) — NITPICK_ONLY, first clean-class pass
P5-F-01 BC-2.21.017 anchors, P5-F-02 AC<->test traceability, P5-F-03 fixture EC citation,
P5-F-04 130-col doc line: all FIXED. All 7 P4 findings VERIFIED-FIXED.

### P6 (0 / 0 / 0) — CLEAN
Fresh context, branch `731b3ec8`. All P5 NIT fixes verified. No findings.

### P7 (0 / 0 / 4) — NITPICK_ONLY
Independent full review, no prior-findings list, branch `731b3ec8`.
- P7-F-01 fixture e2e test overclaims AC trace: FIXED (`436912f5`; now AC-188-001..008 + 010,
  explicitly not 009/011).
- P7-F-02 BC-2.21.008 stale "9 test functions" count: FIXED (BC-2.21.008 v1.13 "9 in
  `mod story_187`"; BC-INDEX v2.38.24).
- P7-F-03 `on_data` doc omits new behavior: FIXED (`c6bd91e3`; `s7comm.rs` stays 1223 lines).
- P7-F-04 decoder leniency: `decode_write_var_area` does not check item count >= 1 or var-spec
  `0x12 0x0A`; `decode_plc_control_service` does not check `0xFD`. Spec-conformant per
  BC-2.21.012 PC3. CARRY-FORWARD to STORY-191/192 as `STORY-191-192-DECODER-LENIENCY`
  (`drift-items-and-carry-forwards.md`). Per DF-VALIDATION-001 it must be research-validated
  before any GitHub issue is filed; NO issue filed.

## Human Rulings (2026-10-04)

- **R1** — AC-188-010 / BC-2.21.008 PC4 surface is an analyzer-side bounded record
  (`S7AckErrorObservation` list, cap `MAX_S7_ACK_ERROR_OBSERVATIONS`=1024, saturating dropped
  count), NOT stderr (ADR-0004 flooding rationale).
- **R2** — per adversary F-07: add an exact per-(rosctr, error_class, error_code) count map
  (`S7AckErrorKey`, <=131,072 keys) plus `pdu_reference` on each observation.

## Orchestrator Decisions

- **F-03** — classification result kept as documented classification-only placeholder.
- **F-06** — Ack/Ack_Data recording gated on BC-2.21.009 bounds pass.
- **F-09** — `Unrecognized(0xFF)` shared placeholder accepted as residual (NIT).
- **P2-F-10** — BC-2.21.009 cross-reference only; NOT added to STORY-188 `bcs`.
- **Whole-BC-set sweep (P3)** — mandated reverse-check sweep of BC-2.21.008..017.

## Carry-Forwards

- `STORY-191-192-DECODER-LENIENCY` (P7-F-04) — new; needs research validation (DF-VALIDATION-001)
  before any issue; no issue filed.
- PRF-005 (Kani assertion re-proving `Vec::get`) — RESOLVED in the STORY-188 branch by the VP-051
  third harness; pending merge.
- F-09 `Unrecognized(0xFF)` shared placeholder — accepted residual.

## Process-Gaps (open lesson candidates)

- **Lesson 16 (sibling-sweep recurrence)** — sibling-sweep misses recurred in P2 and P3
  (lessons.md item 16). Remains an open lesson candidate.
- **Lesson 17 (no automated BC YAML frontmatter validation)** — 5 invalid BC frontmatters found
  only by a manual parse in P4 (lessons.md item 17). Remains an open lesson candidate.
- **Cycle-closing obligation (orchestrator rule S-7.02):** each of lessons 16 and 17 needs either
  a follow-up story or a justified deferral at feature-s7comm cycle close. No stories created now.

## Final Verification Evidence

| Check | Result |
|-------|--------|
| Full suite | 2837 passed / 0 failed |
| clippy / fmt / doc-tense | clean |
| Kani `verify_classify_job_ack_function_param_slicing_safe` (VP-051) | VERIFICATION SUCCESSFUL (src logic unchanged since) |
| BC-5.39.001 | CONVERGED, 7 passes, closing trio P5/P6/P7 |
| Spec versions | BC-2.21.008 v1.13, BC-2.21.009 v1.12, BC-2.21.010..016 v1.4 (011/012/014 v1.3), BC-2.21.017 v1.5, BC-INDEX v2.38.24, VP-INDEX v2.55, STORY-188 v1.7 |
| Input-hash (canonical) | STORY-188 ba2851e->d9a1797; STORY-187 ebe2614->308d737; STORY-194 f3ea734 unchanged; STORY-184..194 11/11 MATCH; background-stale 22 unchanged |
| Merge | NOT YET MERGED — branch HEAD `c6bd91e3`, no PR. Next: demo-recorder -> push -> pr-manager |

## Traceability

- Story: `.factory/stories/STORY-188.md` (v1.7)
- Machine-readable state: `.factory/cycles/feature-s7comm/STORY-188/adversary-convergence-state.json`
- Red Gate log: `.factory/cycles/feature-s7comm/STORY-188/implementation/red-gate-log.md`
- BCs: `.factory/specs/behavioral-contracts/ss-21/` (BC-2.21.008..017)
