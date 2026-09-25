---
document_type: behavioral-contract
level: L3
version: "1.7"
status: draft
producer: product-owner
timestamp: 2026-09-24T00:00:00Z
phase: f2
origin: greenfield
extracted_from: null
traces_to: .factory/specs/domain/domain-spec.md
subsystem: SS-21
capability: CAP-21
lifecycle_status: active
introduced: feature-s7comm
modified:
  - version: "1.7"
    date: 2026-09-25
    change: "STORY-187 P22-F-1: replace stale line-number anchors with symbol names. Architecture Anchors' Tests anchor cited the doc comment of `test_BC_2_21_002_empty_dt_followed_by_frame_same_delivery_stays_unclassified` at '(lines ~2495-2505)' — line numbers drift with every test-file edit and are not a stable anchor. Replaced with a reference to the function name only (no line numbers). Swept all 8 STORY-187 SS-21 BCs (BC-2.21.001/002/004/005/006/007/008/009) for other body-level line-number citations into `tests/s7comm_analyzer_tests.rs` or `src/analyzer/s7comm.rs`: none found outside this BC's own Architecture Anchors entry (remaining line-number mentions in BC-2.21.001/006/008 are confined to their `modified:` frontmatter history, left untouched per instruction). No change to Preconditions/Postconditions/Invariants/Edge Cases — anchor-stability correction only."
  - version: "1.6"
    date: 2026-09-25
    change: "STORY-187 per-story adversarial pass 16 (P16-F-4) (NIT). Architecture Anchors 'Tests anchor' arithmetic error: the pass-13 (v1.5) entry said 'two are newly added since the prior anchor count (10, pass 4, F-33)' but the current count is 13 — three tests were added since pass 4, not two: `test_BC_2_21_002_empty_dt_followed_by_frame_same_delivery_stays_unclassified` (pass 12) was omitted from the pass-13 delta accounting alongside the two pass-14 additions already cited (`test_BC_2_21_002_non_0x32_dt_frame_on_classic_flow_not_dissected`, `test_BC_2_21_002_unparseable_cotp_does_not_classify`). Corrected 'two' to 'three' and added the missing trace for the empty-DT test: Postcondition 6 (a `protocol_id: None` DT frame carries no protocol evidence and never sets `classified_protocol`) and Edge Case EC-004's `None`-DT-carries-no-evidence premise, together with BC-2.20.013 Postcondition 1a (frame-boundary discipline) — the test guards against `tpkt_payload`/`protocol_id` evaluation leaking past the empty-payload DT frame's own boundary into a trailing CR frame's bytes within the same delivery. Full 13-function test list verified unchanged by direct grep against `tests/s7comm_analyzer_tests.rs` (worktree HEAD). No change to Preconditions/Postconditions/Invariants/Edge Cases — Architecture Anchors arithmetic/traceability correction only."
  - version: "1.5"
    date: 2026-09-25
    change: "STORY-187 per-story adversarial pass 13 (P13-F-1): Architecture Anchor test-count re-verification. The Architecture Anchors 'Tests anchor' entry was stale — it cited 10 tests from pass 4 (F-33), before tests added in passes 12-14. Re-grepped `tests/s7comm_analyzer_tests.rs` directly and replaced with the actual current count (13 `test_BC_2_21_002_*` functions) and the full function-name list, verified 2026-09-25 against worktree HEAD 38ff7ee1. Explicitly traced the two new tests: `test_BC_2_21_002_non_0x32_dt_frame_on_classic_flow_not_dissected` to Postcondition 3 / Invariant 2 / Invariant 4, and `test_BC_2_21_002_unparseable_cotp_does_not_classify` to Postcondition 1. No change to Preconditions/Postconditions/Invariants/Edge Cases — Architecture Anchors traceability correction only."
  - version: "1.4"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 4→5 findings (F-39/F-42/F-43). F-39 (MEDIUM): the Verification Properties table row for the four-way dispatch totality property said 'proptest P1 ... VP-NNN allocation deferred to the F2 INTEGRATE sub-burst' — stale and self-contradictory: this BC's own VP Anchors section (below) already cites VP-053 (proptest P0, not P1) as registered per VP-INDEX.md v2.48, and VP-INDEX.md itself lists VP-053 as registered proptest P0 tracing this BC. Row corrected to 'VP-053 (proptest P0) — registered; see VP Anchors', removing the stale P1/deferred wording and the priority mismatch. Swept all 8 STORY-187 SS-21 per-BC Verification Properties rows (BC-2.21.001/002/004/005/006/007/008/009) for the same 'VP-NNN ... deferred' pattern contradicting a registered VP Anchors citation: only this BC (BC-2.21.002) had the defect. BC-2.21.004/009 already correctly cite VP-051 as registered (F-14/F-35, prior passes); BC-2.21.005/006/007 say 'VP-NNN allocation deferred' but their own VP Anchors sections explicitly state 'None dedicated — not ... registered', so no contradiction exists there — left unchanged. BC-2.21.008's VP row/VP Anchors are addressed separately under F-43 (this same burst). NOTED, not fixed (out of this burst's per-story BC scope): BC-2.21.027's Verification Properties row (line ~100) has the identical stale 'VP-NNN allocation deferred to the F2 INTEGRATE sub-burst' pattern next to no VP Anchors registration mismatch of its own text, but VP-INDEX.md registers VP-053 (proptest P0) tracing BC-2.21.027 — flagged for a wave-gate drift item; not edited here."
  - version: "1.3"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 4 (F-32/F-33/F-35). F-32 (HIGH): Canonical Test Vectors row for a DT frame with `protocol_id: None` on a flow's first-observed DT frame was misrouted to 'Unclassified gap (BC-2.21.028)' — wrong: this BC's own Postcondition 5 (lines ~78-82) and the implementation (`src/analyzer/s7comm.rs` `dispatch_cotp_frame`'s DT-branch catch-all arm, doc-commented as BC-2.21.027) both route this outcome to BC-2.21.027; BC-2.21.028 is the distinct unparseable-COTP-payload path (`parse_cotp_header` returning `None`, Postcondition 1 / Related BCs), and Invariant 1 forbids a single input class reaching more than one branch. Row corrected to BC-2.21.027, now consistent with Postcondition 5, Related BCs, and the VP-053 Anchors citation. Grepped this BC and BC-2.21.001 for any other 027/028 misrouting: BC-2.21.001's Postcondition 1 `classified_protocol` bullet cites '(BC-2.21.027/028)' as a generic dual reference to both unclassified-gap branches (not a specific routing claim) — no defect found, no change made there. F-33: Architecture Anchors' Tests anchor undercounted this BC's test suite — actual is 10 `test_BC_2_21_002_*` functions in `tests/s7comm_analyzer_tests.rs` `mod story_187`, not 9; added `test_BC_2_21_002_setup_comm_fixture_pcap_well_formed_no_findings` (exercises the committed Setup Communication pcap fixture) to the anchor list. Re-verified all 8 STORY-187 SS-21 per-BC test counts recorded in the pass-3 sweep (BC-2.21.001/002/004/005/006/007/008/009) against the current test file: only this BC's count was stale (9 vs. actual 10); BC-2.21.001 (11), BC-2.21.004 (3), BC-2.21.005 (2), BC-2.21.006 (3), BC-2.21.007 (3), BC-2.21.008 (7), BC-2.21.009 (7) all confirmed accurate, no changes needed to those files' counts."
  - version: "1.2"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 3 (F-31), re-anchor sweep. Traceability 'Stories' field corrected from '(TBD — story-writer assigns in F3)' to 'STORY-187 (also a formal-hardening re-verification anchor for STORY-194)', matching the Story Anchor section below. Architecture Module's '(planned)' marker removed. Architecture Anchors' anchor corrected: as implemented, `S7commAnalyzer::on_data` is an INHERENT `pub fn` on `S7commAnalyzer` — there is currently no `impl StreamAnalyzer for S7commAnalyzer` in the tree (the prior anchor's `impl StreamAnalyzer for S7commAnalyzer { fn on_data(...) }` framing was aspirational and never matched the code); wiring `on_data` to the `StreamAnalyzer` trait/dispatcher is STORY-193's explicit scope per the type's own doc comment ('Not yet registered with the dispatcher ... `DispatchTarget::S7comm` wiring is STORY-193's scope'). The four-way `match` this BC specifies is implemented in the private helper `dispatch_cotp_frame`, called from `on_data`, not inlined directly in `on_data` itself. Added a Tests anchor citing `tests/s7comm_analyzer_tests.rs`'s `mod story_187` BC-2.21.002-labeled test functions (9 tests; verified test-name/BC-ID alignment, no drift found)."
  - version: "1.1"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 1 (F-01/F-02/F-12/F-13/F-14), human ruling 2026-09-24. F-02: Postcondition 6 rewritten — classification now requires a `Some(byte)` `protocol_id` on the first qualifying DT frame (`0x32`->Classic, `0x72`->Plus, other->Unclassified); a `protocol_id: None` DT frame carries no protocol evidence and never sets `classified_protocol`, deferring classification to a later DT frame if any. Postcondition 5 amended to distinguish the `None` sub-case (no classification effect) from the `Some(other)` sub-case (sets Unclassified) while both still dispatch identically to the unclassified-gap path. BC-2.21.001's Postcondition 1 (`classified_protocol` bullet) already stated the correct non-`None` requirement and is confirmed the winning definition; this BC's prior Postcondition 6 wording ('any protocol_id value, including None') is superseded. Added Edge Case EC-004 (None-DT first, then 0x32 DT -> Classic). F-12: Postcondition 3 amended to gate classic S7comm dissection on the flow's STICKY `classified_protocol == Some(Classic)`, not merely on the current frame's `protocol_id == Some(0x32)` — a 0x32-leading DT frame on a flow already classified Plus or Unclassified by an earlier DT frame is NOT dissected (ADR-014 Decision 2 no-misattribution guarantee). Added Edge Case EC-005 (Plus-classified flow, later 0x32 DT frame not dissected). Added Invariant 4 tying the gate to Invariant 2's misattribution guarantee."
deprecated: null
deprecated_by: null
replacement: null
retired: null
removed: null
removal_reason: null
inputs:
  - docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md
  - .factory/specs/architecture/ARCH-INDEX.md
input-hash: "cf116b5"
---

# BC-2.21.002: `S7commAnalyzer::on_data` Four-Way Dispatch on `CotpHeader::protocol_id`

## Description

`S7commAnalyzer::on_data` is the SS-21 entry point that drives the frame-walk loop
defined at the SS-20 boundary (BC-2.20.013): for every complete TPKT frame extracted,
it calls `iso_on_tcp::parse_tpkt_header` then `iso_on_tcp::parse_cotp_header`, then
branches on the returned `CotpHeader::protocol_id` per ADR-014 Decision 2's four-row
disambiguation table. This BC formalizes that dispatch as the single integration point
between SS-20's frame extraction and SS-21's protocol-specific dissection — every
other BC-2.21.NNN classification/parsing contract is reached through exactly one of
this BC's four branches.

## Preconditions

1. A complete TPKT frame has been extracted by the frame-walk loop (BC-2.20.013), and
   `parse_cotp_header` has been called on its COTP payload.
2. `S7commFlowState` exists for the flow (created lazily per BC-2.21.001 if this is the
   first `on_data` call).

## Postconditions

1. If `parse_cotp_header` returns `None` (BC-2.20.011, unrecognized TPDU type or
   truncated-beyond-carry-repair): the frame is routed to the unclassified-gap path
   (BC-2.21.028) — never force-fit to any of the three recognized branches.
2. If `parse_cotp_header` returns `Some(CotpHeader { tpdu_type: ConnectRequest | ConnectConfirm, .. })`:
   `S7commFlowState.session_established` is updated per BC-2.21.001 Postcondition 1;
   no protocol classification occurs (classification is deferred to the first DT
   frame, per ADR-014 Decision 2 row 3).
3. If `parse_cotp_header` returns `Some(CotpHeader { tpdu_type: DataTransfer, protocol_id: Some(0x32), .. })`
   **and** the flow's sticky `S7commFlowState.classified_protocol` is (or, by this very
   frame per Postcondition 6, becomes) `Some(S7Protocol::Classic)`: dispatch to classic
   S7comm dissection — `parse_s7comm_header` is called on the slice beginning at
   `payload_offset` (BC-2.21.004 onward). If the flow's sticky `classified_protocol`
   was already established as `Some(S7Protocol::Plus)` or
   `Some(S7Protocol::Unclassified)` by an earlier DT frame on this flow, this
   `0x32`-leading DT frame is **not** dissected — no `parse_s7comm_header` call is made
   and no finding is emitted for it, per ADR-014 Decision 2's no-misattribution
   guarantee: a flow, once classified, is never re-interpreted under a different
   protocol's parser (Reconciled per human ruling, STORY-187 per-story adversarial pass
   1, F-12, 2026-09-24).
