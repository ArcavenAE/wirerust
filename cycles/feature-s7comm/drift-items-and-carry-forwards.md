---
document_type: open-tracking-items
level: ops
version: "1.0"
status: living
producer: state-manager
timestamp: 2026-09-07T05:02:36Z
cycle: "feature-s7comm"
traces_to: STATE.md
---

# Open Drift Items & Active Carry-Forwards

Extracted from STATE.md's `## Drift Items` and `## Active Carry-Forwards` sections on
2026-09-07 (compact-state, per orchestrator directive to keep STATE.md to frontmatter /
Project Metadata / Phase Progress / Current Phase Steps / Decisions Log / Skip Log /
Blocking Issues / Session Resume Checkpoint only). These are all still OPEN/ACTIVE items
(resolved/cleared items were separately archived to `cycles/history/open-items-archive.md`
in the same compaction pass). Two of the rows below (`PG-MERGE-CLASSIFIER-F4` and
`PG-CANONICAL-HOLDOUT-NOT-AC-ENFORCED-WATCH`) are also summarized in STATE.md's kept
Session Resume Checkpoint for immediate resume visibility.

---

## Open Drift Items

| ID | Summary | Source | Target |
|----|---------|--------|--------|
| DRIFT-VP039-BC207038-TLS-TODO-001 | VP-INDEX carries stale TODOs for VP-039 (TLS reassembly). Out of feature-iec104 scope. | feature-iec104 F2 review (D-438) | SS-07 TLS owner — next TLS maintenance sweep |
| STORY-INDEX-IN-INPUTS-CHURN | Stories listing STORY-INDEX.md as input (STORY-164/165 + STORY-175/176/177/178/179 + STORY-157) re-stale on every index version bump. | D-477 → D-483 | Human decision: structural fix pending |
| DRIFT-docstring-scan | Python docstring RED-tense scanning not implemented in bin/check-green-doc-tense; deferred from wave-86 per F-W86S-P4-002 PO ruling (policy v5); confirmed-stale sites scrubbed by STORY-183 (test_lint_cycle_artifact.py:3,:5,:6,:125); separate future story needed. | wave-86 F-W86S-P4-002 PO ruling (policy v5) | future wave/maintenance |
| DRIFT-e2e-sibling-harnesses | tests/enip_e2e_real_pcaps_tests.rs + tests/e2e_corpus_smoke_tests.rs carry same LOCAL_SAMPLES/fixture_present silent-skip idiom STORY-182 fixes for IEC-104; ENIP pair is same ITI CC-BY-4.0 class (direct analog); deferred wave-86 per F-W86S-P6-002 orchestrator ruling (scope containment); follow-up story candidate at next planning. NOTE: tests/bc_2_12_011_story127_tests.rs REMOVED from this class — uses synthetic fallback (writes synthetic_16pkt_pcapng on None then runs full assertions), NOT in the silent-skip class. (corrected D-532 per F-W86S-P15-004) NOTE: tests/e2e_corpus_smoke_tests.rs is a directory-level skip VARIANT (:206-224), not the same fixture_present idiom as enip_e2e_real_pcaps_tests.rs. (wording aligned D-533) | wave-086 F-W86S-P6-002 orchestrator ruling (D-522) | next planning cycle |
| DRIFT-stale-red-scrub | 3 adjudicated stale RED-prose sites: tests/iec104_analyzer_tests.rs:6271 + tests/modbus_detection_tests.rs:2472/:2480 + tests/iec104_analyzer_tests.rs:6948-6953 (third site — currently falls through the `_` catch-all; stale post-STORY-180; past-tense reword prescribed; added D-533 per F-W86S-P16-003; Pattern-31/32 contiguity limitation documented in STORY-183 v2.6, regex widening deferred); PO reword prescriptions in DF-GREEN-DOC-TENSE-SWEEP v6 (policies.yaml); owner: next maintenance sweep. | wave-086 F-W86S-P6-009/010 PO adjudication (D-522) + D-533 F-W86S-P16-003 | next maintenance sweep |
| DRIFT-py-surface-outside-bin | tests/fixtures/mk_modbus_*.py + fuzz/seed_corpus.py are Python surface files outside STORY-183's bin/*.py glob scope; STORY-183 v2.10 covers bin/check-green-doc-tense + related patterns only. Scope extension deferred pending human decision. | wave-086 pass-13 F-P13-013 observation (D-530) | next wave or maintenance: extend STORY-183 scope or new story |
| DRIFT-TOOLCHAIN-ROLL-CLIPPY | The no-pin rolling `rust-toolchain@stable` (CLAUDE.md W7.1 / "no rust-toolchain pin") periodically promotes new clippy lints to `-D warnings` errors, breaking develop CI for every PR until a gate-fix lands. Recurred 2026-09-05: CI rolled to rustc/clippy 1.98.1, promoting `clippy::drain_collect`, fixed by gate-fix PR #461 (bd244ddf); precedent gate-fix PR #439. | D-547 gate-fix PR #461 (2026-09-05) | Revisit toolchain-pin decision at a future maintenance/planning pass (human infra decision) |
| DRIFT-STORY183-INHERITED-PATTERN-DOC-COMMENTS | **(F-S183-IMPL-P3-001, 2026-09-05).** Pre-existing `#` pattern-doc comments in `bin/test_check_green_doc_tense.py` (~:167-169, ~:200-203) became scan-eligible when STORY-183 added `.py` to `bin/check-green-doc-tense`'s scan glob; they currently do NOT self-flag only because embedded regex literals (`\b`/`\s+`) break the match. A future prose cleanup dropping those literals would silently make the Green-doc-tense gate self-flag them. Latent, zero live impact now. | wave-086 STORY-183 implementation pass 3 (D-549, 2026-09-05) | Reword these inherited comments to avoid bare pattern phrases in a future maintenance sweep |

