---
document_type: lessons-learned
level: ops
version: "1.3"
status: in-progress
producer: state-manager
timestamp: 2026-09-25T00:00:00Z
cycle: "feature-s7comm"
inputs: [STATE.md]
input-hash: "[live-state]"
traces_to: STATE.md
---

# Lessons Learned — feature-s7comm

<!-- Durable lessons from this cycle for future VSDD factory runs.
     Organized by category: agent-level, process-level, infrastructure-level.
     Each lesson is numbered continuously and includes the pass/burst
     where it was discovered. -->

## Agent-Level

_(none recorded this cycle)_

## Process-Level

1. **[process-gap] PG-CHECK-GREEN-DOC-TENSE-BLINDSPOT** — `bin/check-green-doc-tense` missed
   the "Every test … MUST FAIL" and "until the STORY-184 implementer delivers" RED-tense
   phrasings during STORY-184's per-story review (F-184-P1-003). These phrasings are
   semantically RED (describing not-yet-implemented behavior) but did not match any of the
   linter's existing TIER-1 patterns. This is a gate-coverage gap, not a one-off miss —
   the same phrase shapes are plausible in any future story's test-header prose. Candidate
   disposition: a self-improvement story adding these phrase shapes as new TIER-1 patterns,
   or a documented, justified deferral if the phrase class is judged too narrow to warrant a
   dedicated pattern. Per DF-VALIDATION-001, any GitHub issue filed from this finding requires
   research-agent validation first.
   _Discovered: STORY-184 per-story adversarial pass 1, 2026-09-06/07._

2. **[process-gap] PG-CANONICAL-HOLDOUT-NOT-AC-ENFORCED** — `DF-CANONICAL-FRAME-HOLDOUT-001`
   (the policy requiring an RFC-independent canonical-frame holdout test per parser story) is
   not enforced by any acceptance-criterion-level automated check. As a direct consequence,
   three defects escaped early detection during STORY-184: the missing RFC-independent holdout
   test itself, a §5/§6 RFC-section citation error, and the min-7-vs-4 minimum-length
   divergence (see `DEFERRED-BC-2.20.005-STALE-LEN4` in STATE.md Active Carry-Forwards) — none
   were caught until deep in per-story adversarial review, well after the story's initial TDD
   implementation. Candidate disposition: add an AC-level enforcement mechanism (a checklist
   item, a lint rule, or a story-template gate) that fails a story's Red Gate or Step-4.5 entry
   if no canonical-frame holdout test is present for a story that implements a wire-format
   parser. Per DF-VALIDATION-001, any GitHub issue filed from this finding requires
   research-agent validation first.
   _Discovered: STORY-184 per-story adversarial review (mid-story RFC-min-7 rework), 2026-09-06/07._
   **Recurrence #2 (STORY-185, 2026-09-07):** the RFC/ISO canonical-frame holdout tests for the
   COTP TPDU-type parser exist and are correct, but again no acceptance criterion explicitly
   cites or requires them — same gate-coverage gap, second occurrence. This is nearing the 3×
   codification threshold: flagged for a self-improvement follow-up (AC-level enforcement
   mechanism per the candidate disposition above) if it recurs a third time on STORY-186.

3. **[accepted-residual] STORY-185 regression-guard-comment overstatement** — a code comment
   introduced during STORY-185's implementation claimed broader regression coverage than the
   guard it annotates actually provides (a documentation-only overstatement, not a functional
   gap). Flagged as a NIT during pr-review and accepted as a non-blocking residual — the comment
   text is imprecise but the underlying regression guard itself is correct and sufficient.
   No code change made; recorded here so a future doc-pass can tighten the comment wording.
   _Discovered: STORY-185 PR #467 review, 2026-09-07._