4. If `parse_cotp_header` returns `Some(CotpHeader { tpdu_type: DataTransfer, protocol_id: Some(0x72), .. })`:
   dispatch to the S7comm-plus framing-only path (BC-2.21.024/025/026).
5. If `parse_cotp_header` returns `Some(CotpHeader { tpdu_type: DataTransfer, protocol_id: Some(other), .. })`
   where `other ∉ {0x32, 0x72}`, or `protocol_id: None` on a DT frame with an empty
   payload (BC-2.20.010): dispatch to the unclassified-gap path (BC-2.21.027) —
   identical DISPATCH treatment to Postcondition 1's `None`-from-`parse_cotp_header`
   case (neither is ever counted as S7comm and no dissection of any kind is attempted).
   The two sub-cases differ, however, in their effect on `classified_protocol` (see
   Postcondition 6): a `Some(other)` `protocol_id` IS protocol evidence and — on the
   flow's first DT frame carrying a `Some(byte)` `protocol_id` — sets
   `classified_protocol = Some(S7Protocol::Unclassified)` (sticky thereafter); a `None`
   `protocol_id` (empty DT payload) carries NO protocol evidence at all and never sets
   `classified_protocol` by itself, regardless of whether it is the flow's first DT
   frame (Reconciled per human ruling, STORY-187 per-story adversarial pass 1, F-02,
   2026-09-24).
