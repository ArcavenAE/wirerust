---
document_type: story
level: ops
story_id: STORY-188
title: "S7comm Job/Ack_Data Function-Code Classification: Setup Comm, Read/Write Var, Download/Upload Triads, PLC Control, PLC Stop"
epic_id: E-23
version: "1.5"
status: ready
producer: story-writer
timestamp: 2026-09-24T00:00:00Z
phase: f3
traces_to: .factory/specs/prd.md
points: 8
priority: P1
cycle: feature-s7comm
wave: 91
target_module: analyzer/s7comm
subsystems: [SS-21]
estimated_days: null
assumption_validations: []
risk_mitigations: []
tdd_mode: strict
feature_id: feature-s7comm
depends_on: [STORY-187]
blocks: [STORY-189]
behavioral_contracts: [BC-2.21.008, BC-2.21.010, BC-2.21.011, BC-2.21.012, BC-2.21.013, BC-2.21.014, BC-2.21.015, BC-2.21.016, BC-2.21.017]
verification_properties: [VP-051, VP-052, VP-054]  # VP-051 added v1.3: STORY-188 carries the registered Kani harness story_188::vp051_kani::verify_classify_job_ack_function_param_slicing_safe (VP-INDEX v2.55, closes PRF-005)
inputs:
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.008.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.010.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.011.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.012.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.013.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.014.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.015.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.016.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.017.md
  - .factory/specs/architecture/ARCH-INDEX.md
  - docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md
  - .factory/research/s7comm-mitre-ics-tagging.md
  - .factory/research/s7comm-canonical-fc-vectors.md
input-hash: "3ae34a3"
---

> **tdd_mode:** `strict` — full TDD Iron Law enforced.

# STORY-188: S7comm Job/Ack_Data Function-Code Classification

## Narrative

**As a** security analyst using wirerust to inspect classic S7comm traffic,
**I want** the S7comm analyzer to classify every Job/Ack_Data function-code byte into a
named `S7ClassicFunction` variant — Setup Communication, Read/Write Var (with area-code
decode), the Program-Download triad, the Upload triad, PLC Control (with PI-service
string decode), and PLC Stop,
**so that** downstream MITRE technique emission (STORY-191/192) has a correct,
non-force-fit classification surface to key on.

