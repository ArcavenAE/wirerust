---
document_type: behavioral-contract
level: L3
version: "1.5"
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
  - version: "1.5"
    date: 2026-09-25
    change: "STORY-187 per-story adversarial pass 13 (P13-F-1): Architecture Anchor test-count re-verification. The Architecture Anchors 'Tests anchor' entry was stale — it cited 11 tests and an 'F-31, no drift found' claim from pass 3, before tests added in passes 12-14. Re-grepped `tests/s7comm_analyzer_tests.rs` directly and replaced with the actual current count (13 `test_BC_2_21_001_*` functions) and the full function-name list, verified 2026-09-25 against worktree HEAD 38ff7ee1. Removed the stale 'no drift found (F-31)' claim, which no longer held. Also made explicit: EC-008/EC-009 (overwrite-then-match interaction) trace to `test_BC_2_21_001_most_recent_cr_direction_wins`, and the `session_established` monotonic (false→true only) property traces to `test_BC_2_21_001_session_established_is_monotonic`. No change to Preconditions/Postconditions/Invariants/Edge Cases — Architecture Anchors traceability correction only."
  - version: "1.4"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 11/12 (P11-F-2), orchestrator decision (LOW). Aligned Postcondition 1's `cr_observed_dir` bullet to the implementation (`src/analyzer/s7comm.rs` CR arm, `dispatch_cotp_frame`'s `ConnectRequest` arm ~:637, and the field's own doc comment): the field 'records the direction of the most recently observed COTP CR (overwritten by each CR; not cleared on a matching CC — `session_established` is monotonic [only ever transitions false→true], so retention has no observable effect).' Previously this BC described the field only as tracking the 'most recently observed, not-yet-matched' CR, which did not name the implementation's deliberate choice to leave the field populated (and unconditionally overwritten by each new CR) even after a CR/CC match — a wording gap, not a behavior gap, since no postcondition ever reads `cr_observed_dir` after `session_established` is set. EC-007 (repeated CR, no intervening CC) reworded from 'reflects the most recent CR (or is left at its first-observed value — implementation's choice)' to state deterministic most-recent-CR-wins overwrite semantics (not an implementation choice). Added EC-008 (CR(A), CR(B), CC(A) → established, since CC(A) is opposite the most recent CR(B), the superseded CR(A) no longer participates) and EC-009 (CR(A), CR(B), CC(B) → not established, since CC(B) matches the direction of the most recent CR(B)) to make the overwrite-then-match interaction explicit. No postcondition/invariant semantics changed — precision-only wording fix aligning the spec to already-correct, already-shipped code."
  - version: "1.3"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 4→5 findings (F-39/F-42/F-43). F-42 (LOW): Postcondition 1's `classified_protocol` bullet tail ('remains `None` until the first DT frame is observed') and Edge Case EC-002's Expected Behavior ('set only once, from the *first* DT frame observed') both imprecisely said 'the first DT frame' where BC-2.21.002 Postconditions 5/6 (F-02 ruling, pass 1) require 'the first DT frame carrying a `Some(byte)` `protocol_id`' — a DT frame with `protocol_id: None` does not classify and is not the qualifying 'first DT frame' for this purpose. Both wordings corrected to 'the first DT frame carrying a `Some(byte)` `protocol_id`', reconciling this BC with BC-2.21.002's already-correct Postcondition 6 phrasing. No postcondition/invariant semantics changed (the underlying rule was already correctly stated in the surrounding sentence and in BC-2.21.002) — precision-only wording fix."
  - version: "1.2"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 3 (F-31), re-anchor sweep. Traceability 'Stories' field corrected from the F2-drafting placeholder '(TBD — story-writer assigns in F3)' to 'STORY-187' — story decomposition has since completed and this BC's implementation landed in STORY-187. Architecture Module and Architecture Anchors' '(planned)' markers removed: `src/analyzer/s7comm.rs` now exists and `pub struct S7commFlowState { .. }` (with `carry_c2s`/`carry_s2c`, `carry_overflow_reported_c2s`/`_s2c`, `session_established`, `cr_observed_dir: Option<Direction>`, `classified_protocol: Option<S7Protocol>`, `malformed_header_reported_c2s`/`_s2c`) is the actual, implemented struct this BC specifies — nothing about it remains 'planned.' Added a Tests anchor citing `tests/s7comm_analyzer_tests.rs`'s `mod story_187` BC-2.21.001-labeled test functions (11 tests; verified test-name/BC-ID alignment, no drift found)."
  - version: "1.1"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 1 (F-01/F-02/F-12/F-13/F-14), human ruling 2026-09-24. F-01: Postcondition 1's `session_established` bullet redefined — 'matching CC' now precisely means a COTP CC observed in the direction OPPOSITE to a previously-observed CR on the same flow (CC-only, CC-before-CR, and same-direction CC do NOT set it; CR alone does not set it). Added an explicit 'at minimum' permission for an additional pending-CR-direction tracking field (e.g. `cr_observed_dir: Option<Direction>`). Added Edge Cases EC-004..EC-007 (CC-only mid-flow capture, CC-before-CR, same-direction CC, repeated CR). Canonical Test Vectors' session-tracking row amended to specify opposite-direction CC. F-02 (BC-2.21.002 reconciliation, this BC unchanged/confirmed as the winning definition): Postcondition 1's `classified_protocol` bullet already correctly required a non-`None` `protocol_id` — no change needed here; BC-2.21.002 Postconditions 5/6 were the ones amended to match."
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