6. On the first DT frame observed for a flow whose `protocol_id` is `Some(byte)`
   (`0x32` -> `Some(S7Protocol::Classic)`, `0x72` -> `Some(S7Protocol::Plus)`, any other
   byte -> `Some(S7Protocol::Unclassified)` — BC-2.21.001 Postcondition 1),
   `S7commFlowState.classified_protocol` is set exactly once (first-classification-wins,
   BC-2.21.001 Edge Case EC-002); subsequent DT frames on the same flow — including ones
   whose `protocol_id` differs, is a different `Some(other)`, or is `None` — never
   overwrite it. A DT frame whose `protocol_id` is `None` (empty DT payload,
   BC-2.20.010) carries NO protocol evidence: it never sets `classified_protocol`, and
   if it is the flow's first DT frame, classification remains deferred —
   `classified_protocol` stays `None` — until a later DT frame (if any) on the same flow
   carries a `Some(byte)` `protocol_id`, at which point THAT later frame is the one that
   sets it. (This postcondition's prior wording — "any `protocol_id` value, including
   `None`" — is SUPERSEDED and reconciled with BC-2.21.001 Postcondition 1 per human
   ruling, STORY-187 per-story adversarial pass 1, F-02, 2026-09-24: BC-2.21.001's
   non-`None` requirement wins.)

