---
document_type: behavioral-contract
level: L3
version: "1.3"
status: draft
producer: product-owner
timestamp: 2026-09-06T00:00:00Z
phase: f2
origin: greenfield
extracted_from: null
traces_to: .factory/specs/domain/domain-spec.md
subsystem: SS-21
capability: CAP-21
lifecycle_status: active
introduced: feature-s7comm
modified:
  - version: "1.3"
    date: 2026-10-05
    change: "STORY-188 pass-3 sweep (no P3-F finding): added EC-005 for the defensive unsliceable-block `NoParameterBlock` return (implemented in `classify_job_ack_function`, unreachable behind BC-2.21.009 bounds check, asserted not-NoParameterBlock-iff-param_length>0 by the STORY-188 Kani harness); Verification Properties now states proof scope accurately (deterministic 256-value enumeration test + 2000-case proptest skeleton, full run deferred to STORY-194; VP-052 status draft)."
  - version: "1.2"
    date: 2026-10-04
    change: "STORY-188 pass-2 P2-F-07 (NIT): arm anchor re-cited from :385 (fn signature) to the match-arm line :417; function itself stays :385. P2-F-02 (MINOR): PC2/EC-003 Setup-Communication-Ack param_length==0 claim removed (disproved by canonical cnblogs Ack_Data, param_length 8); replaced with bare Ack_Data empty-parameter-block example. P2-F-04 (NIT): Verification Properties VP-NNN -> VP-052 (proptest P1); VP-INDEX version ref v2.48 -> v2.55."
  - version: "1.1"
    date: 2026-10-04
    change: "STORY-188 per-story adversarial pass 1 remediation (F-07): replaced every (planned) Architecture marker and the TBD Stories placeholder with concrete anchors verified against worktree HEAD f33b4337 and Stories: STORY-188; added verifying-test list. No behavioral change."
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

# BC-2.21.017: Unrecognized Job/Ack_Data Function Code Classified `Unrecognized(fc)` — Totality of the FC Match; Empty-Parameter-Block Shared Treatment

## Description

This BC is the terminal fallback arm and totality anchor for the entire Job/Ack_Data
function-code classification group (BC-2.21.010 through BC-2.21.016): any FC byte not
equal to `0xF0`, `0x04`, `0x05`, `0x1A`-`0x1C`, `0x1D`-`0x1F`, `0x28`, or `0x29`
classifies as `S7ClassicFunction::Unrecognized(fc)` — never force-fit to one of the
named variants. This BC also defines the shared empty-parameter-block treatment
referenced by every classification BC in this group (BC-2.21.010 through 016 Edge
Cases): when `param_length == 0`, there is no FC byte to classify at all, which is a
distinct condition from "FC byte present but unrecognized."

## Preconditions

1. `header.rosctr ∈ {Rosctr::Job, Rosctr::AckData}`.
2. The parameter block is bounds-validated per BC-2.21.009.
3. Either (a) `param_length >= 1` and `data[header_len] ∉ {0xF0, 0x04, 0x05, 0x1A,
   0x1B, 0x1C, 0x1D, 0x1E, 0x1F, 0x28, 0x29}`, or (b) `param_length == 0`.

## Postconditions

1. For case (a): the frame is classified `S7ClassicFunction::Unrecognized(fc)` where
   `fc = data[header_len]` — the actual byte value is preserved (not discarded) so a
   future extension or B2 anomaly heuristic can inspect it without re-parsing.
2. For case (b): the frame is classified `S7ClassicFunction::NoParameterBlock` (a
   distinct variant from `Unrecognized`, since "no FC byte present" and "FC byte
   present but unknown" are semantically different conditions — e.g. a bare Ack_Data
   (ROSCTR 0x03) with an empty parameter block, as in STORY-187's `minimal_ack_data_pdu`
   test frame; a Setup Communication Ack_Data does NOT carry `param_length == 0` — the
   canonical frame has `param_length` 8 and classifies as `SetupCommunication`,
   BC-2.21.008/BC-2.21.010).
3. No `Finding` is emitted for either case at the B1 dissection layer — an
   unrecognized-but-otherwise-well-formed FC byte is not itself a malformed-frame
   condition (distinguished from BC-2.21.004/007/008/009's bounds/ROSCTR/length
   safe-reject paths, which do emit T0814). Whether an unrecognized FC value warrants
   an anomaly signal is a B2 policy decision, not a B1 dissection fact.

## Invariants

1. **Totality of the FC match**: every `u8` value at `data[header_len]` (when
   `param_length >= 1`) maps to exactly one `S7ClassicFunction` variant across
   BC-2.21.010 through this BC — no value is unreachable, no value reaches more than
   one arm.
2. **`Unrecognized` vs. `NoParameterBlock` are distinct**: the two "no positive
   classification" outcomes are never conflated into a single catch-all, since B2 may
   need to treat them differently (an empty parameter block is normal for some
   Ack_Data responses; an unrecognized non-empty FC byte is a genuinely novel or
   non-conformant function code).