# BC-2.21.001: `S7commFlowState` Owns TPKT/COTP Carry Buffers, S7comm Classification State, and Per-Direction Dedup Flags

## Description

`S7commFlowState` (`src/analyzer/s7comm.rs`, SS-21) is the per-flow bookkeeping struct
for a TCP/102 flow classified `DispatchTarget::S7comm`. Per ADR-014 Decision 1, SS-20
(`iso_on_tcp.rs`) is deliberately stateless — the directional TPKT/COTP carry buffers
it requires (BC-2.20.013/014) are fields on this struct, not on a separate
`IsoOnTcpFlowState`. This BC is the SS-21 counterpart to BC-2.20.016: it defines the
concrete field set `S7commFlowState` carries, establishing the single source of truth
every other BC-2.21.NNN contract's flow-state references point back to.

## Preconditions

1. A TCP flow has been classified `DispatchTarget::S7comm` (port 102, Rule 9, ADR-014
   Decision 2) and `S7commAnalyzer::on_data` has been called at least once for it.

## Postconditions

1. `S7commFlowState` contains, at minimum, the following fields:
   - `carry_c2s: Vec<u8>`, `carry_s2c: Vec<u8>` — directional TPKT frame-reassembly
     carry buffers (BC-2.20.013), bounded at `MAX_S7_ISO_ON_TCP_CARRY_BYTES = 65,535`
     (BC-2.20.014).
   - `carry_overflow_reported_c2s: bool`, `carry_overflow_reported_s2c: bool` —
     per-direction dedup flags for the carry-overflow T0814 finding (BC-2.20.014).
   - `session_established: bool` — set when a COTP CC (BC-2.20.008) is observed in the
     direction OPPOSITE to a previously-observed COTP CR (BC-2.20.007) on this flow.
     "Matching CC" is precisely defined as directionally-opposite-of-a-prior-CR, not
     merely "any CC after any CR": a CC observed with no prior CR on this flow
     (CC-only, e.g. a mid-flow capture start), a CC observed BEFORE any CR, and a CC
     observed in the SAME direction as the flow's already-observed CR do NOT set this
     flag; a CR alone, with no subsequent opposite-direction CC, does not set it
     either. Classification of the upper-layer protocol is deferred until the first DT
     frame regardless of this flag's value (Reconciled per human ruling, STORY-187
     per-story adversarial pass 1, F-01, 2026-09-24).
   - `cr_observed_dir: Option<Direction>` (or an equivalently-purposed field — this
     "at minimum" field set explicitly PERMITS but does not mandate this exact name)
     — an ADDITIONAL field, beyond `session_established` itself, hosting the
     pending-CR-tracking state that makes the directional-matching rule above testable:
     it records the direction of the most recently observed COTP CR on this flow
     (overwritten by each subsequent CR — most-recent-CR-wins is this field's defined
     overwrite semantics, not an implementation choice), so a later CC can be tested
     for direction-opposite-ness against it. This field is NOT cleared on a matching
     opposite-direction CC: it continues to reflect the most recent CR even after a
     successful CR/CC match. This retention has no observable effect on this BC's own
     behavior, because `session_established` only ever transitions `false` → `true`
     (monotonic) and no postcondition reads `cr_observed_dir` after that transition —
     the field's post-match value is simply unobserved, not meaningfully "stale."
     Any implementation shape that lets the analyzer determine "was the most recent
     CR's direction opposite this CC's direction" satisfies this BC (F-01 ruling,
     human-ratified 2026-09-24; overwrite/non-clearing semantics aligned to the
     shipped implementation, orchestrator decision, STORY-187 pass 11, P11-F-2,
     2026-09-24).
   - `classified_protocol: Option<S7Protocol>` — set on the first DT frame with a
     non-`None` `protocol_id` (BC-2.21.002); `S7Protocol` distinguishes `Classic`,
     `Plus`, and `Unclassified` (BC-2.21.027/028); remains `None` until the first DT
     frame carrying a `Some(byte)` `protocol_id` is observed (a DT frame with
     `protocol_id: None` carries no protocol evidence and does not qualify, F-02/F-42).
   - `malformed_header_reported_c2s: bool`, `malformed_header_reported_s2c: bool` —
     per-direction dedup flags for S7comm-header-level bounds/truncation rejects
     (BC-2.21.004/007/008/009), distinct from SS-20's carry-overflow dedup flags
     (Invariant 2).
