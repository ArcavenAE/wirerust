---
document_type: story
level: ops
story_id: STORY-188
title: "S7comm Job/Ack_Data Function-Code Classification: Setup Comm, Read/Write Var, Download/Upload Triads, PLC Control, PLC Stop"
epic_id: E-23
version: "1.2"
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
verification_properties: [VP-052, VP-054]
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
input-hash: "6c507d9"
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
(wired in STORY-187) with the function-code match over `data[header_len]`.

## Behavioral Contracts

| BC ID | Title | Story Role |
|-------|-------|-----------|
| BC-2.21.008 | `parse_s7comm_header` for ROSCTR=Ack (0x02) and Ack_Data (0x03) Requires 12 Bytes (Error Class + Error Code) | Postcondition 4 only (consumption/logging of `error_class`/`error_code`, for BOTH Ack and Ack_Data) — re-anchored here from STORY-187 per F-13 ruling, 2026-09-24, rescoped to explicitly include Ack_Data per the canonical-frame holdout ruling (DF-CANONICAL-FRAME-HOLDOUT-001, 2026-09-24); parse/extraction (Postconditions 1-3) remains STORY-187's scope |
| BC-2.21.010 | Setup Communication (FC 0xF0) Classified | Session negotiation |
| BC-2.21.011 | Read Var (FC 0x04) Classified | No area-code decode (read-only, no seeded technique) |
| BC-2.21.012 | Write Var (FC 0x05) Classified With Area-Code Extraction | Primary write indicator |
| BC-2.21.013 | Program-Download Triad Classified (0x1A/0x1B/0x1C) | Per-frame classification only; session correlation is STORY-191 |
| BC-2.21.014 | Upload Triad Classified (0x1D/0x1E/0x1F), Distinguished From Download | Negative-evidence guarantee |
| BC-2.21.015 | PLC Control (FC 0x28) Classified With PI-Service String Decode | Multiplexed function; 5 named services |
| BC-2.21.016 | PLC Stop (FC 0x29) Classified | Dedicated, unambiguous STOP |
| BC-2.21.017 | Unrecognized FC / Empty Parameter Block — Totality Anchor | Terminal fallback arm |

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
  postcondition 3)
- Multi-item parameter blocks are classified using only the first item's area code
  (traces to BC-2.21.012 postcondition 4)
- **Test:** `test_BC_2_21_012_write_var_area_code_extraction`,
  `test_BC_2_21_012_write_var_area_code_exhaustive_over_all_u8` (proptest, VP-052)

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
- **Test:** `test_BC_2_21_013_download_triad_classified_independently`

### AC-188-005: Upload triad classified and structurally disjoint from Download
(traces to BC-2.21.014 postcondition 1)
- Given `data[header_len] ∈ {0x1D, 0x1E, 0x1F}`
- When the function-code classifier runs
- Then `0x1D`->`StartUpload`, `0x1E`->`Upload`, `0x1F`->`EndUpload` (traces to
  BC-2.21.014 postconditions 1-3)
- None of the three Upload variants is ever classified as, aliased to, or conflated with
  the Download triad despite the adjacent FC-value ranges (traces to BC-2.21.014
  postcondition 4)
- **Test:** `test_BC_2_21_014_upload_triad_classified_disjoint_from_download` and
  `proptest_vp054_download_upload_structural_disjointness`

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

### AC-188-007: FC 0x29 classified as PLC Stop with no further decode
(traces to BC-2.21.016 postcondition 1)
- Given `data[header_len] == 0x29`
- When the function-code classifier runs
- Then the frame is classified `S7ClassicFunction::PlcStop`; no sub-operation decode is
  required or attempted (traces to BC-2.21.016 postcondition 2)
- **Test:** `test_BC_2_21_016_plc_stop_classified`

### AC-188-008: Unrecognized FC and empty parameter block are distinct terminal outcomes
(traces to BC-2.21.017 postcondition 1)
- Given `param_length >= 1` and `data[header_len]` not equal to any named FC value
- When the function-code classifier runs
- Then the frame is classified `S7ClassicFunction::Unrecognized(fc)`, preserving the raw
  byte value (traces to BC-2.21.017 postcondition 1)