4. **[process-gap] PG-MERGE-CLASSIFIER-F4** — the Claude Code permission classifier blocks or
   hangs on agent-dispatched `gh pr merge` for F4 story PRs. Concrete evidence across this
   cycle: PR #465 blocked, PR #467 (STORY-185) blocked; PR #466 (STORY-184) slipped through
   without issue, so the failure is intermittent, not universal. Disposition: **not** filed as
   a defect this cycle — the human has elected an operating-arrangement workaround (run each
   F4 story merge manually at the wave boundary for the rest of F4, STORY-186..194) rather than
   root-causing the classifier behavior. Recorded in STATE.md Active Carry-Forwards
   (`PG-MERGE-CLASSIFIER-F4`) as a standing arrangement so future bursts expect it. If a root
   cause is later pursued, per DF-VALIDATION-001 any GitHub issue filed from this finding
   requires research-agent validation first.
   _Discovered: STORY-184/185 PR merge attempts, 2026-09-06/07._

5. **[accepted-residual] ADR-014 vs ADR-0014 naming-convention NIT** — `s7comm.rs` doc-comments
   and the STORY-186 CHANGELOG entry refer to the architecture decision as "ADR-014", while the
   repo's established convention (per `CLAUDE.md`'s ADR table and the filename
   `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md`) is the 4-digit form
   "ADR-0014". Cosmetic only — no functional or traceability impact, the numeric ID is
   unambiguous either way. Non-blocking; accepted as a residual for a future doc-pass or the
   STORY-187 spec-review checkpoint rather than a same-burst fix.
   _Discovered: STORY-186 PR #470 review, 2026-09-07._

6. **[accepted-residual] BC-2.20.014 stale "OPEN ITEM (2026-09-07)" forward-reference — RESOLVED (D-567, 2026-09-24).** —
   `BC-2.20.014`'s v1.1 text still carries an "OPEN ITEM (2026-09-07)" marker requesting the
   ADR-0014 reconciliation note that documents the defense-in-depth reclassification. That note
   **WAS** added (`docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md`,
   landed on `develop` as part of `294174f5`) but the OPEN ITEM marker on `BC-2.20.014` itself
   was never cleared to reflect that the request is satisfied. Non-blocking (the substantive
   content is correct and complete); marked for resolution at the STORY-187 spec pass or a
   future maintenance sweep — not fixed this burst, since editing `BC-2.20.014` now would
   trigger a canonical input-hash rehash cascade across every story/BC that cites it, on top of
   the wave just merged.
   _Discovered: STORY-186 PR #470 review, 2026-09-07._
   **Resolution (D-567, 2026-09-24, pre-STORY-187 spec pass):** the OPEN ITEM marker was cleared
   as part of `BC-2.20.014` v1.1→v1.2 (product-owner, consistency-validator-confirmed clean after
   one MAJOR fixed). `STORY-186` rehashed fresh (`259af26`) in the same burst's full E-23 rehash
   sweep; `--scan` confirms MATCH. The anticipated rehash cascade landed together with the
   substantive fix rather than being deferred further.

7. **[accepted-residual] BC-2.20.014 canonical-vector "At-bound, legitimate" row self-inconsistency**
   — `BC-2.20.014`'s canonical test-vector table includes an "At-bound, legitimate" row
   describing a 65,535-byte carry buffer that is "still incomplete" (awaiting more bytes to
   complete a frame). Under the `u16` length-field cap, a TPKT frame's maximum total length is
   65,535 bytes, so a 65,535-byte carry residual that is simultaneously "still incomplete" is
   unrealizable — the vector as written describes a state the type system cannot produce. The
   implementation and its tests correctly sidestep this by not attempting to construct the
   unrealizable case; the defect is in the spec's illustrative vector text, not in behavior.
   Non-blocking; flagged for a spec-vector precision pass alongside item 6 above (same BC file,
   same deferred-edit rationale — avoid a rehash cascade on the just-merged wave).
   _Discovered: STORY-186 adversarial review, 2026-09-07._
   **Resolution (D-567, 2026-09-24, pre-STORY-187 spec pass):** the unrealizable 65,535-byte
   "still incomplete" residual in `BC-2.20.014`'s canonical vector was corrected to the
   realizable 65,534-byte bound (EC-001), with EC-001/EC-002 differentiated and a new EC-006
   added for the synthetic literal-65,535 case; VP table rows 1-3 marked RESOLVED and folded
   into VP-050 (no new VP needed). `BC-2.20.014` v1.1→v1.2. Fresh-context consistency-validator
   audit (3 passes, final clean after one MAJOR fixed) confirmed the correction is internally
   consistent and fully propagated across `VP-INDEX.md`, `verification-architecture.md`,
   `verification-coverage-matrix.md`, `ARCH-INDEX.md`, and `specs/prd.md`.