2. No field on `S7commFlowState` duplicates a field SS-20 owns — the carry buffers
   listed above are the *only* SS-20-originated state; everything else is
   S7comm-specific.
3. `S7commFlowState` is created lazily on the first `on_data` call for a newly
   classified flow and stored in the analyzer's per-flow map, keyed by `FlowKey`
   (mirrors `Iec104FlowState`/`DnpFlowState` precedent).

## Invariants

1. **Single state owner**: exactly one `S7commFlowState` exists per classified flow;
   no shadow or duplicate state struct exists elsewhere in SS-21.
2. **Dedup-flag separation**: carry-overflow dedup (SS-20-originated, BC-2.20.014) and
   malformed-S7comm-header dedup (SS-21-originated, this BC) are tracked by distinct
   flags, mirroring the IEC-104 precedent (BC-2.19.026 Invariant 5) of never
   conflating anomaly classes under one suppression flag.
3. **No S7comm-plus-specific decode state**: per ADR-014 Decision 6, `S7commFlowState`
   does not carry any field implying function-code-level S7comm-plus state (e.g., no
   `plus_last_function` field) — only `classified_protocol` and the bounded
   session-setup-metadata fields defined in BC-2.21.025.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | A flow is classified `DispatchTarget::S7comm` but never sends any bytes before close | `S7commFlowState` is never created (lazy-on-first-`on_data`); `on_flow_close` is a no-op for this flow |
