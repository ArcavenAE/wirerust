---
document_type: lessons-learned
level: ops
version: "1.0"
status: in-progress
producer: state-manager
timestamp: 2026-09-07T02:15:00Z
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

6. **[accepted-residual] BC-2.20.014 stale "OPEN ITEM (2026-09-07)" forward-reference** —
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

## Policy Candidates

| Lesson | Proposed Policy | Scope | Status |
|--------|----------------|-------|--------|
| 1 | Extend `bin/check-green-doc-tense` TIER-1 patterns with the "MUST FAIL" / "until the STORY-NNN implementer delivers" phrase shapes | Doc-tense gate coverage | proposed |
| 2 | AC-level enforcement of `DF-CANONICAL-FRAME-HOLDOUT-001` (Red Gate or Step-4.5 entry check for a canonical-frame holdout test on parser stories) — 2 occurrences (STORY-184, STORY-185); did NOT recur on STORY-186 — 3x codification threshold not triggered | Story-template / gate discipline | proposed — watch closed for this epic pending a future recurrence |
| 4 | Root-cause or document the Claude Code permission classifier's intermittent blocking of agent-dispatched `gh pr merge` on F4 story PRs (PG-MERGE-CLASSIFIER-F4) | Merge-authorization tooling | deferred — human workaround in place for rest of F4 (STORY-186 merge again human-executed) |
| 8 | Add an F2/F3 cross-BC consistency checkpoint that diffs paired/coupled BCs within a subsystem for state-model contradictions (not just per-BC self-consistency) — motivated by the BC-2.20.013-vs-2.20.014 contradiction escaping to STORY-186 per-story review | F2 spec-evolution / F3 story-decomposition gate discipline | proposed — see DRIFT-F2-CROSS-BC-CONSISTENCY-CHECK |