8. **[process-gap] Inherited BC-2.20.013-vs-2.20.014 spec contradiction escaped F2/F3 review** —
   `BC-2.20.013` (walk-first resync CONSUMES garbage bytes on a bad version byte) and
   `BC-2.20.014` (the carry-overflow guard, as originally specified, assumed garbage
   ACCUMULATES rather than being consumed by the walk-first resync) were mutually inconsistent
   from their original F2 authoring — the contradiction produced a literal dead-code path
   (finding F-02, STORY-186 Pass 1) requiring a mid-story human ruling (Option B,
   defense-in-depth) and a same-burst `BC-2.20.013`/`BC-2.20.014` v1.0→v1.1 reconciliation. This
   is the **second** coupled-BC contradiction in this epic to escape early detection (see lesson
   2's `PG-CANONICAL-HOLDOUT-NOT-AC-ENFORCED`, a related-but-distinct gap) and was not caught
   during F2 spec-evolution's fresh-context consistency audit (D-558) nor F3 story
   decomposition (D-560/D-561) — only surfacing at STORY-186's per-story implementation
   adversarial review, well after both BCs had been authored, reviewed, and cited as story
   inputs. Candidate disposition: extend the F2 consistency-audit protocol (or add a dedicated
   F2/F3 cross-BC consistency checkpoint) to specifically diff paired/coupled BCs within a
   subsystem for state-model contradictions (one BC's precondition assumption invalidated by a
   sibling BC's guarantee), not just internal self-consistency per BC. Recorded as a new open
   Drift Item (`DRIFT-F2-CROSS-BC-CONSISTENCY-CHECK`) in
   `cycles/feature-s7comm/drift-items-and-carry-forwards.md`. Per DF-VALIDATION-001, any GitHub
   issue filed from this finding requires research-agent validation first.
   _Discovered: STORY-186 per-story adversarial pass 1, 2026-09-07._

9. **[process-gap] BC reclassification did not propagate to VP-INDEX/verification-architecture/
   verification-coverage-matrix/PRD** — the 2026-09-07 `BC-2.20.013`/`BC-2.20.014` v1.0→v1.1
   defense-in-depth reconciliation (item 8 above, `DRIFT-F2-CROSS-BC-CONSISTENCY-CHECK`) updated
   the two BC files but did not propagate to the four downstream verification/spec artifacts that
   cite the affected VPs (`VP-INDEX.md`, `verification-architecture.md`,
   `verification-coverage-matrix.md`, `specs/prd.md` narrative + RTM bullets for VP-050/VP-055) —
   they continued to describe the pre-reconciliation scope for over two weeks, undetected until a
   fresh-context consistency-validator audit during the D-567 pre-STORY-187 spec pass (2026-09-24)
   caught the staleness while reconciling `BC-2.20.014` v1.1→v1.2. This is the **second**
   occurrence of the `DRIFT-F2-CROSS-BC-CONSISTENCY-CHECK` failure class in this epic — same root
   cause (a BC-level spec edit not triggering a mandatory downstream-artifact sweep), one hop
   further downstream (VP/arch/PRD artifacts rather than a sibling BC). Per the Cycle-Closing
   Checklist, a recurring process-gap needs a follow-up story or a justified deferral; disposition
   here is a **deferral** — tracked against the existing `DRIFT-F2-CROSS-BC-CONSISTENCY-CHECK`
   drift item (same root cause, no new drift item opened) rather than a dedicated follow-up story,
   because the STORY-187 spec pass already found and fixed this occurrence in the same burst that
   discovered it. Target: fold into the future F2 process improvement (extend the consistency-audit
   protocol to sweep VP-INDEX/verification-architecture/verification-coverage-matrix/PRD whenever
   a BC's VP allocation or scope changes, not just sibling BCs) at the feature-s7comm cycle close.
   Per DF-VALIDATION-001, any GitHub issue filed from this finding requires research-agent
   validation first.
   _Discovered: D-567 pre-STORY-187 spec pass (fresh-context consistency-validator audit), 2026-09-24._

10. **[accepted-residual] FIX-STORY186-ATBOUND-RELABEL PR #473 deferred NITs F3/F5/F6** —
    three NIT-severity findings from the fix PR's cycle-1 pr-review were deferred by agreement
    rather than fixed same-burst: **F3** — the new `test_BC_2_20_014_live_near_bound_residual_
    single_call` test does not re-check the overflow flag after delivering the final byte, and
    the new AC-186-004(b) tests carry no S2C-direction coverage (only C2S exercised); **F5** —
    commit `34b721a0` was typed `fix:` when its content was documentation-only and should have
    been `docs:`; **F6** — the `on_data` overflow-check doc comment carries redundant phrasing
    left over from the earlier wording. None block correctness; F1 MINOR and F2/F4 NIT from the
    same review cycle were fixed in `fd253c3c`. Non-blocking; flagged for a future test-coverage
    pass (F3) or maintenance sweep (F5/F6) rather than a second fix-PR cycle.
    _Discovered: FIX-STORY186-ATBOUND-RELABEL PR #473 review cycle 1, 2026-09-24._

11. **[process-gap] (a) Sibling-sweep misses across passes** — BC anchor test counts were not
    re-swept across sibling BCs after remediation test additions, recurring 3 times within
    STORY-187's 24-pass per-story adversarial loop: **F-31**, **F-33**, **P13-F-1**. Each
    occurrence is the same shape — a test-count anchor in one BC/sibling artifact goes stale
    when a remediation burst adds tests to a related BC without sweeping the anchor counts of
    its siblings. Disposition: **deferred** — tracked against a new drift item
    `DRIFT-P187-SIBLING-TESTCOUNT-SWEEP` (`cycles/feature-s7comm/drift-items-and-carry-
    forwards.md`) rather than `DRIFT-F2-CROSS-BC-CONSISTENCY-CHECK`, since the recurrence
    pattern here (anchor counts specifically, within-pass rather than cross-artifact) is a
    narrower, more mechanically-checkable class than that drift item's general cross-BC
    state-model contradiction scope. Per DF-VALIDATION-001, any GitHub issue filed from this
    finding requires research-agent validation first.
    _Discovered: STORY-187 per-story adversarial passes 13/31/33-equivalent, 2026-09-24/25._

12. **[process-gap] (b) AC notes not verified against test bodies** — acceptance-criterion
    notes claiming specific test assertions were present were written without re-reading the
    actual test bodies to confirm the claim, surfacing 3 times in STORY-187's convergence loop:
    **P16-F-1**, **P19-F-2**, **P20-F-1**. Disposition: **deferred** — tracked against a new
    drift item `DRIFT-P187-AC-NOTE-TEST-VERIFICATION`
    (`cycles/feature-s7comm/drift-items-and-carry-forwards.md`); candidate fix is an AC-note
    authoring checklist step requiring the author to re-read the cited test body before writing
    an assertion claim. Per DF-VALIDATION-001, any GitHub issue filed from this finding
    requires research-agent validation first.
    _Discovered: STORY-187 per-story adversarial passes 16/19/20, 2026-09-24/25._

13. **[process-gap] (c) Canonical-frame holdout surfaced a real BC error — policy validated.**
    The canonical-frame holdout (`DF-CANONICAL-FRAME-HOLDOUT-001`) caught a genuine
    specification error during STORY-187's convergence: the Ack/Ack_Data header-length
    question (resolved by the human ruling that both are 12-byte headers, see convergence-
    report.md) was surfaced by the holdout test disagreeing with the then-current spec text,
    not by a reviewer's manual reading. This is a positive result validating the holdout
    policy's value (see lesson 2/`PG-CANONICAL-HOLDOUT-NOT-AC-ENFORCED`'s enforcement-gap
    concern from earlier stories) — no follow-up action needed beyond noting the validation;
    not a process gap in the usual sense, recorded in this numbered sequence per the
    orchestrator's grouping.
    _Discovered: STORY-187 per-story adversarial convergence (Ack/Ack_Data ruling), 2026-09-24/25._

14. **[process-gap] (d) 24-pass convergence driven by wording-only findings on a large diff.**
    STORY-187's per-story adversarial loop took 24 passes to converge — the highest of any F4
    story so far (STORY-184: 10, STORY-185: 5, STORY-186: 5) — largely because each
    fresh-context reviewer surfaced new doc-wording issues on a ~5k-line diff rather than new
    substantive defects (see the closing trio P22 LOW/NIT-wording, P23 one LOW test-adequacy
    fixed + NITs, P24 NITPICK_ONLY). Candidate dispositions: a wording-only severity floor
    (wording-only findings do not reset the clean-pass streak), or a dedicated convergence rule
    for doc-only drift. **Human ruling (2026-09-25)** closed this story's loop pragmatically
    (accept-with-residuals after the P23 fix batch) without resolving the general process
    question. Disposition: **deferred** — tracked against a new drift item
    `DRIFT-P187-WORDING-CONVERGENCE-VELOCITY`
    (`cycles/feature-s7comm/drift-items-and-carry-forwards.md`); target a future
    BC-5.39.001 convergence-protocol improvement.
    _Discovered: STORY-187 per-story adversarial convergence, closing trio P22/P23/P24,
    2026-09-25._

15. **[accepted-residual] STORY-187 non-blocking residuals carried from the convergence loop**
    — **P15-F-1**, **P17-F-2** (accepted, non-blocking); **P20-F-2**/**P20-F-3** (partially
    addressed, residual scope carried forward); **P21-F-1** (accepted, non-blocking); the P22
    and P24 wording-only NITs (accepted, non-blocking, see lesson 14 above for the pattern).
    None block correctness or the human-ruled convergence closure. Full pass-level context:
    `cycles/feature-s7comm/STORY-187/convergence-report.md`.
    _Discovered: STORY-187 per-story adversarial convergence, 2026-09-24/25._

## Infrastructure-Level

1. **[infra] Nested-subagent messaging deadlock** — pr-manager (dispatched as a subagent for
   STORY-186's PR lifecycle) could not receive its **own** grandchild sub-agents' (security-
   reviewer, pr-reviewer, github-ops CI-check) `SendMessage` replies — those replies routed to
   the parent (orchestrator) session instead of back to pr-manager. Caused a ~40-minute stall
   and a BLOCKED escalation; the orchestrator took over the PR tail directly to unblock
   delivery. Mitigation identified: reviewer/github-ops results should return via an awaited
   Agent-tool dispatch (to the actual dispatcher) rather than a teammate-plus-`SendMessage`
   pattern when the dispatcher is itself a subagent (not the top-level session). Already filed
   via SendFeedback — no further factory-side action needed this cycle.
   _Discovered: STORY-186 PR #470 delivery, 2026-09-07._

2. **[infra] Idle-notification echo storm** — completed roster teammates (`sec-review`,
   `gh-verify-ci`) repeatedly re-woke the orchestrator with duplicate idle notifications after
   completing their work, until an explicit `TaskStop` silenced them. Harness/runtime
   notification-plumbing issue, not a factory logic defect. Already filed via SendFeedback — no
   further factory-side action needed this cycle.
   _Discovered: STORY-186 PR #470 delivery, 2026-09-07._

3. **[infra] adversarial-review skill fork yielded before dispatched adversary passes reported**
   — the `adversarial-review` skill's fork returned control before its own dispatched adversary
   passes had actually reported back; one adversary teammate (`adv-p1`) wedged for
   approximately 15 hours and had to be `TaskStopped` and re-dispatched to make progress.
   Harness/runtime fork-lifecycle issue. Already filed via SendFeedback — no further
   factory-side action needed this cycle.
   _Discovered: STORY-186 per-story adversarial review dispatch, 2026-09-07._

4. **[infra] demo-recorder dispatch stall on stream watchdog** — a demo-recorder dispatch during
   the FIX-STORY186-ATBOUND-RELABEL delivery stalled for 600s against the stream watchdog with
   no files written (no partial `.tape`/screenshot output on disk). Succeeded on retry once the
   dispatch was given a prebuild step and bounded render timeouts. Harness/runtime dispatch-
   timeout issue, not a factory logic defect; recorded here as a mitigation note for future
   demo-recorder dispatches (prebuild + bounded render timeouts) rather than filed as a
   standalone defect this cycle.
   _Discovered: FIX-STORY186-ATBOUND-RELABEL demo-evidence re-render, 2026-09-24._

5. **[process-gap] (e) Nested demo-recorder/VHS stall recurrence** — the demo-recorder
   dispatch stall pattern from infra item 4 (stream-watchdog timeout, no partial output on
   disk) recurred during STORY-187's demo recording (in progress as of this checkpoint burst).
   Mitigation already identified in item 4 (prebuild step + bounded render timeouts) was
   applied and the recurrence was successfully mitigated. Harness/runtime dispatch-timeout
   issue, not a factory logic defect; no further factory-side action needed — recorded here to
   confirm the item-4 mitigation generalizes across stories.
   _Discovered: STORY-187 demo-recorder dispatch, 2026-09-24/25._

## Policy Candidates

| Lesson | Proposed Policy | Scope | Status |
|--------|----------------|-------|--------|
| 1 | Extend `bin/check-green-doc-tense` TIER-1 patterns with the "MUST FAIL" / "until the STORY-NNN implementer delivers" phrase shapes | Doc-tense gate coverage | proposed |
| 2 | AC-level enforcement of `DF-CANONICAL-FRAME-HOLDOUT-001` (Red Gate or Step-4.5 entry check for a canonical-frame holdout test on parser stories) — 2 occurrences (STORY-184, STORY-185); did NOT recur on STORY-186 — 3x codification threshold not triggered | Story-template / gate discipline | proposed — watch closed for this epic pending a future recurrence |
| 4 | Root-cause or document the Claude Code permission classifier's intermittent blocking of agent-dispatched `gh pr merge` on F4 story PRs (PG-MERGE-CLASSIFIER-F4) | Merge-authorization tooling | deferred — human workaround in place for rest of F4 (STORY-186 merge again human-executed) |
| 8 | Add an F2/F3 cross-BC consistency checkpoint that diffs paired/coupled BCs within a subsystem for state-model contradictions (not just per-BC self-consistency) — motivated by the BC-2.20.013-vs-2.20.014 contradiction escaping to STORY-186 per-story review | F2 spec-evolution / F3 story-decomposition gate discipline | proposed — see DRIFT-F2-CROSS-BC-CONSISTENCY-CHECK; recurred D-567 one hop downstream (VP/arch/PRD), see lesson 9 |
| 9 | Extend the same F2/F3 consistency checkpoint (lesson 8's proposed policy) to sweep VP-INDEX/verification-architecture/verification-coverage-matrix/PRD whenever a BC's VP allocation or scope changes, not just sibling BCs — motivated by the D-567 finding that the D-565 BC-2.20.013/014 reconciliation itself did not propagate to those four artifacts | F2 spec-evolution gate discipline | deferred (D-567) — tracked against DRIFT-F2-CROSS-BC-CONSISTENCY-CHECK (same root cause, no new drift item); target feature-s7comm cycle close |
| 11 | Add a mechanical sibling-BC test-count-anchor re-sweep step to remediation bursts (narrower than lesson 8/9's general cross-BC checkpoint) — motivated by the F-31/F-33/P13-F-1 recurrence in STORY-187's convergence loop | Remediation-burst discipline / story-writer-test-writer checklist | proposed — see DRIFT-P187-SIBLING-TESTCOUNT-SWEEP |
| 12 | Require AC-note authors to re-read the cited test body before writing an assertion claim (an authoring checklist step) — motivated by the P16-F-1/P19-F-2/P20-F-1 recurrence in STORY-187's convergence loop | Story-writer / AC-note authoring discipline | proposed — see DRIFT-P187-AC-NOTE-TEST-VERIFICATION |
| 14 | Introduce a wording-only severity floor (or a dedicated convergence rule for doc-only drift) so that wording-only findings do not reset the per-story adversarial clean-pass streak — motivated by STORY-187 needing 24 passes to converge, largely on doc-wording findings on a ~5k-line diff | BC-5.39.001 convergence-protocol discipline | proposed — see DRIFT-P187-WORDING-CONVERGENCE-VELOCITY; human ruling (2026-09-25) closed STORY-187 pragmatically without resolving the general question |