---

## Active Carry-Forwards (open)

| ID | Summary | Target |
|---|---------|--------|
| ROUTE-W74-DEFERRED | **RESOLVED (D-509, 2026-07-24)** — OBS-1 absorbed by STORY-181 AC-181-004. OBS-2 remains open. | RESOLVED (OBS-1); OBS-2 per ROUTE-W74-OBS-2 |
| ROUTE-W74-OBS-2 | ROUTE-W74 OBS-2 not absorbed by STORY-166/181. Pending human scope decision. | Next wave or maintenance run |
| PERF-RERUN-001 | AC-149-003 re-run PASS at maint-2026-07-21. Remains OPEN per human scope decision D-490. | Next maintenance run |
| SEC-001 | **RESOLVED (D-509, 2026-07-24)** — absorbed into STORY-181 (wave-85). PR #438 5555495b delivered. | CLOSED |
| PR-407-FORK-RELEASE-OPS | **UPDATE (D-556, 2026-09-06).** Full review complete: code-reviewer MERGE-WITH-CHANGES, security-reviewer SAFE-WITH-CHANGES, no CRITICAL/HIGH; opt-in/inert-on-base-repo design CONFIRMED (0 repo variables set). `CHANGES_REQUESTED` review POSTED to contributor (2026-09-06T16:10Z) — blocking: CR-004 (Homebrew OS guard), CR-005 (unknown-channel tag misleading notes), SEC-407-04/CR-003 (injection-guard scope gap, CWE-94/78), CR-006 (dead doc refs); strongly-recommended: CR-001/002 (triplicated sign/notarize logic + Homebrew drift); fast-follow: SEC-407-01/02; LOW SEC-407-03 (dtolnay SHA reconciliation vs #451). Remains OPEN. | Awaiting contributor response to posted review |
| PR-451-DTOLNAY-PIN-CONFLICT | Human PR #451 (dtolnay-toolchain pin) reviewed APPROVE-WITH-CHANGES/security CLEAN (D-553), NOT merged pending the doc/policy contradiction vs. CLAUDE.md's dtolnay exemption text. PR DIRTY/conflicting since D-554 (2026-09-05). **UPDATE (D-556, 2026-09-06):** DEFERRED per explicit human decision this run — no action taken; still DIRTY/conflicting and still carrying the unresolved policy contradiction. | Human rebases + resolves policy contradiction |
| SCORECARD-ENABLEMENT-RUNBOOK | Before setting SCORECARD_ENABLED=true: document CWE-200 publish_results:true risk. | Whenever scorecard is enabled |
| ROUTE-DOC-DEFER-2026-07-21 | PR #431 review residuals: ADR-0001 Consequences (LOW), ADR-0002 Deviations (NIT), ADR-0012 stale (LOW). | Next doc sweep |
| PG-W84-012 | bin-selftest required-status-check gap. Ops task PENDING: devops-engineer + human authorization required. Also: wire test_lint_cycle_artifact.py + test_compute_input_hash.py (F-W86S-P9-012). | Ops task (devops-engineer dispatch, future wave) |
| F-007-PROCESS-GAP | **RESOLVED-SUBSUMED (D-550, S-7.02, 2026-09-05).** [process-gap] Self-application smoke AC gap in STORY-183 — folded into `PG-W86-001` (story-writer positive-coverage-assertion checklist gap), whose disposition is local carry-forward requiring DF-VALIDATION-001 research-agent validation before any GitHub issue filing. | RESOLVED-SUBSUMED — tracked under PG-W86-001, next maintenance/planning for upstream-filing validation. |
| DF-MERGE-AUTH-STANDING-GRANT-W86 | **GOVERNANCE (D-547, 2026-09-05).** Human granted the orchestrator a STANDING merge-authorization for wave-86 and forward ("in the future you can do these merges") — future pr-manager dispatches may execute automated squash-merges (`gh pr merge --squash --admin --delete-branch`) once a PR is CI-green AND per-story adversarial convergence is complete, instead of HALT-to-human per DF-MERGE-AUTH-CLASSIFIER-001. Supersedes the STORY-180/181 human-executes-merge pattern for subsequent wave-86 PRs (starting with STORY-182 PR #460). | Applies to wave-86 and forward per-story delivery merges |
| MAINT-2026-09-05-HOLDOUT-GAPS | **(D-553, 2026-09-05).** 5 LOW holdout coverage gaps + 1 fixture-wiring opportunity routed to product-owner from Sweep 4: base IEC-104 untimed detection, Modbus detection, ENIP fixtures, DNP3/ARP feature-tree seeds, `summary --hosts`, and wiring HS-133..136 to STORY-182's committed fixtures. Per DF-VALIDATION-001, any promoted to a GitHub issue require research-agent validation first. | product-owner backlog triage |
| BC-2.20.002-LOW-DOUBLE-GUARD | **(D-562, 2026-09-07).** Accepted LOW residual: BC-2.20.002's version-reject canonical test vectors use length `0x04`, now double-guarded (rejected by both the version check and the new min-length-7 check from the STORY-184 RFC-min-7 rework). Coverage is preserved — the vectors still exercise the version-reject path. Not a defect; optional future test-vector hardening only. | Optional future maintenance sweep |
| PG-MERGE-CLASSIFIER-F4 | **GOVERNANCE/OPERATING ARRANGEMENT (D-563, 2026-09-07).** The Claude Code permission classifier blocks/hangs agent-dispatched `gh pr merge` for F4 story PRs (blocked #465, #467; #466 slipped through). Human has elected to run each F4 story merge manually at the wave boundary for the rest of F4 — future pr-manager dispatches should expect this and route the merge step to the human rather than retrying the agent-dispatched merge. | Standing arrangement for the remainder of F4 (STORY-186..194) |
| PG-CANONICAL-HOLDOUT-NOT-AC-ENFORCED-WATCH | **WATCH (D-563, 2026-09-07).** `PG-CANONICAL-HOLDOUT-NOT-AC-ENFORCED` (see `cycles/feature-s7comm/lessons.md`) has now recurred twice (STORY-184, STORY-185) — nearing the 3× codification threshold. If it recurs a third time on STORY-186, escalate to a self-improvement story per DF-VALIDATION-001 (research-agent validation required before any GitHub issue). | Watch on STORY-186 adversarial review |