- Given `param_length == 0`
- Then the frame is classified `S7ClassicFunction::NoParameterBlock` — a distinct
  variant from `Unrecognized`, since "no FC byte present" and "FC byte present but
  unknown" are semantically different (traces to BC-2.21.017 postcondition 2)
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

### AC-188-010: Ack- and Ack_Data-ROSCTR error_class/error_code are consumed and logged
(traces to BC-2.21.008 postcondition 4)
- Given `parse_s7comm_header` returned `Some(header)` with `header.rosctr == Rosctr::Ack`
  and `header.error_class`/`header.error_code` both `Some(byte)` (parsed by STORY-187)
- When `S7commAnalyzer::on_data` processes this Ack-ROSCTR header
- Then the observed `error_class`/`error_code` values are logged/surfaced by the
  analyzer (e.g. via its structured logging or diagnostic surface) — this is the
  consumption obligation BC-2.21.008 Postcondition 4 re-anchors to this story (F-13
  ruling, human-ratified 2026-09-24); no function-code classification (Group 3/4,
  BC-2.21.010 onward) is attempted for an Ack-ROSCTR header, since Ack carries no
  parameter block to classify
- Given `parse_s7comm_header` returned `Some(header)` with `header.rosctr ==
  Rosctr::AckData` and `header.error_class`/`header.error_code` both `Some(byte)`
  (parsed by STORY-187, `header_len == 12` per the 2026-09-24 canonical-frame holdout
  ruling, DF-CANONICAL-FRAME-HOLDOUT-001)
