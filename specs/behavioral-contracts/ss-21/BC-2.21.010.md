---
document_type: behavioral-contract
level: L3
version: "1.4"
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
  - version: "1.4"
    date: 2026-10-05
    change: "STORY-188 pass-4 P4-F-03 (MINOR) sweep: frontmatter YAML validity - unescaped inner double quotes in a prior modified-entry change string (which made the frontmatter unparseable) converted to single quotes; no semantic change. No change to Preconditions/Postconditions/Invariants/Edge Cases."
  - version: "1.3"
    date: 2026-10-05
    change: "STORY-188 pass-3 P3-F-05 (NIT): EC-001 'Unrecognized-adjacent 'no function code present' case' now names `S7ClassicFunction::NoParameterBlock` (per BC-2.21.017 PC2). Pass-3 sweep: Job/AckData call-site anchors corrected (~1084/~1091 -> ~1079/~1087, verified at worktree HEAD e24c6f7d); VP-052 citation clarified (VP-052 source_bc is BC-2.21.017/019/022/023; this BC is covered only via the shared FC-match totality)."
  - version: "1.2"
    date: 2026-10-04
    change: "STORY-188 pass-2 P2-F-07 (NIT): arm anchor re-cited from :385 (fn signature) to the match-arm line :403; function itself stays :385."
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

# BC-2.21.010: Job/Ack_Data Function-Code Byte Classifies Setup Communication (FC 0xF0)

## Description