3. **No force-fit, ever**: this invariant is restated here as the terminal
   confirmation of the principle stated throughout this feature's scope (ADR-014
   Decision 2's protocol_id table, BC-2.20.011's TPDU-type reject, BC-2.21.007's
   ROSCTR reject) — the FC classification layer is the last of three "never force-fit"
   gates in this feature's full dissection chain.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | `data[header_len] == 0x00` | `Unrecognized(0x00)` |
| EC-002 | `data[header_len] == 0x06` (a plausible but unassigned FC value adjacent to Read/Write Var) | `Unrecognized(0x06)` — no proximity-based guessing |
| EC-003 | `param_length == 0` on a bare Ack_Data (ROSCTR 0x03) with an empty parameter block (e.g. STORY-187's `minimal_ack_data_pdu` test frame; NOT a Setup Communication Ack_Data, whose canonical frame has `param_length` 8 → `SetupCommunication`) | `NoParameterBlock` — non-anomalous for an Ack_Data carrying no parameters |
| EC-004 | `data[header_len] == 0xFF` | `Unrecognized(0xFF)` |
| EC-005 | `param_length >= 1` but `header_len + param_length` overflows `usize` or exceeds `data.len()` (caller bounds-check BC-2.21.009 violated — unreachable behind `s7comm_bounds_ok`) | Defensive: `NoParameterBlock` is returned rather than panicking (`checked_add` / `data.get(header_len..end)` in `classify_job_ack_function`); no out-of-bounds slice is ever constructed |

## Canonical Test Vectors

| `param_length` / `data[header_len]` | Expected classification | Category |
|---|---|---|
| `>= 1` / `0x00` | `Unrecognized(0x00)` | edge-case: no force-fit |
| `>= 1` / `0xFF` | `Unrecognized(0xFF)` | edge-case: no force-fit |
| `0` / n/a | `NoParameterBlock` | edge-case: legitimate empty response |

## Verification Properties

| Property | Proof Method |
|----------|--------------|
| The full Job/Ack_Data FC classification match (BC-2.21.010 through this BC) is total and non-overlapping over all 256 `u8` values plus the `param_length == 0` case | VP-052 (proptest P1, status draft) (mirrors VP-046's `classify_frame_format` totality treatment). Evidence in STORY-188: `test_BC_2_21_017_fc_classification_total_over_all_256_values` (deterministic enumeration of all 256 FC bytes under both Job and AckData header lengths against an independent oracle) and `story_188::vp052::proptest_vp052_fc_classification_totality` (2000 sampled cases; a skeleton — the full non-vacuous run is deferred to STORY-194; the Userdata-group half of VP-052 is STORY-189 scope) |

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| Capability Anchor Justification | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 — the totality guarantee that makes the whole `S7ClassicFunction` classification surface exhaustively safe for B2 to consume |
| L2 Domain Invariants | None directly |
| Architecture Module | SS-21 (`src/analyzer/s7comm.rs`: `S7ClassicFunction` :339, `classify_job_ack_function` :385 (`Unrecognized`/`NoParameterBlock` arms)) |
| ADR | ADR-014 Decision 2 (no-force-fit philosophy, applied here at the FC layer) |
| Stories | STORY-188 |
| Feature | feature-s7comm |
| MITRE Techniques | (none — this is a negative-classification/totality contract, not a positive emission surface) |

## Related BCs

- BC-2.21.010 through BC-2.21.016 — composes with (all sibling FC classification arms; this BC is their totality anchor)
- BC-2.20.011 — composes with (the SS-20 TPDU-type no-force-fit precedent this BC extends to the FC layer)

## Architecture Anchors

- `src/analyzer/s7comm.rs:417` — `other => S7ClassicFunction::Unrecognized(other)` terminal arm and the `param_length == 0` / unsliceable-block `NoParameterBlock` early returns of `classify_job_ack_function` (function at `src/analyzer/s7comm.rs:385`, `pub fn classify_job_ack_function(data, header_len, param_length) -> S7ClassicFunction`); variants at :339
- `tests/s7comm_analyzer_tests.rs` `mod story_188` — verifying tests: `test_BC_2_21_017_unrecognized_fc_and_empty_parameter_block`, `test_BC_2_21_017_fc_classification_total_over_all_256_values`, `story_188::vp052::proptest_vp052_fc_classification_totality` (VP-052)

## Story Anchor

STORY-188 (also a formal-hardening re-verification anchor for STORY-194)

## VP Anchors

- VP-052 (proptest P1) — S7comm Function-Code and Userdata-Group Classification
  Totality (Including the Load-Bearing 0x03/0x04/0x07 Group Correction); registered
  F2 INTEGRATE sub-burst per VP-INDEX.md v2.55; traces BC-2.21.017, BC-2.21.019,
  BC-2.21.022, BC-2.21.023

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | none |
| **Deterministic** | yes |
| **Thread safety** | Send + Sync |
| **Overall classification** | pure core — proptest P1 target |
