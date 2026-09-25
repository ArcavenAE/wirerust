---
document_type: per-story-convergence-report
level: ops
version: "1.0"
status: complete
producer: state-manager
timestamp: 2026-09-25T00:00:00Z
phase: step-4.5-per-story-adversarial
inputs: []
input-hash: "[live-state]"
traces_to: STATE.md
story: STORY-187
cycle: feature-s7comm
passes_total: 24
verdict: CONVERGED
criterion: BC-5.39.001
clean_streak: [P22, P23, P24]
final_head: "a44e8277 (feature/STORY-187-s7comm-header-dispatch, worktree .worktrees/STORY-187 — NOT YET MERGED)"
base: "develop 47951b7a"
story_version: "1.17"
---

# Convergence Report — STORY-187 (compact)

## Pipeline Run: 2026-09-25 (checkpoint burst D-569)
## Product: wirerust — STORY-187 S7comm Flow State Completion, Four-Way protocol_id Dispatch Skeleton, and parse_s7comm_header Pure-Core Parser (wave 90)
## Iterations: 24

---

## Verdict: CONVERGED — BC-5.39.001 SATISFIED, closed per HUMAN RULING (2026-09-25): accept convergence with residuals after one final fix batch

## Trajectory (compact)

STORY-187 required 24 per-story adversarial passes to converge — substantially more than
STORY-184 (10), STORY-185 (5), or STORY-186 (5) — driven by the story's larger diff (~5k lines
across three TDD rounds plus mid-story scope changes from human rulings on Ack_Data header
length and other classification questions). This is a compact report: full per-pass finding
text for passes not called out below is not itemized here — significant/ruling-driving passes
and the closing trio are recorded; the remaining passes' findings were addressed inline
(fixes or accepted-residual dispositions) as part of the same convergence loop.

Earlier **clean** passes (0 blocking findings): **P10, P11, P15, P17, P18, P21, P24.**

| Pass | Verdict | Notes |
|------|---------|-------|
| P1–P9 | mixed findings | Findings addressed via fixes and/or the human rulings below (F-01, F-02, F-12, F-13, F-14, Ack/Ack_Data 12-byte-header ruling, F-40); not separately itemized in this compact report |
| P10 | CLEAN | — |
| P11 | CLEAN | — |
| P12 | findings | P12 architectural observation — LI=0 DT/resync artefact can sticky-classify a flow Unclassified; logged as `D-P12-1` (drift item, needs DF-VALIDATION-001 research validation) |
| P13–P14 | mixed findings | Sibling-sweep-miss recurrence observed (see lessons.md item (a): F-31/F-33/P13-F-1) |
| P15 | CLEAN | Residual: P15-F-1 (accepted, non-blocking) |
| P16 | findings | — |
| P17 | CLEAN | Residual: P17-F-2 (accepted, non-blocking) |
| P18 | CLEAN | — |
| P19 | findings | — |
| P20 | findings | AC-notes-vs-test-body verification gap observed (lessons.md item (b): P20-F-1); residuals P20-F-2/F-3 partially addressed |
| P21 | CLEAN | CPU-amplification observation — quadratic carry re-copy in the pre-existing STORY-186 `on_data` carry path; logged as `D-P21-1`. Residual: P21-F-1 (accepted, non-blocking) |
| P22 | LOW/NIT | wording-only findings |
| P23 | LOW (fixed) + NITs | one LOW test-adequacy finding, fixed post-convergence with mutation proof (`cargo-mutants` evidence-arithmetic survivors killed by `6705ed8b`); remaining NITs accepted as residuals |
| P24 | NITPICK_ONLY | Closing pass of the clean trio (P22/P23/P24) |

---

## Headline Narrative

**Red Gate PASSED across 3 TDD rounds** (see `cycles/feature-s7comm/STORY-187/implementation/
red-gate-log.md`): Round 1 (stubs `3be2730a`; tests `c0029757`, 20 RED + 5 passing-by-design,
20 pre-existing `story_186` green; implementation to 45/45), Round 2 (tests `36265c0b`/
`b4a86c92`, 7 RED; implementation `409c0d5d` to 59/59), Round 4 (Ack_Data 12-byte-header
ruling; tests `6861cf2a`, 6 RED; implementation `b97464cf` to 68/68). Final `s7comm_analyzer_
tests` **83/83** green; full suite **2,803 passed / 0 failed**; clippy/fmt/doc-tense/
changelog-gate all clean.

**Formal + mutation verification:** Kani VP-051 parse harness executed locally — VERIFICATION
SUCCESSFUL (0/264, 6/6 covers). `cargo-mutants` 27.1.0 serial over the STORY-187 `s7comm.rs`
diff: 43 mutants — all viable non-equivalent killed (3 late survivors in evidence arithmetic
killed by `6705ed8b`), 1 equivalent (empty `Some(0x72)` placeholder arm, deferred to
STORY-190), 2 unviable.