For `rosctr ∈ {Job, AckData}` (Postcondition of BC-2.21.006/007's ROSCTR gate) with a
bounds-validated, non-empty parameter block (BC-2.21.009), the first byte of the
parameter block (`data[header_len]`) is the S7comm function code (FC). This BC and
BC-2.21.011 through BC-2.21.017 jointly define `S7ClassicFunction`, the classification
label enum that part B2 maps MITRE ATT&CK for ICS techniques onto. `FC == 0xF0`
classifies as `S7ClassicFunction::SetupCommunication` — the session-negotiation
function exchanged once per S7comm session (analogous to IEC-104's STARTDT, but
carrying protocol-version/PDU-size negotiation parameters rather than a bare control
function). Setup Communication applies symmetrically to both Job (request) and
Ack_Data (response) ROSCTR — the same FC byte identifies the operation in both
directions of the exchange (this BC's modeling decision, stated once here and inherited
by BC-2.21.011 through BC-2.21.016).

## Preconditions

1. `header.rosctr ∈ {Rosctr::Job, Rosctr::AckData}`.
2. The parameter block is bounds-validated per BC-2.21.009 and `param_length >= 1`.
3. `data[header_len] == 0xF0`.

## Postconditions

1. The frame is classified `S7ClassicFunction::SetupCommunication`.
2. No further parameter-block bytes (protocol version, max AMQ, PDU size negotiation
   fields) are interpreted by this BC — Setup Communication's negotiated parameters
   carry no MITRE-technique-relevant signal per the source research
   (`.factory/research/s7comm-mitre-ics-tagging.md`) and are out of B1 dissection
   scope beyond FC-level classification.
3. This classification applies identically whether `header.rosctr == Job` (request) or
   `header.rosctr == AckData` (response) — `S7ClassicFunction::SetupCommunication`
   carries no request/response discriminant of its own; direction is available
   separately from the flow's `c2s`/`s2c` delivery direction if a future consumer
   needs it.

## Invariants

1. **Symmetric Job/Ack_Data FC semantics**: this BC establishes the modeling
   convention — stated once, applying to every FC value in this group — that the same
   `S7ClassicFunction` variant set covers both request and response PDUs, since the FC
   byte position and meaning do not change between the two ROSCTR values.
2. **No force-fit**: FC `0xF0` maps to exactly one variant; no other FC value maps to
   `SetupCommunication`.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | `param_length == 0` (no FC byte present at all) — see BC-2.21.017 Edge Case for the shared empty-parameter-block treatment | Classified `S7ClassicFunction::NoParameterBlock` (a variant distinct from `Unrecognized(fc)`, BC-2.21.017 Postcondition 2), defined once in BC-2.21.017 and referenced by every classification BC in this group |
| EC-002 | `data[header_len] == 0xF0` but `header.rosctr == Userdata` | Not reachable — Userdata's parameter block has a structurally different layout (BC-2.21.018); this precondition's ROSCTR gate prevents cross-interpretation |

## Canonical Test Vectors

| `header.rosctr` / `data[header_len]` | Expected classification | Category |
|---|---|---|
| `Job` / `0xF0` | `SetupCommunication` | happy-path: request |
| `AckData` / `0xF0` | `SetupCommunication` | happy-path: response |

## Verification Properties

(No independent VP-NNN — classification-label mapping verified by table-driven unit
tests, mirroring IEC-104's function-code/TypeID classification precedent, BC-2.19.019
et al. The proptest P1 totality obligation for the full FC match is anchored to
BC-2.21.017, the terminal `Unrecognized` fallback arm.)

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| Capability Anchor Justification | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 — function-code classification is the core dissection behavior CAP-21's description names ("full S7comm PDU dissection (function codes...)") |
| L2 Domain Invariants | None directly |
| Architecture Module | SS-21 (`src/analyzer/s7comm.rs`: `S7ClassicFunction` :339, `classify_job_ack_function` :385) |
| ADR | ADR-014 Decision 5 (function-code table, informational for classification — MITRE emission is B2 scope) |
| Stories | STORY-188 |
| Feature | feature-s7comm |
| MITRE Techniques | (none emitted by this BC — Setup Communication carries no MITRE mapping per `.factory/research/s7comm-mitre-ics-tagging.md` §S7comm wire-field basis, "session negotiation; scan/flood context" is a T0814/T0846 *aggregate* signal, not a per-PDU FC 0xF0 tag; B2 decides whether/how to use Setup Communication frequency as burst-detection evidence) |

## Related BCs

- BC-2.21.006 — depends on (ROSCTR/param_length this classification reads)
- BC-2.21.009 — depends on (bounds check precedes FC-byte access)
- BC-2.21.011 through BC-2.21.017 — composes with (sibling FC classification arms of the same match)

## Architecture Anchors

- `src/analyzer/s7comm.rs:403` — `classify_job_ack_function` (function at `src/analyzer/s7comm.rs:385`, `pub fn classify_job_ack_function(data, header_len, param_length) -> S7ClassicFunction`), `0xF0 => S7ClassicFunction::SetupCommunication` arm; called from `dispatch_classic_s7comm` (:1033) for `Rosctr::Job` (~1079) and `Rosctr::AckData` (~1087) after the BC-2.21.009 bounds check
- `src/analyzer/s7comm.rs:339` — `pub enum S7ClassicFunction { SetupCommunication, ReadVar, WriteVar(S7AreaCode), RequestDownload, DownloadBlock, DownloadEnded, StartUpload, Upload, EndUpload, PlcControl(PlcControlService), PlcStop, Unrecognized(u8), NoParameterBlock }` (the classification surface part B2 maps MITRE techniques onto; `NoParameterBlock` added for BC-2.21.017)
- `tests/s7comm_analyzer_tests.rs` `mod story_188` — verifying tests: `test_BC_2_21_010_setup_communication_classified`, `test_BC_2_21_010_fc_classification_fixture_pcap_end_to_end`, `story_188::canonical::test_BC_2_21_010_canonical_setup_communication_ack_data_classified` (canonical Setup Communication Ack_Data vector, cnblogs source per BC-2.21.008 Canonical Test Vectors), plus `story_188::vp052::proptest_vp052_fc_classification_totality` (exercises this arm as part of the shared FC-match totality anchored to BC-2.21.017; VP-INDEX v2.55 VP-052 `source_bc` lists BC-2.21.017/019/022/023, not this BC)
- `.factory/research/s7comm-mitre-ics-tagging.md` §S7comm wire-field basis — FC table source

## Story Anchor

STORY-188

## VP Anchors

(None dedicated — covered by BC-2.21.017's totality proptest.)

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | none |
| **Deterministic** | yes |
| **Thread safety** | Send + Sync |
| **Overall classification** | pure core |