This story is purely classification (part B1 in the source research's terminology) — no
`Finding` is emitted here. It extends `S7commAnalyzer::on_data`'s classic-S7comm branch
(wired in STORY-187) with the function-code match over `data[header_len]`. The per-frame
classification result computed in `dispatch_classic_s7comm` is a **deliberate
classification-only placeholder** (B1 scope): it has no observable output in this story and
is consumed by STORY-191/STORY-192. The only observable `on_data` behavior added here is
the Ack/Ack_Data error-observation record (AC-188-010).

## Behavioral Contracts

| BC ID | Title | Story Role |
|-------|-------|-----------|
| BC-2.21.008 | `parse_s7comm_header` for ROSCTR=Ack (0x02) and Ack_Data (0x03) Requires 12 Bytes (Error Class + Error Code) | Postcondition 4 only (consumption of `error_class`/`error_code` as a bounded analyzer-side record + exact count map, for BOTH Ack and Ack_Data; bounds-gated per EC-009) — re-anchored here from STORY-187 per F-13 ruling, 2026-09-24, rescoped to explicitly include Ack_Data per the canonical-frame holdout ruling (DF-CANONICAL-FRAME-HOLDOUT-001, 2026-09-24); parse/extraction (Postconditions 1-3) remains STORY-187's scope |
| BC-2.21.010 | Job/Ack_Data Function-Code Byte Classifies Setup Communication (FC 0xF0) | Session negotiation |
| BC-2.21.011 | Job/Ack_Data Function-Code Byte Classifies Read Var (FC 0x04) | No area-code decode (read-only, no seeded technique) |
| BC-2.21.012 | Job/Ack_Data Function-Code Byte Classifies Write Var (FC 0x05) With Area-Code Extraction | Primary write indicator |
| BC-2.21.013 | Program-Download Sequence Classified — Request Download (0x1A), Download Block (0x1B), Download Ended (0x1C) | Per-frame classification only; session correlation is STORY-191 |
| BC-2.21.014 | Upload Sequence Classified — Start Upload (0x1D), Upload (0x1E), End Upload (0x1F) — Distinguished From Program Download | Negative-evidence guarantee |
| BC-2.21.015 | PLC Control (FC 0x28) Classified With PI-Service String Decode — `P_PROGRAM`/`_INSE`/`_DELE`/`_GARB`/`_MODU` | Multiplexed function; 5 named services |
| BC-2.21.016 | PLC Stop (FC 0x29) Classified — Dedicated STOP Request, No Service-String Ambiguity | Dedicated, unambiguous STOP |
| BC-2.21.017 | Unrecognized Job/Ack_Data Function Code Classified `Unrecognized(fc)` — Totality of the FC Match; Empty-Parameter-Block Shared Treatment | Terminal fallback arm |

## Acceptance Criteria

### AC-188-001: FC 0xF0 classifies as Setup Communication
(traces to BC-2.21.010 postcondition 1)
- Given `header.rosctr ∈ {Job, AckData}`, a bounds-validated parameter block with
  `param_length >= 1`, and `data[header_len] == 0xF0`
- When the function-code classifier runs
- Then the frame is classified `S7ClassicFunction::SetupCommunication`
- No further parameter-block bytes (protocol version, PDU size negotiation) are
  interpreted beyond FC-level classification (traces to BC-2.21.010 postcondition 2)
- This classification applies identically for `rosctr == Job` and `rosctr == AckData`
  (traces to BC-2.21.010 postcondition 3)
- **Test:** `test_BC_2_21_010_setup_communication_classified`

### AC-188-002: FC 0x04 classifies as Read Var with no area-code decode
(traces to BC-2.21.011 postcondition 1)
- Given `data[header_len] == 0x04`
- When the function-code classifier runs
- Then the frame is classified `S7ClassicFunction::ReadVar`; no area-code or
  item-descriptor decoding is performed (traces to BC-2.21.011 postcondition 2)
- **Test:** `test_BC_2_21_011_read_var_classified_no_area_decode`

### AC-188-003: FC 0x05 classifies as Write Var with first-item area-code extraction
(traces to BC-2.21.012 postcondition 1)
- Given `data[header_len] == 0x05` and a well-formed first address-item descriptor
- When the function-code classifier runs
- Then the frame is classified `S7ClassicFunction::WriteVar(area)` where `area` maps
  `0x80`->`DirectPeripheral`, `0x81`->`Inputs`, `0x82`->`Outputs`, `0x83`->`Markers`,
  `0x84`->`DataBlock`, `0x85`->`InstanceDb`, `0x1C`->`Counters`, `0x1D`->`Timers`, any
  other byte -> `Unrecognized(byte)` (traces to BC-2.21.012 postcondition 2)
- If the item descriptor cannot be read, classification remains `WriteVar` with an
  undetermined area — never a hard reject of the whole frame (traces to BC-2.21.012
  postcondition 3). **Descriptor-length rule (pinned, F-09):** the first item
  descriptor is decoded only when the parameter block is at least 14 bytes (FC + item
  count + 12-byte S7ANY item) AND the syntax-id byte at parameter offset 4 is `0x10`;
  a shorter block or any other syntax id yields the placeholder
  `Unrecognized(0xFF)`; the area byte is read at parameter offset 10 (traces to
  BC-2.21.012 postcondition 3)
- **Accepted residual (EC-001):** the not-decoded placeholder `Unrecognized(0xFF)` is
  indistinguishable from a genuine area byte `0xFF` (both are non-T0835/T0836 areas);
  accepted per F-09 (NIT) and recorded in BC-2.21.012 postcondition 3
- Multi-item parameter blocks are classified using only the first item's area code
  (traces to BC-2.21.012 postcondition 4)
- **Test:** `test_BC_2_21_012_write_var_area_code_extraction`,
  `test_BC_2_21_012_write_var_area_code_exhaustive_over_all_u8` (proptest, BC-2.21.012 Invariant 1; no VP),
  `test_BC_2_21_012_write_var_descriptor_length_boundary_11_12_13_14` (descriptor-length
  boundary: param blocks of 11/12/13/14 bytes)

### AC-188-004: Program-Download triad classified independently and never conflated with Upload
(traces to BC-2.21.013 postcondition 1)
- Given `data[header_len] ∈ {0x1A, 0x1B, 0x1C}`
- When the function-code classifier runs
- Then `0x1A`->`RequestDownload` (traces to BC-2.21.013 postcondition 1), `0x1B`->
  `DownloadBlock` (postcondition 2), `0x1C`->`DownloadEnded` (postcondition 3)
- No block-type/number/content interpretation is performed at this layer; the three FCs
  are never confused with the structurally similar Upload triad (traces to BC-2.21.013
  postcondition 4)
- Correlating a full download session into T0843/T0889/T0821 evidence is explicitly
  deferred to STORY-191 (traces to BC-2.21.013 postcondition 5)
- **Test:** `test_BC_2_21_013_download_triad_classified_independently`; regression guard
  against a collapsed `0x1A..=0x1F` range: `proptest_vp054_download_upload_structural_disjointness`
  (VP-054, shared with AC-188-005)

### AC-188-005: Upload triad classified and behaviorally disjoint from Download
(traces to BC-2.21.014 postcondition 1)
- Given `data[header_len] ∈ {0x1D, 0x1E, 0x1F}`
- When the function-code classifier runs
- Then `0x1D`->`StartUpload`, `0x1E`->`Upload`, `0x1F`->`EndUpload` (traces to
  BC-2.21.014 postconditions 1-3)
- None of the three Upload variants is ever classified as, aliased to, or conflated with
  the Download triad despite the adjacent FC-value ranges (traces to BC-2.21.014
  postcondition 4)
- **Test:** `test_BC_2_21_014_upload_triad_classified_disjoint_from_download` and
  `proptest_vp054_download_upload_structural_disjointness` (VP-054, the anchor for the
  Download/Upload disjointness property; named by BC-2.21.013 and BC-2.21.014 Verification
  Properties; skeleton here, full non-vacuous run in STORY-194). The disjointness the
  tests assert is behavioral (no Download FC yields an Upload variant or vice versa);
  structural arm separation is established by code inspection, not by a test

### AC-188-006: FC 0x28 (PLC Control) classified with PI-service string decode
(traces to BC-2.21.015 postcondition 1)
- Given `data[header_len] == 0x28`
- When the function-code classifier decodes the length-prefixed ASCII service-name
  string
- Then a byte-exact match against `"P_PROGRAM"`, `"_INSE"`, `"_DELE"`, `"_GARB"`,
  `"_MODU"` sets `service` to the corresponding `PlcControlService` variant
  (`ProgramStart`, `BlockActivate`, `BlockDelete`, `MemoryCompress`, `RamToRom`) (traces
  to BC-2.21.015 postcondition 2)
- If the string cannot be read or does not byte-exactly match any of the five, `service`
  is `PlcControlService::Unrecognized` — never a hard reject of the whole frame (traces
  to BC-2.21.015 postcondition 3)
- Bare `FC == 0x28` classification alone is never sufficient for downstream technique
  tagging without the service-string decode (traces to BC-2.21.015 postcondition 4)
- **Test:** `test_BC_2_21_015_plc_control_service_string_decode` (one case per named
  string, plus one for the unrecognized fallback)

### AC-188-007: FC 0x29 classified as PLC Stop by FC byte only (length-prefixed service name not decoded)
(traces to BC-2.21.016 postcondition 1)
- Given `data[header_len] == 0x29`
- When the function-code classifier runs
- Then the frame is classified `S7ClassicFunction::PlcStop`; no sub-operation decode is
  required or attempted (traces to BC-2.21.016 postcondition 2)
- `param_length == 1` (FC byte only, bare `[0x29]`) is still `PlcStop`; `param_length == 0`
  is NOT this case — no FC byte is present, so it is `NoParameterBlock` (traces to
  BC-2.21.016 Edge Case EC-001; BC-2.21.017 postcondition 2)
- PLC Stop's wire layout differs from PLC Control (`0x28`): 5 reserved bytes after the FC
  byte, NO `0xFD` marker and NO `u16` block-argument length, followed by a 1-byte
  service-name length (`0x09`) and the ASCII name `"P_PROGRAM"` (canonical frame
  `29 00 00 00 00 00 09 50 5F 50 52 4F 47 52 41 4D`). The frame therefore DOES carry a
  length-prefixed service name, but it is deliberately NOT decoded: classification is by
  the FC byte ONLY, and the BC-2.21.015 service-string decode is never applied to `0x29`
  (traces to BC-2.21.016 postcondition 3)
- **Tests:** `test_BC_2_21_016_plc_stop_classified`; canonical layout vector covered by
  `story_188::canonical::test_BC_2_21_016_canonical_plc_stop_classified` (AC-188-011)

### AC-188-008: Unrecognized FC and empty parameter block are distinct terminal outcomes
(traces to BC-2.21.017 postcondition 1)
- Given `param_length >= 1` and `data[header_len]` not equal to any named FC value
- When the function-code classifier runs
- Then the frame is classified `S7ClassicFunction::Unrecognized(fc)`, preserving the raw
  byte value (traces to BC-2.21.017 postcondition 1)
- Given `param_length == 0` (e.g. a bare Ack_Data with an empty parameter block such as
  STORY-187's `minimal_ack_data_pdu`; NOT a Setup Communication Ack_Data, whose canonical
  frame has `param_length` 8 and classifies as `SetupCommunication`)
- Then the frame is classified `S7ClassicFunction::NoParameterBlock` — a distinct
  variant from `Unrecognized`, since "no FC byte present" and "FC byte present but
  unknown" are semantically different (traces to BC-2.21.017 postcondition 2)
- Defensive: if `param_length >= 1` but the parameter block cannot be sliced (offset
  arithmetic overflows or exceeds `data.len()`; unreachable behind `s7comm_bounds_ok`),
  the classifier returns `NoParameterBlock` rather than panicking (traces to BC-2.21.017
  Edge Case EC-005)
- No `Finding` is emitted for either case at this layer (traces to BC-2.21.017
  postcondition 3)
- **Test:** `test_BC_2_21_017_unrecognized_fc_and_empty_parameter_block`

### AC-188-009: The full Job/Ack_Data FC classification match is total over all 256 u8 values plus the empty-parameter-block case
(traces to BC-2.21.017 invariant — VP-052 totality obligation)
- Given any `u8` value at `data[header_len]` (when `param_length >= 1`) or the
  `param_length == 0` case
- When the function-code classifier runs
- Then exactly one of BC-2.21.010 through BC-2.21.017's outcomes applies — no value is
  unhandled, no value maps to more than one outcome
- **Test:** `proptest_vp052_fc_classification_totality` (skeleton in this story, full run
  in STORY-194)

### AC-188-010: Ack- and Ack_Data-ROSCTR error_class/error_code are consumed into a bounded analyzer-side record and an exact count map
(traces to BC-2.21.008 postcondition 4)

Surface ratified by human ruling 2026-10-04 (ruling 1: analyzer-side bounded record, **no
stderr / no log output** — ADR-0004 flooding rationale; ruling 2: add an exact
per-`(rosctr, error_class, error_code)` count map and a `pdu_reference` on each
observation). Surface: `S7commAnalyzer::ack_error_observations()` (list of
`S7AckErrorObservation { rosctr, error_class, error_code, pdu_reference }`),
`ack_error_observations_dropped()` (saturating `u64`), `ack_error_counts()`
(`BTreeMap<S7AckErrorKey, u64>`), cap `MAX_S7_ACK_ERROR_OBSERVATIONS = 1024`.

- Given `parse_s7comm_header` returned `Some(header)` with `header.rosctr == Rosctr::Ack`
  and `header.error_class`/`header.error_code` both `Some(byte)` (parsed by STORY-187), and
  BC-2.21.009's bounds check passes
- When `S7commAnalyzer::on_data` processes this Ack-ROSCTR header
- Then one `S7AckErrorObservation` carrying `(rosctr, error_class, error_code,
  pdu_reference)` is appended to the arrival-ordered observation list and the matching
  `S7AckErrorKey` count is incremented — no stderr/log output is produced; no
  function-code classification (Group 3/4, BC-2.21.010 onward) is attempted for an
  Ack-ROSCTR header, since Ack carries no parameter block to classify (traces to
  BC-2.21.008 postcondition 4)
- Given `header.rosctr == Rosctr::AckData` with `error_class`/`error_code` both
  `Some(byte)` (`header_len == 12` per the 2026-09-24 canonical-frame holdout ruling,
  DF-CANONICAL-FRAME-HOLDOUT-001) and the bounds check passing
- When `S7commAnalyzer::on_data` processes this Ack_Data-ROSCTR header
- Then the observation + count are recorded identically to the Ack case, and (if
  `param_length >= 1`) the parameter block at `data[header_len] == data[12]` is also
  classified by `classify_job_ack_function` independently — the error-record obligation
  and the FC classification obligation apply independently and both fire for a
  well-formed Ack_Data frame (traces to BC-2.21.008 postcondition 4). The classification
  result itself is a deliberate classification-only placeholder (no observable output
  here; consumed by STORY-191/192)
- **Bounds gating (EC-009, F-06 ruling):** recording requires BC-2.21.009's bounds check
  to pass. An Ack/Ack_Data whose declared `header_len + param_length + data_length`
  exceeds the available bytes yields the T0814 malformed-header finding and NO
  observation (neither list nor count map) and no FC classification (traces to
  BC-2.21.008 postcondition 4 / EC-009; BC-2.21.009 postcondition 2)
- **Cap behavior:** the observation list holds only the first
  `MAX_S7_ACK_ERROR_OBSERVATIONS` (1024) observations; every observation beyond the cap
  increments `ack_error_observations_dropped` (saturating) instead of being listed. The
  count map keeps counting beyond the cap, so tallies stay exact (bounded by
  construction at <= 2 x 256 x 256 = 131,072 keys, saturating counts) (traces to
  BC-2.21.008 postcondition 4)
- Job (`0x01`) and Userdata (`0x07`) frames carry no error fields: never recorded and
  contribute no count-map key (traces to BC-2.21.008 postcondition 4)
- A zero error class/code (`error_class == 0x00`, `error_code` zero-equivalent) is
  recorded the same as any other value, for either ROSCTR — a zero is a normal
  successful-Ack(_Data) value, not itself flagged or suppressed (BC-2.21.008 Edge Case
  EC-004)
- **Tests:** `test_BC_2_21_008_ack_error_class_code_consumed_and_logged`,
  `test_BC_2_21_008_ack_data_error_class_code_consumed_and_logged`,
  `test_BC_2_21_008_zero_error_class_code_logged_for_ack_and_ack_data`,
  `test_BC_2_21_008_job_frames_record_no_ack_error_observation`,
  `test_BC_2_21_008_userdata_frames_record_no_ack_error_observation`,
  `test_BC_2_21_008_job_frames_contribute_no_histogram_key`,
  `test_BC_2_21_008_ack_error_observations_bounded_by_cap_with_dropped_count`,
  `test_BC_2_21_008_ack_error_counts_exact_for_mixed_frames`,
  `test_BC_2_21_008_ack_error_histogram_counts_beyond_list_cap`,
  `test_BC_2_21_008_ack_error_observation_captures_pdu_reference`,
  `test_BC_2_21_008_bounds_failing_ack_data_records_no_ack_error_observation`
  (test names retain the historical `_logged` suffix; the surface is the bounded record,
  not logging)

### AC-188-011: Canonical public-reference byte vectors exercise each framing invariant (DF-CANONICAL-FRAME-HOLDOUT-001)
(traces to BC-2.21.010 postcondition 1, BC-2.21.011 postcondition 1, BC-2.21.012
postconditions 1-2, BC-2.21.015 postcondition 2, BC-2.21.016 postcondition 3)
- Given a canonical byte sequence taken verbatim from an authoritative/public protocol
  reference for each framing invariant under test — FC at `data[header_len]` (including
  `data[12]` for Ack_Data); Write Var S7ANY syntax id at parameter offset 4 and area byte
  at offset 10; PLC Control layout (`0xFD` marker + `u16` block-arg length + length-prefixed
  service string) vs PLC Stop layout (5 reserved bytes) — with source and byte values
  cited in the test file and in `.factory/research/s7comm-canonical-fc-vectors.md`
  (Setup Communication Ack_Data: BC-2.21.008 Canonical Test Vectors)
- When the frame is parsed, bounds-checked (`s7comm_bounds_ok`) and classified
- Then: Setup Communication Ack_Data (27-byte frame, FC `0xF0` at `data[12]`) ->
  `SetupCommunication` (BC-2.21.010 pc 1); Write Var "DB10 Word18" (area `0x84`) ->
  `WriteVar(DataBlock)` and Write Var "Output0" (area `0x82`) -> `WriteVar(Outputs)`
  (BC-2.21.012 pc 1-2); Read Var DB10 -> `ReadVar` (BC-2.21.011 pc 1); PLC Control
  `"P_PROGRAM"` -> `PlcControl(ProgramStart)` (BC-2.21.015 pc 2); PLC Stop (`0x29`, 5
  reserved bytes + length-prefixed `"P_PROGRAM"`, not decoded) -> `PlcStop` by FC only
  (BC-2.21.016 pc 3)
- Public bytes are used as TEST VECTORS ONLY, never as field-semantics provenance
  (ADR-014 Decision 4, 2026-09-24 permitted-sources note); no Wireshark/Snap7/libnodave
  source is consumed
- **Tests (six):** `story_188::canonical::test_BC_2_21_010_canonical_setup_communication_ack_data_classified`,
  `story_188::canonical::test_BC_2_21_012_canonical_write_var_db_classified`,
  `story_188::canonical::test_BC_2_21_012_canonical_write_var_outputs_classified`,
  `story_188::canonical::test_BC_2_21_011_canonical_read_var_db_classified`,
  `story_188::canonical::test_BC_2_21_015_canonical_plc_control_p_program_classified`,
  `story_188::canonical::test_BC_2_21_016_canonical_plc_stop_classified`
- End-to-end fixture check: `test_BC_2_21_010_fc_classification_fixture_pcap_end_to_end`
  over `tests/fixtures/s7comm-fc-classification.pcap` (synthetic, generated by
  `mk_s7comm_pcap.py`)

## Architecture Mapping

| Component | Module | File | Pure/Effectful |
|-----------|--------|------|---------------|
| `S7ClassicFunction` enum | SS-21 data model | `src/analyzer/s7comm.rs` | N/A (grows across STORY-188/189) |
| `S7AreaCode` enum | SS-21 data model | `src/analyzer/s7comm.rs` | N/A |
| `PlcControlService` enum | SS-21 data model | `src/analyzer/s7comm.rs` | N/A |
| `classify_job_ack_function` | SS-21 classifier | `src/analyzer/s7comm.rs` | Pure (free fn, VP-052/VP-054 target) |
| `S7AckErrorObservation`, `S7AckErrorKey`, `MAX_S7_ACK_ERROR_OBSERVATIONS` | SS-21 data model | `src/analyzer/s7comm.rs` | N/A (plain data; key/observation of the Ack error record) |
| `S7commAnalyzer::on_data` / `record_ack_error_observations` (Ack/Ack_Data error record + count map) | SS-21 effectful shell | `src/analyzer/s7comm.rs` | Effectful (records the already-extracted BC-2.21.008 fields into the bounded observation list, saturating dropped count and exact count map; no stderr/log output; bounds-gated per EC-009; re-anchored from STORY-187 per F-13 ruling) |
| per-frame `classify_job_ack_function` call in `dispatch_classic_s7comm` | SS-21 effectful shell | `src/analyzer/s7comm.rs` | Effectful call site, classification-only placeholder — value unused here, consumed by STORY-191/192 |

Subsystem anchor: SS-21 owns this story's scope — Job/Ack_Data function-code
classification is a core dissection capability of the S7comm analyzer per
ARCH-INDEX.md §SS-21.

## Purity Classification

| Module | Classification | Justification |
|--------|---------------|---------------|
| `classify_job_ack_function` | pure-core | Returns `S7ClassicFunction` by value; no mutation, no finding emission, no I/O |
| `S7ClassicFunction`, `S7AreaCode`, `PlcControlService` | pure-core | Plain data types |
| `S7AckErrorObservation`, `S7AckErrorKey` | pure-core | Plain data types (list element and count-map key) |
| `S7commAnalyzer::on_data` (Ack/Ack_Data error record + count map, AC-188-010) | effectful-shell | Records the already-parsed BC-2.21.008 fields into analyzer-side bounded state; no re-parsing, no `Finding` emission, no stderr/log output |

## VP-052 Proptest Obligation (partial — FC totality sub-part; completed in STORY-189)

**Harness:** `proptest_vp052_fc_classification_totality` (skeleton, this story)
**Method:** proptest
**Priority:** P1

This story covers the Job/Ack_Data function-code totality sub-part of VP-052 (VP status
draft): the deterministic 256-value enumeration test plus a 2000-case proptest skeleton. The
Userdata function-group totality sub-part (including the load-bearing group `0x03`/`0x07`
correction) is covered in STORY-189. Full non-vacuous run of both sub-parts is in
STORY-194.

## VP-051 Kani Harness Obligation (registered v1.3; closes PRF-005)

**Harness:** `story_188::vp051_kani::verify_classify_job_ack_function_param_slicing_safe`
**Method:** Kani
**Priority:** P0 (VP-051; registered in VP-INDEX v2.55 per STORY-188 pass-1 F-08)

Proves VP-051's bounds-ok => no out-of-bounds parameter-block slice obligation by
calling the production `classify_job_ack_function` on inputs passing `s7comm_bounds_ok`
(<= 32-byte symbolic buffer whose header parses and passes the bounds check; the harness
returns early otherwise): it never panics and returns `NoParameterBlock` iff
`param_length == 0` (non-vacuity per DF-KANI-NONVACUITY-001). Safety on inputs that do NOT
pass `s7comm_bounds_ok` is not proven by the harness; it comes from the
`checked_add` / `data.get(header_len..end)` guards in `classify_job_ack_function`
(BC-2.21.017 Edge Case EC-005).
VP-052 remains proptest-only (no Kani harness attributed).

## VP-054 Proptest Obligation

**Harness:** `proptest_vp054_download_upload_structural_disjointness` (anchored in this
story)
**Method:** proptest
**Priority:** P1

Verifies, behaviorally, that no Download-triad value (`0x1A`-`0x1C`) is ever classified as,
aliased to, or conflated with any Upload variant (`0x1D`-`0x1F`) and vice versa (BC-2.21.013
and BC-2.21.014 both name VP-054; VP-INDEX v2.55). That the two triads occupy separate match
arms is a structural property established by code inspection; a behavioral test cannot
assert syntactic arms. Skeleton here (2000 cases over `any::<u8>()`); full non-vacuous run
in STORY-194.

## Tasks

- [ ] Define `pub enum S7ClassicFunction { SetupCommunication, ReadVar,
      WriteVar(S7AreaCode), RequestDownload, DownloadBlock, DownloadEnded, StartUpload,
      Upload, EndUpload, PlcControl(PlcControlService), PlcStop, Unrecognized(u8),
      NoParameterBlock, Userdata(...) }` (the `Userdata(...)` arm is added in STORY-189 —
      define the enum with the arms this story needs; STORY-189 extends it, not
      replaces it)
- [ ] Define `pub enum S7AreaCode { DirectPeripheral, Inputs, Outputs, Markers,
      DataBlock, InstanceDb, Counters, Timers, Unrecognized(u8) }`
- [ ] Define `pub enum PlcControlService { ProgramStart, BlockActivate, BlockDelete,
      MemoryCompress, RamToRom, Unrecognized }`
- [ ] Implement `fn classify_job_ack_function(data: &[u8], header_len: usize,
      param_length: u16) -> S7ClassicFunction`:
  - `param_length == 0` -> `NoParameterBlock` (BC-2.21.017)
  - `data[header_len] == 0xF0` -> `SetupCommunication` (BC-2.21.010)
  - `0x04` -> `ReadVar` (BC-2.21.011)
  - `0x05` -> `WriteVar(area)` with first-item area-code decode (BC-2.21.012)
  - `0x1A`/`0x1B`/`0x1C` -> Download triad (BC-2.21.013)
  - `0x1D`/`0x1E`/`0x1F` -> Upload triad (BC-2.21.014)
  - `0x28` -> `PlcControl(service)` with PI-service string decode (BC-2.21.015)
  - `0x29` -> `PlcStop` (BC-2.21.016)
  - any other byte -> `Unrecognized(fc)` (BC-2.21.017)
- [ ] Call `classify_job_ack_function` from `dispatch_classic_s7comm`'s classic-S7comm
      branch (from STORY-187), for `header.rosctr ∈ {Job, AckData}`, inside the
      BC-2.21.009 bounds-OK branch. This is a **deliberate classification-only
      placeholder** (B1 scope): the per-frame result is not stored or surfaced and has no
      observable effect in this story; it is consumed by STORY-191/STORY-192. Observable
      correctness of classification is verified via direct unit/proptest/Kani calls to
      the pure function, not via `on_data` output
- [ ] Implement the BC-2.21.008 Postcondition 4 consumption obligation (AC-188-010,
      re-anchored from STORY-187 per F-13 ruling, 2026-09-24, rescoped to include
      Ack_Data per DF-CANONICAL-FRAME-HOLDOUT-001; surface ratified by human rulings
      2026-10-04): define `MAX_S7_ACK_ERROR_OBSERVATIONS = 1024`, `S7AckErrorKey`,
      `S7AckErrorObservation` (incl. `pdu_reference`); for every Ack/Ack_Data header
      whose BC-2.21.009 bounds check passes, append an observation (first 1024, then
      saturating dropped count) and increment the exact per-`(rosctr, error_class,
      error_code)` count map (saturating); expose accessors
      `ack_error_observations`/`ack_error_observations_dropped`/`ack_error_counts`; NO
      stderr/log output; Job/Userdata record nothing; bounds-failing frames record
      nothing (EC-009); no re-parsing; for `rosctr == Ack` no FC classification; for
      `AckData` the record fires independently of the placeholder classification call
- [ ] Write the `story_188::canonical` byte-vector tests (six, AC-188-011) citing source
      and byte values, and the VP-051 Kani harness
      `story_188::vp051_kani::verify_classify_job_ack_function_param_slicing_safe`
- [ ] Write `proptest_vp052_fc_classification_totality` and
      `proptest_vp054_download_upload_structural_disjointness` skeletons
- [ ] Write unit tests: one per AC (AC-188-001..011), named `test_BC_2_21_010_*` ..
      `test_BC_2_21_017_*`, plus the eleven `test_BC_2_21_008_*` tests listed under
      AC-188-010, plus `test_BC_2_21_012_write_var_descriptor_length_boundary_11_12_13_14`
- [ ] Extend `tests/fixtures/mk_s7comm_pcap.py` with builders (`read_var_job`,
      `write_var_job`, `simple_fc_job`, `plc_control_job`, `ack_with_error`,
      `ack_data_with_error_and_parameter`, `build_fc_classification_pcap`) and generate the
      new fixture `tests/fixtures/s7comm-fc-classification.pcap` covering every
      STORY-188 function code, plus an Ack-ROSCTR frame and an Ack_Data-ROSCTR frame
      (12-byte header, parameter block at `data[12]`), each with a non-zero
      `error_class`/`error_code` pair (AC-188-010)
- [ ] Verify `cargo test` passes
- [ ] Add a CHANGELOG entry under `[Unreleased] > Added` describing the Job/Ack_Data
      function-code classification and the Ack/Ack_Data error_class/error_code bounded record + count map,
      before creating the PR

## Edge Cases

| ID | Source BC | Description | Expected Behavior |
|----|-----------|-------------|-------------------|
| EC-001 | BC-2.21.012 | Write Var with unrecognized area byte `0xFF` | `WriteVar(Unrecognized(0xFF))` — never force-fit to a named area |
| EC-002 | BC-2.21.013 | `RequestDownload` immediately followed by another `RequestDownload` (no intervening `DownloadEnded`) | Each frame classified independently at this layer; session-level abandon/reset semantics are STORY-191's concern |
| EC-003 | BC-2.21.014 | `0x1A..=0x1F` implemented as a single collapsed range (regression scenario) | MUST NOT occur — `proptest_vp054` regression-guards this explicitly |
| EC-004 | BC-2.21.015 | PI-service parameter block contains `"P_PROGRAM"` followed by additional undecoded trailing bytes | Classified `PlcControl(ProgramStart)`; trailing-byte sub-operation decode (restart disambiguation) is out of this story's scope, handled later per its own dedicated contract |
| EC-005 | BC-2.21.015 | Service-string field truncated (insufficient parameter-block bytes) | `PlcControl(Unrecognized)` — never a hard reject |
| EC-006 | BC-2.21.017 EC-001 | `data[header_len] == 0x00` with `param_length == 1` | `Unrecognized(0x00)` |
| EC-007 | BC-2.21.008 EC-004 (F-13 re-anchor) | Ack- or Ack_Data-ROSCTR header with `error_class == 0x00` and `error_code` zero-equivalent (no error reported) | Recorded (list + count map) the same as any other value, for either ROSCTR — a zero is a normal successful-Ack(_Data) value, not itself flagged or suppressed |
| EC-008 | BC-2.21.008 EC-007/EC-008 (canonical-frame holdout ruling, 2026-09-24) | Ack_Data header with a non-empty parameter block AND non-zero `error_class`/`error_code` (real-world Setup Communication response shape) | The error fields are recorded AND the parameter block at `data[header_len] == data[12]` is classified by `classify_job_ack_function` independently — a populated error-class/code pair never suppresses FC classification |
| EC-009 | BC-2.21.008 EC-009 (F-06 ruling 2026-10-04) | Ack/Ack_Data header parses `Some` (>= 12 bytes) but declared `header_len + param_length + data_length` exceeds the available bytes (BC-2.21.009 bounds check FAILS) | T0814 malformed-header finding emitted; NO error observation recorded (neither list nor count map); no FC classification attempted. Test: `test_BC_2_21_008_bounds_failing_ack_data_records_no_ack_error_observation` |
| EC-010 | BC-2.21.008 postcondition 4 (human ruling 2 2026-10-04) | More than `MAX_S7_ACK_ERROR_OBSERVATIONS` (1024) Ack/Ack_Data observations, or a histogram key repeated beyond the list cap | List holds only the first 1024; `ack_error_observations_dropped` increments (saturating) for each beyond; the `(rosctr, error_class, error_code)` count map keeps counting exactly past the cap. Tests: `..._bounded_by_cap_with_dropped_count`, `..._histogram_counts_beyond_list_cap` |
| EC-011 | BC-2.21.012 postcondition 3 (F-09) | Write Var parameter block shorter than 14 bytes (11/12/13) or syntax id != `0x10` | Placeholder `WriteVar(Unrecognized(0xFF))`; exactly 14 bytes with syntax id `0x10` decodes the area byte at parameter offset 10. Test: `test_BC_2_21_012_write_var_descriptor_length_boundary_11_12_13_14` |
| EC-012 | BC-2.21.017 EC-005 (pass-3 propagation) | `param_length >= 1` but `header_len + param_length` overflows `usize` or exceeds `data.len()` (BC-2.21.009 bounds check violated by the caller; unreachable behind `s7comm_bounds_ok`) | Defensive `NoParameterBlock` returned (via `checked_add` / `data.get`); no panic and no out-of-bounds slice. Covered by the VP-051 Kani harness (not-`NoParameterBlock` iff `param_length > 0` on bounds-ok inputs) |

## Token Budget Estimate

| Context Source | Estimated Tokens |
|---------------|-----------------|
| This story spec | ~9,000 |
| BC files (9 BCs: BC-2.21.008, BC-2.21.010-017) | ~9,300 |
| ADR-014 Decision 5, s7comm-mitre-ics-tagging.md (classification surface only) | ~6,000 |
| src/analyzer/s7comm.rs (from STORY-187) | ~5,000 |
| Test file delta + fixture generator extension | ~4,200 |
| **Total** | **~33,500** |
| Agent context window | 200K for Sonnet |
| **Budget usage** | **~17%** |

## Previous Story Intelligence

| Story | Key Decisions | Patterns Established | Gotchas Discovered |
|-------|--------------|---------------------|-------------------|
| STORY-187 | `parse_s7comm_header` complete; classic-S7comm dispatch branch wired | `header_len` (10 or 12) is the offset where the parameter block begins | Function-code classification MUST NOT emit findings — it is pure classification (part B1); MITRE emission is deferred to STORY-191/192 (part B2), matching the source BCs' own repeated "classification surface only; emission is authored later" framing |

The Download-triad/Upload-triad adjacency (`0x1A`-`0x1F`) is the single highest-risk spot
for an accidental collapsed-range bug in this story — implement as two explicit,
non-overlapping match sub-ranges, never one `0x1A..=0x1F` range with a secondary
disambiguation step.

## Architecture Compliance Rules

Extracted from `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md`:
- **ADR-014 Decision 5**: this story's classification surface feeds T0835/T0836 (Write
  Var area codes), T0843/T0889/T0821 (Download triad), T0858/T0816 (PLC Control/PLC
  Stop) — but this story itself emits no findings; it only establishes the correct,
  non-force-fit classification labels those later stories key on.