## Invariants

1. **Exhaustive four-way branch**: every possible `parse_cotp_header` return value
   (`None`; `Some` with `tpdu_type` ∈ {ConnectRequest, ConnectConfirm}; `Some` with
   `tpdu_type: DataTransfer` and `protocol_id` ∈ {`Some(0x32)`, `Some(0x72)`,
   `Some(other)`, `None`}) is routed to exactly one of the branches above — no branch
   is reachable from more than one input class, and no input class reaches zero
   branches.
2. **Load-bearing correctness (ADR-014 Decision 2)**: this dispatch is the single
   location in wirerust's binary-ICS analyzer family where post-classification
   disambiguation determines *which named protocol* a flow is attributed to, not
   merely whether the flow is malformed. A defect here can misattribute non-S7comm
   traffic (MMS, ICCP, unrecognized) to S7comm — the correctness property this BC and
   BC-2.21.027/028 jointly guarantee never occurs.
3. **No re-dispatch on protocol change**: per Postcondition 6, a flow's classification
   is sticky from its first qualifying (`Some(byte)` `protocol_id`) DT frame; this is a
   deliberate simplicity choice, not an oversight — flagged for B2's consideration as a
   possible anomaly signal, not re-classified by B1.
4. **Sticky classification gates re-dissection, not just re-classification**:
   Postcondition 3's classic-dissection eligibility check (`classified_protocol ==
   Some(Classic)`) is the concrete mechanism by which Invariant 2's no-misattribution
   guarantee holds across a flow's ENTIRE lifetime, not merely at the moment of first
   classification — every subsequent `0x32`-leading DT frame on an already-classified
   flow is still subject to this gate (F-12 ruling, human-ratified 2026-09-24).

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | A flow observes only CR/CC frames, never a DT frame, before the pcap ends | `classified_protocol` remains `None` for the lifetime of the flow; no dissection of any kind occurs; the flow is simply "S7comm-port-102, session tracked, never classified" |
| EC-002 | A flow's very first observed frame is a DT frame with `protocol_id: Some(0x32)` (no prior CR/CC observed, e.g. mid-capture start) | Classification proceeds normally from the DT frame alone; `session_established` remains `false` (no CR/CC observed) but this does not block classic S7comm dissection |
| EC-003 | Two DT frames arrive back-to-back within a single `on_data` call (multiple frames per delivery, mirrors BC-2.20.013's multi-frame walk) | Each frame is dispatched independently through this BC's branches; `classified_protocol`'s first-write-wins rule applies across the pair in arrival order |
| EC-004 | A flow's first DT frame has `protocol_id: None` (empty payload — no protocol evidence), and a later DT frame on the same flow has `protocol_id: Some(0x32)` | `classified_protocol` remains `None` after the first (`None`-`protocol_id`) DT frame — it is not "used up" as the flow's classifying frame. The second (`Some(0x32)`) DT frame is the one that sets `classified_protocol = Some(S7Protocol::Classic)` (Postcondition 6) and is dissected via Postcondition 3, since `classified_protocol` is still unset at the moment this frame is dispatched (F-02 ruling) |
| EC-005 | A flow's first DT frame has `protocol_id: Some(0x72)` (classifies `Some(S7Protocol::Plus)`), and a later DT frame on the same flow has `protocol_id: Some(0x32)` | Sticky `classified_protocol` remains `Some(S7Protocol::Plus)`. The later `0x32`-leading DT frame is evaluated against Postcondition 3's gate and found NOT eligible for classic dissection (`classified_protocol` is already `Some(Plus)`, not `Some(Classic)`) — no `parse_s7comm_header` call, no finding, per ADR-014 Decision 2's no-misattribution guarantee (F-12 ruling) |

## Canonical Test Vectors

| `parse_cotp_header` result | Dispatch outcome | Category |
|---|---|---|
| `None` | Unclassified gap (BC-2.21.028) | reject: unparseable COTP payload |
| `Some(CotpHeader{tpdu_type: ConnectRequest, protocol_id: None, ..})` | Session tracking only, no classification | happy-path: session establishment |
| `Some(CotpHeader{tpdu_type: DataTransfer, protocol_id: Some(0x32), ..})` | Classic S7comm dissection entry (BC-2.21.004) | happy-path: classic dispatch |
| `Some(CotpHeader{tpdu_type: DataTransfer, protocol_id: Some(0x72), ..})` | S7comm-plus framing-only path (BC-2.21.024) | happy-path: plus dispatch |
| `Some(CotpHeader{tpdu_type: DataTransfer, protocol_id: Some(0x00), ..})` | Unclassified gap (BC-2.21.027); sets `classified_protocol = Some(Unclassified)` if this is the flow's first `Some(byte)`-bearing DT frame | reject: neither classic nor plus |
| `Some(CotpHeader{tpdu_type: DataTransfer, protocol_id: None, ..})` on a flow's first-observed DT frame | Unclassified gap (BC-2.21.027; corrected from the previously-misrouted BC-2.21.028, F-32); `classified_protocol` stays `None` — no protocol evidence, classification deferred | reject: empty DT payload, no evidence (F-02) |
| `Some(CotpHeader{tpdu_type: DataTransfer, protocol_id: Some(0x32), ..})` on a flow whose sticky `classified_protocol == Some(Plus)` | Not dissected — no `parse_s7comm_header` call, no finding | reject: misattribution guard (F-12) |

## Verification Properties

| Property | Proof Method (planned) |
|----------|-------------------------|
| The four-way dispatch is total and non-overlapping over every possible `Option<CotpHeader>` value reachable from `parse_cotp_header` — no input value reaches zero or more than one branch | VP-053 (proptest P0) — registered; see VP Anchors (mirrors VP-046's `classify_frame_format` totality treatment) |

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| Capability Anchor Justification | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 — this BC is the dispatch surface CAP-21's description explicitly names as the analyzer's core disambiguation behavior |
| L2 Domain Invariants | INV-2 (Content-First Dispatch Precedence — this dispatch fires only inside a flow already routed to `DispatchTarget::S7comm` via port-102 fallback; it does not itself perform content-first classification, it consumes its result) |
| Architecture Module | SS-21 (`src/analyzer/s7comm.rs`); ADR-014 Decision 2 |
| ADR | ADR-014 Decision 2 (in-analyzer disambiguation, load-bearing) |
| Stories | STORY-187 (also a formal-hardening re-verification anchor for STORY-194) |
| Feature | feature-s7comm |
| MITRE Techniques | (none — dispatch contract only; individual classification branches carry no finding emission in this part; B2 authors emission) |

## Related BCs

- BC-2.20.009 — depends on (`protocol_id` extraction this BC branches on)
- BC-2.20.011 — depends on (the `None` case this BC's Postcondition 1 handles)
- BC-2.21.001 — depends on (`S7commFlowState` fields this BC reads/writes)
- BC-2.21.004 — composes with (classic S7comm dissection entry point)
- BC-2.21.024 — composes with (S7comm-plus framing-only entry point)
- BC-2.21.027 — composes with (unrecognized-protocol_id unclassified-gap path)
- BC-2.21.028 — composes with (unparseable-COTP-payload unclassified-gap path)

## Architecture Anchors

- `src/analyzer/s7comm.rs` — `S7commAnalyzer::on_data` is an INHERENT `pub fn` (`impl S7commAnalyzer { pub fn on_data(&mut self, flow_key: FlowKey, data: &[u8], ts: u32, direction: Direction) }`), not a `StreamAnalyzer` trait method — no `impl StreamAnalyzer for S7commAnalyzer` exists yet in the tree; that wiring is STORY-193's scope (corrected 2026-09-24, F-31; the prior anchor's `impl StreamAnalyzer for S7commAnalyzer { fn on_data(...) }` framing never matched the code). The four-way `match` on `CotpHeader` this BC specifies is implemented in the private helper `fn dispatch_cotp_frame(state: &mut S7commFlowState, cotp: Option<iso_on_tcp::CotpHeader>, tpkt_payload: &[u8], direction: Direction, ts: u32, findings: &mut Vec<Finding>)`, called from `on_data`'s frame-walk loop
- `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md §Decision 2` — the frozen four-row disambiguation table
- `tests/s7comm_analyzer_tests.rs` — Tests anchor: 13 `test_BC_2_21_002_*` functions (re-counted by direct grep, verified 2026-09-25 against worktree HEAD 38ff7ee1): `test_BC_2_21_002_cr_cc_updates_session_no_classification`, `test_BC_2_21_002_cr_cc_only_never_classifies`, `test_BC_2_21_002_classic_s7comm_dispatch_asserts_classified_protocol_classic`, `test_BC_2_21_002_malformed_0x32_frame_on_classic_flow_emits_exactly_one_t0814`, `test_BC_2_21_002_sticky_first_classification`, `test_BC_2_21_002_dt_first_frame_no_prior_cr_cc`, `test_BC_2_21_002_none_protocol_id_dt_first_then_0x32_dt_classifies_classic`, `test_BC_2_21_002_two_frames_one_delivery_first_write_wins`, `test_BC_2_21_002_0x32_dt_frame_not_dissected_when_sticky_classified_plus_or_unclassified`, `test_BC_2_21_002_empty_dt_followed_by_frame_same_delivery_stays_unclassified`, `test_BC_2_21_002_setup_comm_fixture_pcap_well_formed_no_findings`, `test_BC_2_21_002_non_0x32_dt_frame_on_classic_flow_not_dissected`, `test_BC_2_21_002_unparseable_cotp_does_not_classify`. Of these, three are newly added since the prior anchor count (10, pass 4, F-33): `test_BC_2_21_002_empty_dt_followed_by_frame_same_delivery_stays_unclassified` (added pass 12) traces Postcondition 6 (a `protocol_id: None` DT frame carries no protocol evidence and never sets `classified_protocol`, per F-02) and Edge Case EC-004's `None`-DT-carries-no-evidence premise, together with BC-2.20.013 Postcondition 1a (frame-boundary discipline) — per the test's own doc comment on `test_BC_2_21_002_empty_dt_followed_by_frame_same_delivery_stays_unclassified`, this is a frame-bounded-dispatch regression guard: `tpkt_payload`/`protocol_id` evaluation for the empty-payload DT frame must stop at that frame's own boundary and never read into a trailing CR frame's bytes within the same delivery (which would spuriously classify the flow instead of leaving `classified_protocol` at `None`); `test_BC_2_21_002_non_0x32_dt_frame_on_classic_flow_not_dissected` (added pass 14) traces Postcondition 3 / Invariant 2 / Invariant 4 (non-`0x32` DT frame arriving on an already-classified-Classic flow is not dissected — the sticky-classification gate holds for non-matching, not just competing-protocol, frames); `test_BC_2_21_002_unparseable_cotp_does_not_classify` (added pass 14) traces Postcondition 1 (`parse_cotp_header` returning `None` routes to the unclassified-gap path and never sets `classified_protocol`).

## Story Anchor

STORY-187 (also a formal-hardening re-verification anchor for STORY-194)

## VP Anchors

- VP-053 (proptest P0) — `protocol_id` Four-Way Dispatch Totality and Unclassified
  Never-Force-Fit; registered F2 INTEGRATE sub-burst per VP-INDEX.md v2.48; traces
  BC-2.21.002, BC-2.21.027, BC-2.21.028 (this is the BC's protocol_id-dispatch
  concern, superseding the earlier "anticipated VP-048 range" speculation)

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | reads/writes `S7commFlowState` (per-flow, not global) |
| **Deterministic** | yes — same byte sequence and prior flow state always produce the same dispatch outcome |
| **Thread safety** | single-flow-owner access pattern (mirrors sibling analyzers) |
| **Overall classification** | effectful shell (flow-state mutation) around pure SS-20 parse calls |