| EC-002 | The same flow observes a DT frame, then later another DT frame with a *different* `protocol_id` value than the first | `classified_protocol` is set only once, from the first DT frame carrying a `Some(byte)` `protocol_id` (first-classification-wins); subsequent DT frames with a different `protocol_id` do not overwrite it — flagged as a B2 (MITRE emission) anomaly-detection concern, not a B1 dissection concern |
| EC-003 | `carry_overflow_reported_c2s` and `malformed_header_reported_c2s` are both set to `true` on the same flow direction | Both flags coexist independently; each governs only its own anomaly class's dedup, per Invariant 2 |
| EC-004 | A flow's capture begins mid-session; the first COTP frame observed on this flow is a CC, with no prior CR seen (CC-only, e.g. mid-flow capture start) | `session_established` remains `false`; the pending-CR-direction tracking field remains `None` — a CC with no preceding CR cannot "match" anything (F-01 ruling) |
| EC-005 | A CC is observed before any CR is observed on the flow (out-of-order arrival) | `session_established` remains `false` when the CC arrives (no CR yet observed to be opposite to). If a CR subsequently arrives — in either direction — this BC does not retroactively set `session_established` from the earlier CC: only a CC observed AFTER a CR, in the direction opposite that CR, sets the flag (matching is forward-looking from the CR, never backward-looking from the CC) (F-01 ruling) |
| EC-006 | A CR is observed in direction A, then a CC is observed also in direction A (same direction as the CR, not opposite) | `session_established` remains `false` — a same-direction CC is never "matching," regardless of any COTP reference-number correlation (ADR-014's per-flow model does not decode/correlate COTP reference numbers) (F-01 ruling) |
| EC-007 | A CR is observed in direction A, then a second CR is also observed in direction A (repeated CR, no intervening CC) | `session_established` remains `false`; the pending-CR-direction tracking field is overwritten by each CR and reflects direction A (the most recent CR) — most-recent-CR-wins is this field's defined overwrite semantics, not an implementation choice (F-01 ruling; overwrite semantics clarified, orchestrator decision, STORY-187 pass 11, P11-F-2) |
| EC-008 | A CR is observed in direction A, then a second CR is observed in direction B (opposite of A, no intervening CC), then a CC is observed in direction A (opposite of the most recent CR, B) | `session_established == true` — the pending-CR-direction field holds only the most recent CR (B, per EC-007's overwrite semantics); the earlier, superseded CR(A) no longer participates in matching, so the CC is tested only against B and found opposite (F-01 ruling; added STORY-187 pass 11, P11-F-2) |
| EC-009 | A CR is observed in direction A, then a second CR is observed in direction B (opposite of A, no intervening CC), then a CC is observed in direction B (same direction as the most recent CR, B) | `session_established` remains `false` — the CC is tested only against the most recent CR (B) per EC-007's overwrite semantics, and a same-direction CC is never "matching" (EC-006); the superseded CR(A) does not participate (F-01 ruling; added STORY-187 pass 11, P11-F-2) |

## Canonical Test Vectors

| Scenario | Expected `S7commFlowState` state | Category |
|----------|-----------------------------------|---------|
| First `on_data` call for a newly classified flow, zero bytes delivered | Struct created with all `Vec` fields empty, all `bool` fields `false`, `classified_protocol: None` | happy-path: initialization |
| CR observed on direction A, then CC observed on direction B (opposite of A) on the same flow | `session_established == true`, `classified_protocol` still `None` | happy-path: session tracking without classification |
| CR observed on direction A, then CC observed also on direction A (same direction) | `session_established == false` | reject: same-direction CC is not "matching" (F-01) |
| First DT frame with `protocol_id: Some(0x32)` observed | `classified_protocol == Some(S7Protocol::Classic)` | happy-path: classic classification |

## Verification Properties

(No independent VP-NNN — this BC is a structural/architectural contract, verified by
code-review inspection of the struct definition and field usage, mirroring
BC-2.20.016's treatment. Runtime behaviors of individual fields are exercised by the
proptest/cargo-fuzz harnesses anchored to the specific BCs that mutate each field.)

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| Capability Anchor Justification | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 — this BC defines the per-flow state struct that is the load-bearing data structure for every other CAP-21 dissection behavior |
| L2 Domain Invariants | None directly (architectural/structural state-ownership contract; carry-buffer fields are governed by SS-20's INV-2-adjacent framing, not a distinct domain invariant of their own) |
| Architecture Module | SS-21 (`src/analyzer/s7comm.rs`); ADR-014 Decision 1 (per-flow state placement ruling) |
| ADR | ADR-014 Decisions 1, 8 |
| Stories | STORY-187 |
| Feature | feature-s7comm |
| MITRE Techniques | (none — structural contract, no finding emission) |

## Related BCs

- BC-2.20.013 — depends on (carry-buffer fields defined at the SS-20 boundary, hosted here per ADR-014 Decision 1)
- BC-2.20.014 — depends on (carry-overflow dedup flags hosted here)
- BC-2.20.016 — composes with (SS-20's module-boundary contract; this BC is its SS-21-side counterpart)
- BC-2.21.002 — composes with (`on_data` reads/writes this struct's fields on every call)
- BC-2.21.003 — composes with (`on_flow_close` removes this struct)

## Architecture Anchors

- `src/analyzer/s7comm.rs` — `pub struct S7commFlowState { .. }` field definition (implemented, STORY-187): `carry_c2s: Vec<u8>`, `carry_s2c: Vec<u8>`, `carry_overflow_reported_c2s: bool`, `carry_overflow_reported_s2c: bool`, `session_established: bool`, `cr_observed_dir: Option<Direction>`, `classified_protocol: Option<S7Protocol>`, `malformed_header_reported_c2s: bool`, `malformed_header_reported_s2c: bool`
- `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md §Decision 1` — "Per-flow state placement (resolves F1 §2.3 open question)"
- `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md §Decision 8` — carry-buffer sizing and dedup-flag placement
- `tests/s7comm_analyzer_tests.rs` — Tests anchor: 13 `test_BC_2_21_001_*` functions (re-counted by direct grep, verified 2026-09-25 against worktree HEAD 38ff7ee1): `test_BC_2_21_001_flow_state_field_set`, `test_BC_2_21_001_lazy_flow_state_creation`, `test_BC_2_21_001_never_touched_flow_state_never_created`, `test_BC_2_21_001_dedup_flags_independent_c2s`, `test_BC_2_21_001_cr_then_opposite_cc_sets_session_established`, `test_BC_2_21_001_cr_only_session_not_established`, `test_BC_2_21_001_cc_only_no_prior_cr_session_not_established`, `test_BC_2_21_001_most_recent_cr_direction_wins`, `test_BC_2_21_001_cc_before_cr_session_not_established`, `test_BC_2_21_001_same_direction_cc_session_not_established`, `test_BC_2_21_001_repeated_same_direction_cr_session_not_established`, `test_BC_2_21_001_fields_constructible_with_expected_types`, `test_BC_2_21_001_session_established_is_monotonic`. EC-008/EC-009 (overwrite-then-match interaction) are traced to `test_BC_2_21_001_most_recent_cr_direction_wins`; the `session_established` monotonic (false→true only, never reverts) property is traced to `test_BC_2_21_001_session_established_is_monotonic`.

## Story Anchor

STORY-187

## VP Anchors

(None — structural/architectural contract.)

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | none (per-flow state, not global) |
| **Deterministic** | n/a — structural data-definition contract |
| **Thread safety** | n/a (single-flow-owner access pattern, mirrors sibling analyzers) |
| **Overall classification** | architectural boundary contract |