- Classification must never guess or force-fit: an unrecognized FC value, an
  undeterminable area code, or an unrecognized PI-service string all have honest
  "unrecognized" fallback variants rather than being coerced into a named outcome.
- Pure/effectful boundary: `classify_job_ack_function` is pure; the `on_data` call site
  that invokes it is the effectful shell (the classification call site is new in this story, added in `dispatch_classic_s7comm`).
- **BC-2.21.008 Postcondition 4 re-anchor (F-13 ruling, human-ratified 2026-09-24;
  rescoped 2026-09-24 to explicitly include Ack_Data per the canonical-frame holdout
  ruling, DF-CANONICAL-FRAME-HOLDOUT-001)**: this story owns the consumption (record) of
  the Ack- AND Ack_Data-ROSCTR `error_class`/`error_code` values that STORY-187 already
  parses and returns from `parse_s7comm_header` (BC-2.21.008 Postconditions 1-3,
  unchanged, still STORY-187's scope, `header_len == 12` for both ROSCTR values). This
  story does NOT re-parse those fields — it only records the `Some(byte)` values
  `parse_s7comm_header` already produced (bounded list + exact count map, no stderr/log
  output per ADR-0004, bounds-gated per BC-2.21.009; human rulings 2026-10-04). For Ack_Data specifically, this error-record
  obligation is independent of (and additional to) this story's normal
  `classify_job_ack_function` parameter-block classification, which now correctly reads
  from `data[12]` rather than the pre-ruling `data[10]` assumption.

## Library & Framework Requirements

| Tool | Version | Purpose |
|------|---------|---------|
| Rust stdlib | 1.91+ (2024 edition) | Match patterns, byte-exact string comparison |
| proptest | 1 (pinned in `Cargo.toml`) | VP-052 (partial)/VP-054 totality and disjointness skeletons |

## File Structure Requirements

| File | Action | Purpose |
|------|--------|---------|
| `src/analyzer/s7comm.rs` | MODIFY | Add `S7ClassicFunction`, `S7AreaCode`, `PlcControlService`, `classify_job_ack_function`; add `S7AckErrorKey`/`S7AckErrorObservation`/`MAX_S7_ACK_ERROR_OBSERVATIONS`; call the classifier (classification-only placeholder) from `dispatch_classic_s7comm`; add Ack AND Ack_Data `error_class`/`error_code` bounded record + count map + accessors (BC-2.21.008 Postcondition 4, AC-188-010) |
| `tests/s7comm_analyzer_tests.rs` | MODIFY | Add `mod story_188`: BC-2.21.008 (AC-188-010, eleven tests) + BC-2.21.010-017 unit tests + `canonical` byte-vector tests (AC-188-011) + `area_exhaustive`/`vp052`/`vp054` proptests + `vp051_kani` harness |
| `tests/fixtures/mk_s7comm_pcap.py` | MODIFY | Add builders for Read/Write Var, Download/Upload triad, PLC Control, PLC Stop, Ack and Ack_Data-with-error frames; `build_fc_classification_pcap` |
| `tests/fixtures/s7comm-fc-classification.pcap` | CREATE | Synthetic fixture covering every STORY-188 function code (generated by `mk_s7comm_pcap.py`) |

## Forbidden Dependencies

- Wireshark, Snap7, libnodave source, and any `s7`/`s7-comm`/`s7-client` crate — banned/
  avoid per ADR-014 Decision 4
- `classify_job_ack_function` MUST NOT call `emit_finding` or access `S7commFlowState` —
  it is a pure classification fn; emission lands in STORY-191/192

## Changelog

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.5 | 2026-10-05 | story-writer | STORY-188 per-story adversarial PASS-3 remediation (P3-F-07 plus propagation of BC amendments BC-2.21.008 v1.11, BC-2.21.009 v1.10, BC-2.21.010..017 v1.3). P3-F-07 (NIT) — AC-188-011 now notes the Setup Communication Ack_Data vector is cited in BC-2.21.008 Canonical Test Vectors (not in the research doc, which holds Write Var, PLC Control, PLC Stop, Read Var only). Propagation — AC-188-007 retitled/reworded: PLC Stop carries a length-prefixed service name (`"P_PROGRAM"`) in a different layout, deliberately not decoded (BC-2.21.016 v1.3); AC-188-011 PLC Stop bullet aligned; AC-188-004/005 and the VP-054 section name VP-054 as the anchored Download/Upload disjointness property (BC-013/014 now name it) and state disjointness is behavioral, arm separation by inspection; VP-051 section scope reworded to inputs passing `s7comm_bounds_ok` (unchecked-input safety from `data.get`/`checked_add` guards); VP-052 section notes skeleton + full run deferred to STORY-194; AC-188-008 gains the BC-2.21.017 EC-005 defensive `NoParameterBlock` bullet and new EC-012. Points unchanged (8). |
| 1.4 | 2026-10-04 | story-writer | STORY-188 per-story adversarial PASS-2 remediation (P2-F-03, P2-F-05, P2-F-06, P2-F-09; propagation of BC v1.2/v1.9 amendments incl. P2-F-01, P2-F-02, P2-F-08). P2-F-06 — body BC table titles for BC-2.21.010..017 now verbatim BC H1 titles (BC-2.21.008 row verified unchanged). P2-F-03 — `test_BC_2_21_012_write_var_area_code_exhaustive_over_all_u8` relabeled "(proptest, BC-2.21.012 Invariant 1; no VP)" (VP-052 is FC/Userdata-group totality only). P2-F-09 — Purity Classification note corrected: the classification call site is new in `dispatch_classic_s7comm`, not unchanged from STORY-187. P2-F-05 — `test_BC_2_21_008_userdata_frames_record_no_ack_error_observation` added to AC-188-010 Tests (now eleven; Job/Userdata never-recorded clause), Tasks and File Structure counts updated. P2-F-01 — AC-188-007 gains BC-2.21.016 EC-001 (`param_length == 1` FC-only -> `PlcStop`; `param_length == 0` -> `NoParameterBlock`). P2-F-02 — AC-188-008 `param_length == 0` example is a bare Ack_Data (NOT Setup Communication, canonical `param_length` 8). P2-F-08 — BC-2.21.013 "adjacent in FC-space" wording confirmed in AC-188-005. Points unchanged (8). |
| 1.3 | 2026-10-04 | story-writer | STORY-188 per-story adversarial PASS-1 remediation (F-01, F-02, F-03, F-05, F-06, F-07, F-09), human rulings 2026-10-04 recorded: (1) AC-188-010 surface = analyzer-side bounded record, NO stderr/log output (ADR-0004 flooding rationale); (2) added exact per-`(rosctr, error_class, error_code)` count map and `pdu_reference` on each observation. Changes: F-05/F-06 — AC-188-010 rewritten (removed "logged/surfaced via structured logging or diagnostic surface"; now bounded list cap 1024 + saturating dropped count + count map beyond cap + `pdu_reference`; bounds-gated per EC-009; ten tests listed); EC-009/EC-010 added. F-01 (DF-CANONICAL-FRAME-HOLDOUT-001) — new AC-188-011 citing the six `story_188::canonical::*` tests and `.factory/research/s7comm-canonical-fc-vectors.md`, public bytes as test vectors only (ADR-014 Decision 4). F-02/F-09 — AC-188-003 gains the descriptor-length rule (>=14 bytes, syntax id 0x10 at offset 4, area at offset 10) + boundary test `test_BC_2_21_012_write_var_descriptor_length_boundary_11_12_13_14` + accepted 0xFF collision (orchestrator decision: accepted residual); EC-011 added. AC-188-007 notes PLC Stop wire layout differs, FC-only classification (BC-2.21.016 PC3). F-03 — Task "Wire classify_job_ack_function into on_data" rewritten as a deliberate classification-only placeholder consumed by STORY-191/192 (no observable wiring claimed); Narrative, Architecture Mapping, Purity Classification updated. F-07 — Architecture Mapping/Purity rows (record + count map + `S7AckErrorKey`/`S7AckErrorObservation`), File Structure (new fixture `tests/fixtures/s7comm-fc-classification.pcap`, `mk_s7comm_pcap.py` builders), Tasks, Edge Cases, Token Budget (~33,500, ~17%; BC count unchanged at 9). VP-051 added to `verification_properties` with a VP-051 Kani Harness Obligation section: `story_188::vp051_kani::verify_classify_job_ack_function_param_slicing_safe` (VP-INDEX v2.55, closes PRF-005); `s7comm-canonical-fc-vectors.md` added to `inputs`. Points unchanged (8). |
| 1.2 | 2026-09-24 | story-writer | Synced this story to the STORY-187 canonical-frame holdout human ruling (DF-CANONICAL-FRAME-HOLDOUT-001, 2026-09-24): BC-2.21.008's Postcondition 4 consumption/logging obligation (already re-anchored to this story in v1.1) is now rescoped to explicitly cover BOTH ROSCTR `0x02` (Ack) and `0x03` (Ack_Data) — Ack_Data was previously (incorrectly, per v1.0/v1.1) assumed to carry no error fields of its own, but the amended BC-2.21.008 v1.2 shows Ack_Data also carries `error_class`/`error_code` at `data[10..12]` with `header_len == 12`, and its parameter block (still classified normally by `classify_job_ack_function`) begins at `data[12]`, not `data[10]`. Changes: (1) the Behavioral Contracts table's BC-2.21.008 title row updated verbatim to the current BC H1 ("...for ROSCTR=Ack (0x02) and Ack_Data (0x03)..."); (2) AC-188-010 extended with a parallel Ack_Data given/when/then and a note that the error-logging and FC-classification obligations for Ack_Data fire independently of each other; (3) a new test named `test_BC_2_21_008_ack_data_error_class_code_consumed_and_logged`; (4) Tasks, Architecture Mapping/Purity Classification note, Architecture Compliance Rules, File Structure Requirements, and the fixture-generator task updated to mention Ack_Data alongside Ack; (5) new Edge Case EC-008 (Ack_Data with a populated parameter block AND non-zero error fields — both obligations fire); EC-007 generalized to "Ack- or Ack_Data-ROSCTR" wording. Verified: no fixture, worked example, or AC in this story hardcodes an Ack_Data parameter-block offset of `data[10]` — the classifier signature (`classify_job_ack_function(data, header_len, param_length)`) and every AC already parametrize on `data[header_len]` generically, so no other change was required for the offset correction itself. Points unchanged (a second small logging AC on an already-generic classifier, not judged large enough to require a bump). |
| 1.1 | 2026-09-24 | story-writer | Propagated F-13 human ruling (STORY-187 per-story adversarial pass 1, human-ratified 2026-09-24; BC-2.21.008 v1.1 Postcondition 4 and Traceability/Story Anchor sections) into this story: BC-2.21.008 added to `behavioral_contracts`/`inputs` frontmatter and the body Behavioral Contracts table (Postcondition 4 only — parse/extraction, Postconditions 1-3, remains STORY-187's scope). New AC-188-010 (traces to BC-2.21.008 postcondition 4): `S7commAnalyzer::on_data` consumes/logs the already-parsed Ack-ROSCTR `error_class`/`error_code` values; a zero error class/code is logged the same as any other value (BC-2.21.008 EC-004). Added Architecture Mapping/Purity Classification rows for the Ack-consumption effectful-shell behavior, a Tasks entry, a named unit test (`test_BC_2_21_008_ack_error_class_code_consumed_and_logged`), Edge Case EC-007, a Token Budget BC-count update (8->9 BCs), an Architecture Compliance Rules note on the re-anchor, and a File Structure Requirements update. Points unchanged pending story-writer's points-change recommendation (see handback report) — the added scope is a single small logging AC, not judged large enough on its own to require a bump, but should be considered alongside STORY-187's larger F-01/F-02/F-12 scope growth when wave/points are next reviewed. |
| 1.0 | 2026-09-06 | story-writer | Initial authorship — Job/Ack_Data function-code classification (Setup Comm, Read/Write Var, Download/Upload triads, PLC Control, PLC Stop), VP-052 (partial)/VP-054 skeletons, AC-188-001..009. |
