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
    change: "STORY-188 pass-3 P3-F-03 (MINOR): Description and Invariant 1 no longer claim PLC Stop 'carries no multiplexed service-string field' (false — the canonical vector, research §3, carries FC + 5 reserved bytes + 1-byte length 0x09 + "P_PROGRAM"). Reworded: PLC Stop carries a length-prefixed service name (observed "P_PROGRAM") in a different layout from PLC Control (5 reserved bytes, no 0xFD, no u16 block-argument length); it is deliberately NOT decoded because the FC byte alone identifies the operation. H1 unchanged (immutable-title rule). Postcondition 3 and canonical vector row now state the length-prefixed name; consistent with BC-2.21.015 Invariant 3."
  - version: "1.2"
    date: 2026-10-04
    change: "STORY-188 pass-2 P2-F-07 (NIT): arm anchor re-cited from :385 (fn signature) to the match-arm line :416; function itself stays :385. P2-F-01 (MINOR): EC-001 reworded - FC-byte-only case is param_length == 1 (cites test_BC_2_21_016_plc_stop_classified); param_length == 0 is NoParameterBlock per BC-2.21.017 PC2 and code."
  - version: "1.1"
    date: 2026-10-04
    change: "STORY-188 per-story adversarial pass 1 remediation (F-07): replaced every (planned) Architecture marker and the TBD Stories placeholder with concrete anchors verified against worktree HEAD f33b4337 and Stories: STORY-188; added verifying-test list. Postcondition 3 added recording that PLC Stop wire layout differs from PLC Control (5 reserved bytes, no 0xFD, no u16 block-arg length) and classification is by FC only. Canonical-vector test cited under its corrected name (`test_BC_2_21_016_canonical_plc_stop_classified`, formerly mis-named for BC-2.21.014 — test-name defect fixed)."
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

# BC-2.21.016: PLC Stop (FC 0x29) Classified — Dedicated STOP Request, No Service-String Ambiguity

## Description

`FC == 0x29` is a dedicated PLC Stop request. Unlike `0x28` (BC-2.21.015), where the
service name selects the operation, the FC byte alone fully identifies the operation.
The observed wire frame does carry a length-prefixed service name (canonical vector:
`"P_PROGRAM"`) after the FC byte, but in a different layout from PLC Control (5 reserved
bytes, no `0xFD` marker, no `u16` block-argument length), and that name is deliberately
NOT decoded — it carries no additional signal beyond the FC byte. This BC classifies
`FC == 0x29` as `S7ClassicFunction::PlcStop` directly, with no further parameter-block
decode. It is packaged separately from BC-2.21.015 specifically because it does **not**
share `0x28`'s service-name ambiguity — conflating the two into one BC would
understate the material difference in decode complexity ADR-014 flags for `0x28`
alone, and would wrongly apply `0x28`'s layout to `0x29`.

## Preconditions

1. `header.rosctr ∈ {Rosctr::Job, Rosctr::AckData}`.
2. The parameter block is bounds-validated per BC-2.21.009 and `param_length >= 1`.
3. `data[header_len] == 0x29`.

## Postconditions

1. The frame is classified `S7ClassicFunction::PlcStop`.
2. No sub-operation decode is required or attempted — `0x29` has exactly one meaning.
3. **Wire layout differs from PLC Control (`0x28`)** (canonical vector, `.factory/research/s7comm-canonical-fc-vectors.md` §3, DF-CANONICAL-FRAME-HOLDOUT-001): a PLC Stop parameter block carries 5 reserved bytes after the FC byte, NO `0xFD` marker and NO `u16` block-argument length (the layout BC-2.21.015's service decode relies on), followed by a 1-byte service-name length (`0x09`) and the ASCII name `"P_PROGRAM"` (canonical frame: `29 00 00 00 00 00 09 50 5F 50 52 4F 47 52 41 4D`, 16 bytes). Classification is therefore **by FC byte only**; the length-prefixed name is not decoded, and the BC-2.21.015 `0x28` layout/decode is never applied to `0x29`.

## Invariants

1. **No ambiguity, no decode**: `PlcStop` is the simplest classification arm in this
   group — a direct FC-to-variant mapping. The frame does carry a length-prefixed
   service name (observed `"P_PROGRAM"`) in a layout different from `0x28`'s, but the
   FC byte alone identifies the operation, so that name is deliberately not decoded
   (and `0x28`'s decode is never applied to `0x29`); there is no service-name
   ambiguity to resolve, unlike `0x28`.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | `param_length == 1` (the FC byte only; no bytes follow it in the parameter block) | Still classified `PlcStop` — the FC byte alone is sufficient; an empty remainder is expected, not anomalous (test: `test_BC_2_21_016_plc_stop_classified`, bare `[0x29]`). `param_length == 0` is NOT this case: no FC byte is present, so it is `NoParameterBlock` (BC-2.21.017 Postcondition 2) |

## Canonical Test Vectors

| `data[header_len]` | Expected classification | Category |
|---|---|---|
| `0x29` | `PlcStop` | happy-path |
| `0x29 00 00 00 00 00 09 50 5F 50 52 4F 47 52 41 4D` (canonical PLC Stop parameter block, 16 bytes; research §3) | `PlcStop` (service name not decoded) | canonical vector (`story_188::canonical::test_BC_2_21_016_canonical_plc_stop_classified`) |

## Verification Properties

(No independent VP-NNN — single-arm unit test; totality covered by the shared match
anchored to BC-2.21.017.)

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| Capability Anchor Justification | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| L2 Domain Invariants | None directly |
| Architecture Module | SS-21 (`src/analyzer/s7comm.rs`: `S7ClassicFunction::PlcStop` :339, `classify_job_ack_function` :385) |
| ADR | ADR-014 Decision 5 |
| Stories | STORY-188 |
| Feature | feature-s7comm |
| MITRE Techniques | T0858 (Change Operating Mode, run→stop) — **classification surface only; emission is authored in part B2** |

## Related BCs

- BC-2.21.015 — composes with (the `0x28` sibling operating-mode-change function, contrasted for decode complexity)

## Architecture Anchors

- `src/analyzer/s7comm.rs:416` — `0x29 => S7ClassicFunction::PlcStop` arm of `classify_job_ack_function` (function at `src/analyzer/s7comm.rs:385`, `pub fn classify_job_ack_function(data, header_len, param_length) -> S7ClassicFunction`); deliberately does NOT call `decode_plc_control_service` (:462) because the layouts differ; the `PlcStop` variant of the `S7ClassicFunction` enum at :339 carries no payload
- `tests/s7comm_analyzer_tests.rs` `mod story_188` — verifying tests: `test_BC_2_21_016_plc_stop_classified`, `story_188::canonical::test_BC_2_21_016_canonical_plc_stop_classified` (canonical PLC Stop Job — framing layout per `.factory/research/s7comm-canonical-fc-vectors.md` §3, DF-CANONICAL-FRAME-HOLDOUT-001)

## Story Anchor

STORY-188

## VP Anchors

(None dedicated.)

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | none |
| **Deterministic** | yes |
| **Thread safety** | Send + Sync |
| **Overall classification** | pure core |
