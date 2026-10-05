---
document_type: behavioral-contract
level: L3
version: "1.1"
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
   present but unknown" are semantically different conditions — a Setup Communication
   Ack, for example, legitimately carries `param_length == 0`).
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
| EC-003 | `param_length == 0` on an Ack_Data response to a Setup Communication request | `NoParameterBlock` — the expected, non-anomalous shape for this response type |
| EC-004 | `data[header_len] == 0xFF` | `Unrecognized(0xFF)` |

## Canonical Test Vectors

| `param_length` / `data[header_len]` | Expected classification | Category |
|---|---|---|
| `>= 1` / `0x00` | `Unrecognized(0x00)` | edge-case: no force-fit |
| `>= 1` / `0xFF` | `Unrecognized(0xFF)` | edge-case: no force-fit |
| `0` / n/a | `NoParameterBlock` | edge-case: legitimate empty response |

## Verification Properties

| Property | Proof Method |
|----------|--------------|
| The full Job/Ack_Data FC classification match (BC-2.21.010 through this BC) is total and non-overlapping over all 256 `u8` values plus the `param_length == 0` case | proptest P1 (mirrors VP-046's `classify_frame_format` totality treatment) — VP-NNN allocation deferred to the F2 INTEGRATE sub-burst |

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

- `src/analyzer/s7comm.rs:385` — `other => S7ClassicFunction::Unrecognized(other)` terminal arm and the `param_length == 0` / unsliceable-block `NoParameterBlock` early returns of `classify_job_ack_function` (`src/analyzer/s7comm.rs:385`, `pub fn classify_job_ack_function(data, header_len, param_length) -> S7ClassicFunction`); variants at :339
- `tests/s7comm_analyzer_tests.rs` `mod story_188` — verifying tests: `test_BC_2_21_017_unrecognized_fc_and_empty_parameter_block`, `test_BC_2_21_017_fc_classification_total_over_all_256_values`, `story_188::vp052::proptest_vp052_fc_classification_totality` (VP-052)

## Story Anchor

STORY-188 (also a formal-hardening re-verification anchor for STORY-194)

## VP Anchors

- VP-052 (proptest P1) — S7comm Function-Code and Userdata-Group Classification
  Totality (Including the Load-Bearing 0x03/0x04/0x07 Group Correction); registered
  F2 INTEGRATE sub-burst per VP-INDEX.md v2.48; traces BC-2.21.017, BC-2.21.019,
  BC-2.21.022, BC-2.21.023

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | none |
| **Deterministic** | yes |
| **Thread safety** | Send + Sync |
| **Overall classification** | pure core — proptest P1 target |