**Closing trio (P22/P23/P24):** P22 surfaced only LOW/NIT wording findings. P23 surfaced one
LOW test-adequacy finding — fixed post-convergence with mutation-testing proof — plus NITs. P24
was NITPICK_ONLY. Per **HUMAN RULING (2026-09-25)**: convergence accepted with residuals after
the one final fix batch (the P23 LOW fix); BC-5.39.001's 3-clean-pass criterion is satisfied by
this trio's disposition (P22 LOW/NIT non-blocking, P23 LOW fixed same-burst + NITs non-blocking,
P24 NITPICK_ONLY) rather than requiring three literally-zero-finding passes back to back — the
human ruling is the closure authority for this story per the standing per-story convergence
protocol.

---

## Human Rulings (2026-09-24/25)

- **F-01** — opposite-direction CR/CC session (dispatch/classification ruling for the
  connection-request/connection-confirm session-direction case).
- **F-02** — `None` `DT` (data-transfer without an established session/protocol_id) never
  classifies (ruling that an unclassifiable DT frame yields no classification, not an error).
- **F-12** — classic S7comm dissection is gated on a *sticky* Classic-variant flag once a flow
  has been observed to be Classic.
- **F-13** — Ack error-logging behavior deferred to STORY-188 (not in STORY-187 scope).
- **F-14** — VP-051 Kani harness is P0 priority.
- **Ack and Ack_Data both 12-byte headers** — ruling establishing both ROSCTR variants share a
  12-byte header length, confirmed via the canonical-frame holdout (permitted per ADR-014
  Decision 4, see F-40 below).
- **F-40** — ADR-014 Decision 4 note: public wire-capture bytes are permitted as test vectors
  only (not as implementation-derivation source); provenance for the Ack `0x02` ROSCTR value
  established via permitted design references (icsnpp-s7comm, python-snap7; libs7comm
  consistent with the same value).
- **Convergence closure** — human ruling (2026-09-25) accepting convergence with residuals
  after the P23 fix batch (see Closing trio above).

---

## Non-Blocking Residuals (accepted, carried to lessons.md)

- **P15-F-1** — accepted residual (non-blocking).
- **P17-F-2** — accepted residual (non-blocking).
- **P20-F-2 / P20-F-3** — partially addressed; residual scope carried forward.
- **P21-F-1** — accepted residual (non-blocking).
- P22/P24 wording-only NITs — accepted residuals (see lessons.md item (d) on convergence
  velocity for high-line-count diffs).

Full text and disposition: `cycles/feature-s7comm/lessons.md` (this burst's new items).

---

## Final Verification Evidence

| Check | Result |
|-------|--------|
| Red Gate (3 rounds) | PASSED — see red-gate-log.md |
| `s7comm_analyzer_tests` (final) | 83/83 green |
| Full suite | 2,803 passed / 0 failed |
| clippy / fmt / doc-tense / changelog-gate | clean |
| Kani VP-051 (local) | VERIFICATION SUCCESSFUL (0/264, 6/6 covers) |
| `cargo-mutants` (STORY-187 diff) | 43 mutants: all viable non-equivalent killed, 1 equivalent (deferred STORY-190), 2 unviable |
| BC-5.39.001 convergence | CONVERGED across 24 passes; closing trio P22 (LOW/NIT)→P23 (LOW fixed + NITs)→P24 (NITPICK_ONLY); human-ruled closure 2026-09-25 |
| Story version | v1.17 |
| Merge | **NOT YET MERGED** — `feature/STORY-187-s7comm-header-dispatch`, worktree `.worktrees/STORY-187`, HEAD `a44e8277`, 52 commits over `develop` `47951b7a`; demo recording in progress; next: human-executed merge per `PG-MERGE-CLASSIFIER-F4` |

---

## Traceability

- Story: `.factory/stories/STORY-187.md` (v1.17)
- Red Gate log: `.factory/cycles/feature-s7comm/STORY-187/implementation/red-gate-log.md`
- Machine-readable state: `.factory/cycles/feature-s7comm/STORY-187/adversary-convergence-state.json`
- BC-2.21.001..009 (v-bumped this session): `.factory/specs/behavioral-contracts/ss-21/`
- BC-INDEX: `.factory/specs/behavioral-contracts/BC-INDEX.md` (v2.38.18)
- VP-INDEX: `.factory/specs/verification-properties/VP-INDEX.md` (v2.54) — VP-051 Kani harness
- Lessons: `.factory/cycles/feature-s7comm/lessons.md` (this burst's new items)
- Drift items / carry-forwards: `.factory/cycles/feature-s7comm/drift-items-and-carry-forwards.md`
- Branch: `feature/STORY-187-s7comm-header-dispatch`, worktree `.worktrees/STORY-187`, HEAD `a44e8277` (NOT YET MERGED)
- Note: per the STORY-186 precedent, this narrative report (plus the accompanying
  `adversary-convergence-state.json`) is the authoritative pass-by-pass record for STORY-187;
  full per-pass finding text for passes not called out above was not separately preserved in
  this compact-report format.
