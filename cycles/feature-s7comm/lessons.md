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

## Infrastructure-Level

_(none recorded this cycle)_

## Policy Candidates

| Lesson | Proposed Policy | Scope | Status |
|--------|----------------|-------|--------|
| 1 | Extend `bin/check-green-doc-tense` TIER-1 patterns with the "MUST FAIL" / "until the STORY-NNN implementer delivers" phrase shapes | Doc-tense gate coverage | proposed |
| 2 | AC-level enforcement of `DF-CANONICAL-FRAME-HOLDOUT-001` (Red Gate or Step-4.5 entry check for a canonical-frame holdout test on parser stories) — now 2 occurrences (STORY-184, STORY-185), nearing 3x codification threshold | Story-template / gate discipline | proposed — watch for STORY-186 recurrence |
| 4 | Root-cause or document the Claude Code permission classifier's intermittent blocking of agent-dispatched `gh pr merge` on F4 story PRs (PG-MERGE-CLASSIFIER-F4) | Merge-authorization tooling | deferred — human workaround in place for rest of F4 |
