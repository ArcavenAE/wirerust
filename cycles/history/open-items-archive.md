# Open Items Archive — Resolved / Historical

Extracted from STATE.md on 2026-07-14 (compact-state, D-444).

Contains: resolved Blocking Issues, resolved/stale Open Items / Backlog rows from
feature-protocol-coverage era and prior maintenance cycles. All items below are RESOLVED,
DOWNGRADED, CLOSED, REFUTED, or ACCEPTED — not active tracking items.

---

## Resolved Blocking Issues

| ID | Summary | Priority | Owner | Status |
|----|---------|----------|-------|--------|
| F2-SCOPE-DRIFT-UDP-001 | ADR-012 Decision 6 corrected from TCP-only to TCP+UDP dynamic detection. All docs reconciled. (TransportProto, u16) keying consistent. | HIGH | architect | **RESOLVED 2026-07-01** |

---

## Resolved Open Items / Backlog (feature-protocol-coverage era + maintenance)

| ID | Summary | Final Status |
|----|---------|-------------|
| SEC-005 + SEC-006 | ENIP on_flow_close unwired (CWE-401+CWE-770); DNP3 flow-map no cap+on_flow_close. | **RESOLVED (D-383, PR #362 / issue #342 CLOSED 2026-07-06). STORY-148 SUPERSEDED (D-399).** |
| PERF-001/002 + BENCHMARK-GAP-001 | TLS carry-path +10.3% regression; no fragmented-handshake fixture. | **→ STORY-149 DELIVERED (PR #374, D-395, 2026-07-07). Issue #360 CLOSED.** |
| TLS-DRAIN-DUP-001 | ~220-line C2S/S2C drain-loop duplication in tls.rs. | **RESOLVED — STORY-150 DELIVERED (PR #379, D-402, 2026-07-08)** |
| BC-ANCHOR-DRIFT-OUTOFCYCLE-001 | 12 stale tls.rs anchor sites. | **FOLDED into STORY-150 v1.3 AC-150-006 (D-398, wave 71)** |
| ARCH-INDEX-COUNT-DRIFT-001 | SS-11 34→35, SS-16 15→16. | **RESOLVED 2026-07-01 (ARCH-INDEX v2.10)** |
| DF-CANONICAL-FRAME-HOLDOUT-001-F3-OBLIGATION | Canonical-value ACs + 7 canonical-value holdout scenarios (HS-124..126, HS-129..132). | **RESOLVED D-339 2026-07-02** |
| F4-FIXTURE-NEED-001 | HS-127..132 require crafted pcap fixtures at F4 eval time. | **RESOLVED (D-373, 2026-07-04) — 8 pcaps in `.factory/holdout-fixtures/`** |
| SEC-001-ENIP | Unsafe split-borrow enip.rs `on_data`. | **DOWNGRADED to LOW (D-383, 2026-07-06) — sound-as-written; tech-debt** |
| SEC-001-STORY153 | `unclassified_port_counts` ceiling ~131,072 keys; no doc-comment. | **Issue #361 filed (docs: add ceiling doc-comment). Validated LOW.** |
| SEC-004 + SEC-007 | 7+ counter `+= 1` → saturating_add cosmetic. | **DOWNGRADED to LOW cosmetic (D-383) — FALSE-POSITIVE on overflow (u64).** |
| TLS-FILLBUF-PUBLIC-SEAM-001 + MAINT-SC-001 | fill_buf_for_testing seam (W7.1); indicatif patch. | **W7.1 backlog / optional dep-refresh** |
| ARCH-INDEX-DOCMAP-COMPONENT-COUNT-001 | ARCH-INDEX Document Map '24 components' → system now has 26. | **RESOLVED (ARCH-INDEX v2.12, D-362, 2026-07-03)** |
| F-F2P13-OBS-VP042D | VP-042 sub-property (d) not mapped to dedicated harness. | **RESOLVED (D-375, 2026-07-04) — (d) dropped; VP-042 = 3 harnesses A/B/C** |
| INPUT-HASH-ERROR-STORIES-001 | STORY-001/091/121 input-hash anomalies. | **REFUTED-CLOSED (maint-2026-07-08 DF-VALIDATION-001 triage)** |
| F-F2P11-001 | BC-2.05.010 TCP-path references flow_key.src_port/dst_port. | **RESOLVED (BC-2.05.010 v1.4 lower_port().min(upper_port()), D-375)** |
| F-F2P11-002 | BC-2.05.011 EC-002 label 'Http/502' should be 'Modbus/502'. | **RESOLVED (BC-2.05.011 EC-002 label corrected in F5 sweep)** |
| F-F3P5-001 | dependency-graph.md:277 phantom ProtocolsArgs/AnalyzeArgs types. | **RESOLVED as F-F3P6-004 (dep-graph v3.6)** |
| F-F3P5-002 | STORY-154 AC-154-002 'run_analyze() wires args.coverage_gaps'. | **RESOLVED as F-F3P6-005 (STORY-153/154 v1.5)** |
| BC-STORY-ANCHOR-TBD-001 | 9 feature BCs' Story Anchor section reads 'TBD (F3 story decomposition)'. | **Resolved at F4 story delivery.** |
| F3-ADV-P7-O1 | STORY-153 AC-153-005 udp_unclassified_counts declaration scope clarification. | **Addressed at F4 implementation (data-flow forces correct placement).** |
| F-F3P9-001 | STORY-152 run_protocols stdout-only vs path-routing clarification. | **RESOLVED (STORY-152 v1.5, D-366 — AC-152-002 prose reconciled)** |
| HS-INDEX-ENIP-WAVE-DRIFT-001 | HS-INDEX ENIP feature section waves '63-68' vs dep-graph E-20 waves 58-61. | **CONFIRMED — DEFERRED Route C. Batch into next spec-coherence sweep.** |
| F-F3P10-001 | STORY-153 no Red-Gate test asserting unclassified_flows fires when coverage_gaps_enabled=false. | **Applied at F4 implementation; covered by holdouts HS-040/HS-095.** |
| VP042D-FROZEN-RESIDUAL-001 | VP-INDEX VP-042 sub-property (d) residual in frozen doc. | **RESOLVED (D-375, 2026-07-04) — (d) label dropped from VP-INDEX + ADR-012** |
| BC-2.05.010-LOWERPORT-WORDING-001 | Frozen BC-2.05.010 references non-existent flow_key.src_port/dst_port. | **RESOLVED (D-375, 2026-07-04) — BC-2.05.010 v1.4 lower_port().min(upper_port())** |
| F-F3P12-001 | STORY-151 test_BC_2_18_003_supported_ports_mirror excludes port 53 unnecessarily. | **RESOLVED (STORY-151 v1.5, D-362, 2026-07-03)** |
| F-F3P13-001 | STORY-152 AC-152-002 prose over-claims --json=path file routing. | **RESOLVED (STORY-152 v1.5, D-366 — stdout-only reconcile per frozen BC)** |
| F-F3P18-O2 | STORY-154 render path must re-lookup KNOWN_PROTOCOLS for name. | **RESOLVED (D-371, 2026-07-04 — applied at STORY-154 F4 implementation)** |
| F-F3P18-O1 | Frozen BC-2.12.024 PC-4 uses 'supported: false' field notation. | **RESOLVED-STALE — same as BC-2.12.024-PC4-PHANTOM-SUPPORTED-001, resolved D-375** |
| BC-2.05.010-EC006-UNREACHABLE-001 | BC-2.05.010 EC-006 (Tcp,502)==2 unreachable since classify() routes 502→Modbus. | **Phase-5 BC reconciliation (DF-VALIDATION-001-gated). Deferred.** |
| STORY-152-GLOBAL-FLAG-NOOP-001 | Global --csv/--output-format json silently no-op under protocols. | **RESOLVED (D-370, F-W68-01 fix, PR #354 0e700a9)** |
| BC-2.12.022-FWFIX-SYNC-001 | BC-2.12.022 v1.0 lags shipped --json=PATH file-routing behavior. | **RESOLVED (D-375 — BC-2.12.022 v1.1 synced)** |
| PG-F5-RECONCILE-INCOMPLETE-001 | Spec-reconciliation missed Invariant 2 + phantom variant-shape + BC-INDEX title-cell in first burst. | **RESOLVED (D-377 — 3rd/final sweep completed; checklist codified)** |
| PG-SPEC-FRESHNESS-ON-FIX-001 | No gate ties BC version to shipped CLI flag-matrix when a wave-level fix adds behavior. | **cycle-close retrospective — logged in feature-protocol-coverage lessons** |
| PG-HELP-PROVENANCE-CLI-DOC-001 | clap doc-comments MUST NOT contain internal factory IDs. | **Codified in implementer checklist; applied at STORY-154.** |
| STORY-154-DNS53-TCP-GAP-001 | (Tcp,53) DNS-over-TCP must be genuine gap. | **RESOLVED (D-371 — applied at STORY-154 F4 implementation)** |
| STORY-154-CAN-DECODE-HOIST-001 | Single can_decode hoist in main.rs. | **RESOLVED (D-371 — applied at STORY-154 F4 implementation)** |
| EPICS-TOTAL-BCS-DRIFT-001 | epics.md total_bcs 337 vs BC-INDEX 345 active. | **CONFIRMED — DEFERRED Route C. Batch into next spec-coherence sweep.** |
| STORY-154-ALL-COVERAGEGAPS-TEST-001 | No analyze --all --coverage-gaps combined integration test. | **RESOLVED (D-379, PR #356 commit abc048e)** |
| STORY-154-TESTCOUNT-COMMENT-001 | tests/integration_tests.rs ~line 1161 stale count 20→21 tests. | **LOW cosmetic — addressed in maint follow-up.** |
| STORY-154-WEAK-UNKNOWN-ASSERT-001 | 2 terminal tests use bare contains("unknown"). | **RESOLVED (D-379, PR #356 commit f90dfb8 — tightened to line-level checks)** |
| BC-2.12.024-PC4-PHANTOM-SUPPORTED-001 | BC-2.12.024 PC-4 references phantom `supported:` field. | **RESOLVED (D-375 — BC-2.12.024 v1.2; derived predicate applied)** |
| STORY-153-RUNANALYZE-DOC-STALE-001 | `src/main.rs` run_analyze coverage_gaps doc-comment stale. | **RESOLVED (D-379, PR #356 commit 7fbb57c)** |
| STORY-154-LOOKUP-ARP-DEADCLAUSE-001 | `lookup_protocol_state` ARP disjunct provably unreachable. | **RESOLVED (D-379, PR #356 commit 0fdaa29)** |
| STORY-148-BASIS-RESOLVED-001 | STORY-148 drafted as fix vehicle for SEC-005+SEC-006; PR #362 closed those findings. | **RESOLVED (D-399 — STORY-148 SUPERSEDED by PR #362). Row closed.** |
| DNP3-CLOSEDFLOW-REOPEN-REUSE-001 | Same 5-tuple closes/re-opens within capture → Vec lists flow twice. | **OPEN — DF-VALIDATION-001-gated (research-agent validation required)** |
| SEC-008 + SEC-009 | `closed_flow_direct_operates` Vec not cleared; `CloseReason` dropped. | **OPEN — documented acceptable; no action required.** |
| SILENT-LIMIT-GAPS-001..004 | 4 observability gaps: ARP evictions, Modbus drops, TLS+HTTP dropped_map_entries. | **RESOLVED (D-385, PR #365, develop cc2a87c, 2026-07-06)** |
| MODBUS-INVALID-ADU-LATCH-NOT-A-GAP | Modbus invalid-ADU latch proposed as silent gap. | **REJECTED (D-385 — `parse_errors` already surfaces it)** |
| HTTP-AC008-NEG-TEST-001 | Add negative regression test for dropped_map_entries. | **RESOLVED (D-386, PR #366)** |
| EVICTION-NO-FINDING-NEG-TEST-001 | Regression tests that eviction/drop emit no Finding. | **RESOLVED (D-386, PR #366)** |
| ARP-BINDINGS-EVICT-PRECHECK-COSMETIC-001 | ARP bindings-evicted pre-check duplicated. | **RESOLVED (D-386, PR #366 — insert_binding_lru returns bool; 2 call sites deduped)** |
| REBIND-COUNT-SATURATING-001 | `rebind_count` uses plain `+=` not `saturating_add`. | **RESOLVED — folded into PR #384 PF-001 sweep (c4eb1f4 2026-07-08)** |
| SEC-W71-001 | CWE-22 path traversal in `bin/compute-input-hash`. | **FILED — GitHub issue #392 (2026-07-09)** |
| SEC-W71-002 + SEC-W71-003 | Wave-71 security LOW observations. | **ACCEPTED — no issue to file** |
| CR-W71-001 | Code review MINOR + 3 NITs from wave-71 (no code-review.md written — PG-W71-CODEREVIEW-ARTIFACT). | **CLOSED-UNVERIFIABLE (maint-2026-07-08). PG codified to STORY-158 AC-158-006.** |
| STALE-INPUT-HASH-076-101 | STORY-076 + STORY-101 stale hashes due to BC-2.11.001 v1.9 cascade. | **RESOLVED (D-412 — mechanically re-baselined; MATCH=112 STALE=0)** |
| F-W72-P15-L01 | dep-graph v3.8 frontmatter totals stale. | **RESOLVED at wave-72 close.** |
| CD-03-RC-01 | STORY-INDEX release-mapping note predates wave-72 v0.12.0 targeting. | **RESOLVED — v0.12.0 released (D-422).** |
| II-02-BC-INDEX-BUMP-ASYMMETRY | BC-INDEX bump asymmetry for pre-delivery BC amendments. | **RESOLVED (D-412 — BC-INDEX v2.22 committed)** |
| SC-01-TEMPLATE-REGISTRY | Template-registry entry absent for wave-72 story template variant. | **RESOLVED — STORY-158 delivery accepted absence; advisory closed.** |
| CC-01-STORY-161-TDD-MODE | STORY-161 tdd_mode:strict on governance-only E-11 story. | **RESOLVED (D-413 — confirmed consistent with E-11 stories that include test-writing ACs)** |
| PG-GITFLOW-SQUASH-BACKMERGE | Squash back-merges sever main/develop shared history. | **MITIGATED (D-422 — fast-forward back-merge resolves divergence at v0.12.0).** |

---

## Archived Notes Section

From STATE.md Notes section (pre-compaction):

- `.factory/` is a `factory-artifacts` orphan-branch worktree, gitignored from `develop`.
- Not on crates.io (D-300). Squash-only on develop (D-289). Branch protection (D-290/D-315).
- Cycle `fix-tls-clienthello-frag` CLOSED (D-316). maint-2026-07-01 CLOSED (D-318). Cycle `feature-protocol-coverage` STARTED (D-320) / CLOSED (D-382, 2026-07-05). v0.11.2 RELEASED (PR #358/tag v0.11.2). S-7.02 satisfied (STORY-155). F2 HUMAN GATE APPROVED (D-338). F3 story decomposition COMPLETE (D-339). F3 Passes 1–18 complete: all findings catalogued above. F3 ADVERSARIAL STORY CONVERGENCE ACHIEVED (Pass-16/17/18; BC-5.39.001 SATISFIED). See `cycles/history/decision-log-archive.md` for full narrative.

---

## Active items carried forward (NOT in this archive)

The following items are still active and appear in STATE.md Active Carry-Forwards:
- SEC-001-S168 (STORY-172)
- SEC-001-S158, SEC-002-S158 (advisory, bin/lint-cycle-artifact CI wiring)
- ROUTE-BC-DEFERRED-2026-07-11, ROUTE-W74-DEFERRED, PERF-RERUN-001
- DRIFT-BACKMERGE-SQUASH-001, DRIFT-VP039-BC207038-TLS-TODO-001
- RETRANSMIT-NS-FALSEPOS-001
- STORY-166, F3-handoff cleanup items

---

## Resolved Drift Items (extracted from STATE.md on 2026-09-07, compact-state)

| ID | Summary | Source | Target |
|----|---------|--------|--------|
| DRIFT-BACKMERGE-SQUASH-001 | **RESOLVED (D-491, 2026-07-21).** v0.13.1 back-merge PR #433 TRUE-MERGE (dc7331fb to develop). | v0.12.1 release → RESOLVED D-491 (2026-07-21) | RESOLVED — archive at next compact. |
| PG-W84-UPSTREAM-BATCH | **RESOLVED (D-515, 2026-07-25).** Research-validated: 001 DUP #749; 002 DUP #457; 003 DUP #681; 004 DUP #572; 005 DUP #651/#626; 006 FILED #764; 008 DUP #663. | wave-084 S-7.02 (D-486) | RESOLVED — archive at next compact. |
| PG-W84-LOCAL-BATCH | **RESOLVED (D-553, maint-2026-09-05).** PG-W84-010 + PG-W85-003 combined → STORY-183, DELIVERED (D-549, PR #462 `b273af21`) as part of wave-86 (CLOSED D-550; RELEASED v0.13.3 D-551). PG-W84-012 remains open — tracked separately under Active Carry-Forwards. | wave-084 S-7.02 (D-486) → RESOLVED D-553 | RESOLVED — archive at next compact; PG-W84-012 deferred |
| PG-W85-001 | **RESOLVED (D-515, 2026-07-25).** NOVEL-UPSTREAM — filed drbothen/vsdd-factory#765. | wave-085 pass-2 (D-496) | RESOLVED — archive at next compact. |
| PG-W85-002 | **RESOLVED-DUPLICATE (D-515, 2026-07-25).** Class covered by #470/#507/#216. | wave-085 P2-P4 (D-496/497/498) | RESOLVED — archive at next compact. |
| PG-W85-003 | **RESOLVED (D-553, maint-2026-09-05).** Combined with PG-W84-010 → STORY-183, DELIVERED (D-549, PR #462 `b273af21`) as part of wave-86 (CLOSED D-550; RELEASED v0.13.3 D-551). | wave-085 STORY-180 pass-1 (D-506) → RESOLVED D-553 | RESOLVED — archive at next compact |
| PG-W85-004 | **RESOLVED-DUPLICATE (D-515, 2026-07-25).** Covered by #626 + #696/#651. | wave-085 D-509 (2026-07-24) | RESOLVED — archive at next compact. |
| PG-W85-005 | **RESOLVED (D-553, maint-2026-09-05).** → STORY-182, DELIVERED (D-548, PR #460 `35ffa135`) as part of wave-86 (CLOSED D-550; RELEASED v0.13.3 D-551). | wave-085 gate G1 (D-510) → RESOLVED D-553 | RESOLVED — archive at next compact |
| DRIFT-src-glob-blindspot | **RESOLVED-FOLDED (D-526, 2026-07-26):** fix vehicle = STORY-183 v1.9 (F-W86S-P9-009); pathspec src/*.rs added alongside src/**/*.rs + mitre.rs scan assertion. | wave-086 pass-9 F-W86S-P9-009 [process-gap] | RESOLVED-FOLDED — archive at next compact. |
| DRIFT-STORY183-CHANGELOG-PIPE | **RESOLVED (D-545, 2026-09-04).** Unescaped pipes escaped (`\|`, `\|\|`, `` `\|` ``) in both STORY-182 and STORY-183 changelog rows during the D-545 level-fix burst; validate-table-cell-count hook mismatch closed. | wave-086 pass-24 (D-541, 2026-09-04) → RESOLVED D-545 | RESOLVED — archive at next compact. |
| DRIFT-DEPGRAPH-BACKFILL | **RESOLVED (D-546, 2026-09-04).** GAP-002 and GAP-003 both closed. GAP-003: real Wave 62-66 (STORY-139..142,144..146) + Wave 72-75 (STORY-158..165) sections backfilled, replacing prose placeholders; total_stories anchored to literal `ls STORY-*.md` count = 136, matching STORY-INDEX exactly, zero residual. GAP-002: 9 new BC-to-Stories rows (STORY-167..174, STORY-180) + 4 new VP-to-Stories rows (VP-044..047) + 2 arm-extension rows + 1 Stories-column extension backfilled into the E-22 traceability matrices. dependency-graph.md v3.10→v3.12 (total_edges 138→143; total_points headline reconciled 807→792 exact match against STORY-INDEX total_points: 792, zero residual — the v3.11 807 figure was inherited incremental-delta drift, not a supersession-convention mismatch; total_stories 136 zero residual). Acyclicity re-verified via Kahn across all 136 nodes. | wave-086 story-approval-gate consistency audit (D-544) → PARTIALLY RESOLVED D-545 → RESOLVED D-546 | RESOLVED — archive at next compact. |
| DRIFT-EPICS-STALE-v21 | **RESOLVED (D-545, 2026-09-04).** epics.md bumped v2.1→v2.2: currency reconciliation to 136 stories; E-11 6→23 stories/75 pts; new Epic E-22 section added in full; total_bcs 337→380, reconciled exactly against BC-INDEX v2.37 (0 unassigned, 0 double-assigned, 0 residual gap). | wave-086 story-approval-gate consistency audit (D-544) → RESOLVED D-545 | RESOLVED — archive at next compact. |
| DRIFT-EPICS-NARRATIVE-SECTIONS | **RESOLVED (D-546, 2026-09-04).** epics.md bumped v2.2→v2.3: full `## Epic` narrative sections authored for E-13 (Multi-Tag Finding Schema Migration), E-14 (Modbus TCP Analyzer), and E-16 (ARP Security Analyzer), each inserted at its correct ordinal position (E-13/E-14 between E-12 and E-15; E-16 between E-15 and E-17) following the Goal/BCs/Subsystems-touched/Estimated-stories/Rationale structure used by every other epic section. No numbers changed: E-13 remains 2 stories/0 new BCs/21 pts; E-14 remains 5 stories/25 BCs/45 pts; E-16 remains 6 stories/16 BCs/50 pts — identical to the pre-existing Summary/Coverage table values. total_bcs unchanged at 380. | epics.md v2.2 currency pass (D-545, 2026-09-04) → RESOLVED D-546 | RESOLVED — archive at next compact. |

---

## Resolved/Cleared Active Carry-Forwards (extracted from STATE.md on 2026-09-07, compact-state)

| ID | Summary | Target |
|---|---------|--------|
| ROUTE-BC-DEFER-2026-07-11 | **CLEARED (D-554, 2026-09-05).** All soaked Rust-dep Dependabot PRs merged to develop (#458/#443/#442/#444/#459); develop=adc9428d, CI green. | CLEARED — archive at next compact. |
| DEP-SOAK-FOLLOWUP-2026-07-27 | **CLEARED (D-554, 2026-09-05).** 17 not-yet-soaked crates eligible 2026-07-21..27; Dependabot #434/#435/#436 included. All 5 authorized Rust-dep merges (#459/#458/#444/#443/#442) confirmed LANDED on develop (tip adc9428d), CI green. | CLEARED — archive at next compact. |
| MAINT-2026-09-05-DEP-RUST-MERGE-HANDOFF | **RESOLVED (D-554, 2026-09-05).** All 5 Dependabot Rust-dep bumps (#459 owo-colors `adc9428d`, #458 clap `52681a45`, #444 serde `b5444077`, #443 serde_json `fac7f3a6`, #442 anyhow `03c3c560`) human-AUTHORIZED and MERGED to develop; develop tip `0b1ea806`→`adc9428d`; CI fully green (Test/Fuzz success), zero regression. Cleared `DEP-SOAK-FOLLOWUP-2026-07-27` + `ROUTE-BC-DEFER-2026-07-11`. | RESOLVED — archive at next compact. |
| MAINT-2026-09-05-ACTIONS-BUMPS-HELD | **RESOLVED (D-556, 2026-09-06).** All 3 remaining held bumps supply-chain reviewed MERGE-SAFE (each SHA cryptographically traced to genuine upstream release tag; diffs SHA-swap-only) and MERGED: #455 codeql-action/upload-sarif (`1c7d7401`), #449 ossf/scorecard-action (`36bccab4`, also patches transitive CVE-2026-53488/47262/34986), #436 actions/checkout (`9c499512`). Held set now EMPTY. | RESOLVED — archive at next compact. |
| MAINT-2026-09-05-DOCFIX-QUEUED | **RESOLVED (D-556, 2026-09-06).** PR #465 ("docs: document help subcommand and ADR-0008 numbering gap") squash-merged to develop as `97361cd4`, head branch deleted. pr-reviewer APPROVE, CI 13/13 green. Docs-only, outside CHANGELOG-obligation trigger set. | RESOLVED — archive at next compact. |
| F4-OBLIGATION-ADR014-CLAUDEMD | **RESOLVED (D-562, 2026-09-07).** `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md` + the CLAUDE.md port-102 note LANDED on `develop` via PR #466 (STORY-184 DELIVERED, `7ce0db5c`). ADR-014 status stays `proposed` (moves to `accepted` at F7 when the feature implementation is complete). | RESOLVED — archive at next compact. |
| DEFERRED-BC-2.20.005-STALE-LEN4 | **RESOLVED (2026-09-07, STORY-185 pre-implementation spec fix).** BC-2.20.005's stale "length==4 header-only produces empty payload" reference (postcondition-4 parenthetical, EC-001, EC-002, empty-vector canonical row) corrected to truncated-delivery framing consistent with BC-2.20.004's RFC-min-7 accept floor. COTP-parse behavior contract itself unchanged. BC-2.20.005's own input-hash unaffected (`cf116b5`, derived from ADR-014 + ARCH-INDEX.md, neither touched). `STORY-185` rehashed (`275ae46`→`7f6bb1e`) via canonical `bin/compute-input-hash --write`; `--scan` confirms MATCH and the pre-existing 22-story background-stale set unchanged. | RESOLVED — archive at next compact. |