- When `S7commAnalyzer::on_data` processes this Ack_Data-ROSCTR header
- Then the observed `error_class`/`error_code` values are logged/surfaced by the
  analyzer identically to the Ack case — this obligation was rescoped 2026-09-24 to
  explicitly include Ack_Data (BC-2.21.008 v1.2 Postcondition 4), since Ack_Data was
  previously (incorrectly) assumed to carry no error fields of its own; unlike Ack,
  Ack_Data's parameter block (if `param_length >= 1`, starting at `data[header_len] ==
  data[12]`) IS still classified by `classify_job_ack_function` (BC-2.21.010 onward) —
  the error-class/code logging obligation and the FC classification obligation apply
  independently and both fire for a well-formed Ack_Data frame
- A zero error class/code (`error_class == 0x00`, `error_code` zero-equivalent) is
  logged the same as any other value, for either ROSCTR — a zero is a normal
  successful-Ack(_Data) value, not itself flagged or suppressed (BC-2.21.008 Edge Case
  EC-004)
- **Tests:** `test_BC_2_21_008_ack_error_class_code_consumed_and_logged`,
  `test_BC_2_21_008_ack_data_error_class_code_consumed_and_logged`

## Architecture Mapping

| Component | Module | File | Pure/Effectful |
|-----------|--------|------|---------------|
| `S7ClassicFunction` enum | SS-21 data model | `src/analyzer/s7comm.rs` | N/A (grows across STORY-188/189) |
| `S7AreaCode` enum | SS-21 data model | `src/analyzer/s7comm.rs` | N/A |
| `PlcControlService` enum | SS-21 data model | `src/analyzer/s7comm.rs` | N/A |
| `classify_job_ack_function` | SS-21 classifier | `src/analyzer/s7comm.rs` | Pure (free fn, VP-052/VP-054 target) |
| `S7commAnalyzer::on_data` (Ack error_class/error_code consumption) | SS-21 effectful shell | `src/analyzer/s7comm.rs` | Effectful (logs/surfaces the already-extracted BC-2.21.008 fields; re-anchored from STORY-187 per F-13 ruling) |

Subsystem anchor: SS-21 owns this story's scope — Job/Ack_Data function-code
classification is a core dissection capability of the S7comm analyzer per
ARCH-INDEX.md §SS-21.

## Purity Classification

| Module | Classification | Justification |
|--------|---------------|---------------|
| `classify_job_ack_function` | pure-core | Returns `S7ClassicFunction` by value; no mutation, no finding emission, no I/O |
| `S7ClassicFunction`, `S7AreaCode`, `PlcControlService` | pure-core | Plain data types |
| `S7commAnalyzer::on_data` (Ack error_class/error_code logging, AC-188-010) | effectful-shell | Consumes/logs the already-parsed BC-2.21.008 fields; no re-parsing, no `Finding` emission |

## VP-052 Proptest Obligation (partial — FC totality sub-part; completed in STORY-189)

**Harness:** `proptest_vp052_fc_classification_totality` (skeleton, this story)
**Method:** proptest
**Priority:** P1

This story covers the Job/Ack_Data function-code totality sub-part of VP-052. The
Userdata function-group totality sub-part (including the load-bearing group `0x03`/`0x07`
correction) is covered in STORY-189. Full non-vacuous run of both sub-parts is in
STORY-194.

## VP-054 Proptest Obligation

**Harness:** `proptest_vp054_download_upload_structural_disjointness` (anchored in this
story)
**Method:** proptest
**Priority:** P1

Verifies the Download triad (`0x1A`-`0x1C`) and Upload triad (`0x1D`-`0x1F`) are treated
as two independent sub-ranges with no shared match arm, and no Download value is ever
classified as, aliased to, or conflated with any Upload value. Skeleton here; full run in
STORY-194.

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
- [ ] Wire `classify_job_ack_function` into `S7commAnalyzer::on_data`'s classic-S7comm
      branch (from STORY-187), for `header.rosctr ∈ {Job, AckData}`
- [ ] Implement the BC-2.21.008 Postcondition 4 consumption obligation (AC-188-010,
      re-anchored from STORY-187 per F-13 ruling, 2026-09-24, rescoped 2026-09-24 to
      explicitly include Ack_Data per the canonical-frame holdout ruling
      DF-CANONICAL-FRAME-HOLDOUT-001): when `S7commAnalyzer::on_data` observes a header
      with `rosctr == Ack` OR `rosctr == AckData`, log/surface the already-parsed
      `error_class`/`error_code` values — no re-parsing; for `rosctr == Ack` no
      function-code classification is attempted (no parameter block); for `rosctr ==
      AckData`, the error-field logging fires independently of (and in addition to)
      `classify_job_ack_function`'s normal parameter-block classification at
      `data[header_len] == data[12]`
- [ ] Write `proptest_vp052_fc_classification_totality` and
      `proptest_vp054_download_upload_structural_disjointness` skeletons
- [ ] Write unit tests: one per AC, named `test_BC_2_21_010_*` .. `test_BC_2_21_017_*`,
      plus `test_BC_2_21_008_ack_error_class_code_consumed_and_logged` and
      `test_BC_2_21_008_ack_data_error_class_code_consumed_and_logged` for AC-188-010
- [ ] Extend `tests/fixtures/mk_s7comm_pcap.py` with Read/Write Var, Download/Upload
      triad, PLC Control, and PLC Stop synthetic frames, plus an Ack-ROSCTR frame and an
      Ack_Data-ROSCTR frame (12-byte header, parameter block at `data[12]`), each with a
      non-zero `error_class`/`error_code` pair (AC-188-010)
- [ ] Verify `cargo test` passes
- [ ] Add a CHANGELOG entry under `[Unreleased] > Added` describing the Job/Ack_Data
      function-code classification and the Ack error_class/error_code consumption,
      before creating the PR

## Edge Cases

| ID | Source BC | Description | Expected Behavior |
|----|-----------|-------------|-------------------|
| EC-001 | BC-2.21.012 | Write Var with unrecognized area byte `0xFF` | `WriteVar(Unrecognized(0xFF))` — never force-fit to a named area |
| EC-002 | BC-2.21.013 | `RequestDownload` immediately followed by another `RequestDownload` (no intervening `DownloadEnded`) | Each frame classified independently at this layer; session-level abandon/reset semantics are STORY-191's concern |
| EC-003 | BC-2.21.014 | `0x1A..=0x1F` implemented as a single collapsed range (regression scenario) | MUST NOT occur — `proptest_vp054` regression-guards this explicitly |
| EC-004 | BC-2.21.015 | PI-service parameter block contains `"P_PROGRAM"` followed by additional undecoded trailing bytes | Classified `PlcControl(ProgramStart)`; trailing-byte sub-operation decode (restart disambiguation) is out of this story's scope, handled later per its own dedicated contract |
| EC-005 | BC-2.21.015 | Service-string field truncated (insufficient parameter-block bytes) | `PlcControl(Unrecognized)` — never a hard reject |
| EC-006 | BC-2.21.017 | `data[header_len] == 0x00` with `param_length == 1` | `Unrecognized(0x00)` |
| EC-007 | BC-2.21.008 EC-004 (F-13 re-anchor) | Ack- or Ack_Data-ROSCTR header with `error_class == 0x00` and `error_code` zero-equivalent (no error reported) | Logged/surfaced the same as any other value, for either ROSCTR — a zero is a normal successful-Ack(_Data) value, not itself flagged or suppressed |
| EC-008 | BC-2.21.008 EC-007/EC-008 (canonical-frame holdout ruling, 2026-09-24) | Ack_Data header with a non-empty parameter block AND non-zero `error_class`/`error_code` (real-world Setup Communication response shape) | The error fields are logged AND the parameter block at `data[header_len] == data[12]` is classified by `classify_job_ack_function` independently — a populated error-class/code pair never suppresses FC classification |

## Token Budget Estimate

| Context Source | Estimated Tokens |
|---------------|-----------------|
| This story spec | ~6,700 |
| BC files (9 BCs: BC-2.21.008, BC-2.21.010-017) | ~9,300 |
| ADR-014 Decision 5, s7comm-mitre-ics-tagging.md (classification surface only) | ~6,000 |
| src/analyzer/s7comm.rs (from STORY-187) | ~5,000 |
| Test file delta + fixture generator extension | ~4,200 |
| **Total** | **~31,200** |
| Agent context window | 200K for Sonnet |
| **Budget usage** | **~15%** |

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
  that invokes it is the effectful shell (unchanged from STORY-187).
- **BC-2.21.008 Postcondition 4 re-anchor (F-13 ruling, human-ratified 2026-09-24;
  rescoped 2026-09-24 to explicitly include Ack_Data per the canonical-frame holdout
  ruling, DF-CANONICAL-FRAME-HOLDOUT-001)**: this story owns the consumption/logging of
  the Ack- AND Ack_Data-ROSCTR `error_class`/`error_code` values that STORY-187 already
  parses and returns from `parse_s7comm_header` (BC-2.21.008 Postconditions 1-3,
  unchanged, still STORY-187's scope, `header_len == 12` for both ROSCTR values). This
  story does NOT re-parse those fields — it only logs/surfaces the `Some(byte)` values
  `parse_s7comm_header` already produced. For Ack_Data specifically, this logging
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
| `src/analyzer/s7comm.rs` | MODIFY | Add `S7ClassicFunction`, `S7AreaCode`, `PlcControlService`, `classify_job_ack_function`; wire into `on_data`; add Ack AND Ack_Data `error_class`/`error_code` consumption/logging (BC-2.21.008 Postcondition 4, AC-188-010) |
| `tests/s7comm_analyzer_tests.rs` | MODIFY | Add BC-2.21.008 (AC-188-010, both Ack and Ack_Data cases) + BC-2.21.010-017 unit tests + VP-052 (partial)/VP-054 proptest skeletons |
| `tests/fixtures/mk_s7comm_pcap.py` | MODIFY | Add Read/Write Var, Download/Upload triad, PLC Control, PLC Stop frames |

## Forbidden Dependencies

- Wireshark, Snap7, libnodave source, and any `s7`/`s7-comm`/`s7-client` crate — banned/
  avoid per ADR-014 Decision 4
- `classify_job_ack_function` MUST NOT call `emit_finding` or access `S7commFlowState` —
  it is a pure classification fn; emission lands in STORY-191/192

## Changelog

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.2 | 2026-09-24 | story-writer | Synced this story to the STORY-187 canonical-frame holdout human ruling (DF-CANONICAL-FRAME-HOLDOUT-001, 2026-09-24): BC-2.21.008's Postcondition 4 consumption/logging obligation (already re-anchored to this story in v1.1) is now rescoped to explicitly cover BOTH ROSCTR `0x02` (Ack) and `0x03` (Ack_Data) — Ack_Data was previously (incorrectly, per v1.0/v1.1) assumed to carry no error fields of its own, but the amended BC-2.21.008 v1.2 shows Ack_Data also carries `error_class`/`error_code` at `data[10..12]` with `header_len == 12`, and its parameter block (still classified normally by `classify_job_ack_function`) begins at `data[12]`, not `data[10]`. Changes: (1) the Behavioral Contracts table's BC-2.21.008 title row updated verbatim to the current BC H1 ("...for ROSCTR=Ack (0x02) and Ack_Data (0x03)..."); (2) AC-188-010 extended with a parallel Ack_Data given/when/then and a note that the error-logging and FC-classification obligations for Ack_Data fire independently of each other; (3) a new test named `test_BC_2_21_008_ack_data_error_class_code_consumed_and_logged`; (4) Tasks, Architecture Mapping/Purity Classification note, Architecture Compliance Rules, File Structure Requirements, and the fixture-generator task updated to mention Ack_Data alongside Ack; (5) new Edge Case EC-008 (Ack_Data with a populated parameter block AND non-zero error fields — both obligations fire); EC-007 generalized to "Ack- or Ack_Data-ROSCTR" wording. Verified: no fixture, worked example, or AC in this story hardcodes an Ack_Data parameter-block offset of `data[10]` — the classifier signature (`classify_job_ack_function(data, header_len, param_length)`) and every AC already parametrize on `data[header_len]` generically, so no other change was required for the offset correction itself. Points unchanged (a second small logging AC on an already-generic classifier, not judged large enough to require a bump). |
| 1.1 | 2026-09-24 | story-writer | Propagated F-13 human ruling (STORY-187 per-story adversarial pass 1, human-ratified 2026-09-24; BC-2.21.008 v1.1 Postcondition 4 and Traceability/Story Anchor sections) into this story: BC-2.21.008 added to `behavioral_contracts`/`inputs` frontmatter and the body Behavioral Contracts table (Postcondition 4 only — parse/extraction, Postconditions 1-3, remains STORY-187's scope). New AC-188-010 (traces to BC-2.21.008 postcondition 4): `S7commAnalyzer::on_data` consumes/logs the already-parsed Ack-ROSCTR `error_class`/`error_code` values; a zero error class/code is logged the same as any other value (BC-2.21.008 EC-004). Added Architecture Mapping/Purity Classification rows for the Ack-consumption effectful-shell behavior, a Tasks entry, a named unit test (`test_BC_2_21_008_ack_error_class_code_consumed_and_logged`), Edge Case EC-007, a Token Budget BC-count update (8->9 BCs), an Architecture Compliance Rules note on the re-anchor, and a File Structure Requirements update. Points unchanged pending story-writer's points-change recommendation (see handback report) — the added scope is a single small logging AC, not judged large enough on its own to require a bump, but should be considered alongside STORY-187's larger F-01/F-02/F-12 scope growth when wave/points are next reviewed. |
| 1.0 | 2026-09-06 | story-writer | Initial authorship — Job/Ack_Data function-code classification (Setup Comm, Read/Write Var, Download/Upload triads, PLC Control, PLC Stop), VP-052 (partial)/VP-054 skeletons, AC-188-001..009. |
