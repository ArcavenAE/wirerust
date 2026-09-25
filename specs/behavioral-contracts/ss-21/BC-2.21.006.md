---
document_type: behavioral-contract
level: L3
version: "1.6"
status: draft
producer: product-owner
timestamp: 2026-09-24T12:00:00Z
phase: f2
origin: greenfield
extracted_from: null
traces_to: .factory/specs/domain/domain-spec.md
subsystem: SS-21
capability: CAP-21
lifecycle_status: active
introduced: feature-s7comm
modified:
  - version: "1.6"
    date: 2026-09-25
    change: "STORY-187 per-story adversarial pass 13 (P13-F-1): Architecture Anchor test-count re-verification. The Architecture Anchors 'Tests anchor' entry was stale — it cited 3 tests from pass 3 (F-31), before a test added in passes 12-14. Re-grepped `tests/s7comm_analyzer_tests.rs` directly and replaced with the actual current count (4 `test_BC_2_21_006_*` functions plus the joint proptest) and the full function-name list, verified 2026-09-25 against worktree HEAD 38ff7ee1. Removed the stale 'no drift found (F-31)' claim, which no longer held. No change to Preconditions/Postconditions/Invariants/Edge Cases — Architecture Anchors traceability correction only."
  - version: "1.5"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 9 (F-53): EC-001's both-zero example was wrong on two counts — 'Setup Communication response' is an Ack_Data (0x03) frame, which is out of this BC's 10-byte scope (Ack_Data requires the 12-byte header, BC-2.21.008) and, per the canonical cnblogs frame, carries param_length 8, not 0; and 'a Userdata frame with all information in the parameter block only' is self-contradictory (a populated parameter block implies param_length > 0, not param_length == 0). Replaced with a real in-scope example: a minimal Job PDU (ROSCTR 0x01) with an empty parameter block and no data (param_length == 0, data_length == 0) — the shape `tests/fixtures/mk_s7comm_pcap.py`'s `minimal_job_pdu` labels 'BC-2.21.006 EC-001' (~:304-312). No change to Preconditions/Postconditions/Invariants — edge-case example correction only."
  - version: "1.4"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 7 (F-49): VP-051 source-set expansion. This BC's Postconditions 1-4 (Some(S7commHeader{..., header_len: 10}) for rosctr ∈ {Job, Userdata}; big-endian pdu_reference/param_length/data_length extraction at data[4..6]/data[6..8]/data[8..10]) are asserted by VP-051's Kani harness (S7comm Header Bounds-Before-Slice Safety) as part of the same bounds-and-extraction proof already covering BC-2.21.004/008/009 — architect registering this BC to VP-051's source_bc in VP-INDEX.md in parallel (five-BC set: BC-2.21.004, BC-2.21.006, BC-2.21.007, BC-2.21.008, BC-2.21.009). Verification Properties table row and VP Anchors section corrected from 'cargo-fuzz P1 (combined harness) — VP-NNN allocation deferred' / '(None dedicated — not in VP-051's registered source_bc {BC-2.21.004, BC-2.21.008, BC-2.21.009}...)' to cite VP-051 (Kani P0) as the primary registered target for Postconditions 1-4, joint with BC-2.21.004/007/008/009, with VP-055 (cargo-fuzz P1) complementary — mirroring BC-2.21.004/008/009's own sibling wording. Purity Classification's 'Overall classification' row corrected from 'pure core — cargo-fuzz P1 target' to 'pure core — VP-051 (Kani P0) joint target (BC-2.21.004/007/008/009); VP-055 (cargo-fuzz P1) complementary'. No change to Preconditions/Postconditions/Invariants themselves — verification-anchoring correction only."
  - version: "1.3"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 6 (F-45/N-1): VP-051 source-set sibling sweep. VP Anchors section's 'not in VP-051's registered source_bc {BC-2.21.004, BC-2.21.009}' corrected to '{BC-2.21.004, BC-2.21.008, BC-2.21.009}', matching VP-INDEX.md's current registration (pass 5, F-43) — this BC remains outside VP-051's source_bc (no dedicated VP anchor); only the cited set was stale."
  - version: "1.2"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 3 (F-27/F-31): F-27 (this BC's Related BCs entry): the BC-2.21.008 relation now explicitly states BC-2.21.008 is the accept-path sibling for `data[1] ∈ {0x02, 0x03}` requiring `len ≥ 12`, mirroring this BC's own `data[1] ∈ {0x01, 0x07}` requiring `len ≥ 10`, so BC-2.21.007's length-conditional totality property (see BC-2.21.007 v1.1) has an unambiguous pair of accept-path citations. F-31 re-anchor sweep: Traceability 'Stories' field corrected from '(TBD — story-writer assigns in F3)' to 'STORY-187'. Architecture Module and Architecture Anchors' '(planned)' markers removed — `src/analyzer/s7comm.rs`, `pub fn parse_s7comm_header`, `pub struct S7commHeader { .. }`, and `pub enum Rosctr { Job, Ack, AckData, Userdata }` are all implemented, not planned. Added a Tests anchor citing `tests/s7comm_analyzer_tests.rs`'s `mod story_187` BC-2.21.006-labeled test functions (3 tests, plus the joint `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional`; verified test-name/BC-ID alignment, no drift found)."
  - version: "1.1"
    date: 2026-09-24
    change: "STORY-187 canonical-frame holdout (DF-CANONICAL-FRAME-HOLDOUT-001), human ruling 2026-09-24: Ack and Ack_Data both 12-byte headers. Description and Postcondition 1 corrected — Ack_Data (0x03) no longer belongs to the 10-byte common-header happy-path group; control passes to BC-2.21.008 for `data[1] ∈ {0x02, 0x03}` (was `data[1] == 0x02` only). This BC's Postconditions now apply only to `rosctr ∈ {Job, Userdata}`. Canonical Test Vectors' Ack_Data row removed (moved to BC-2.21.008, which now documents the Ack_Data 12-byte shape); Related BCs updated. See BC-2.21.008's `modified:` entry for the full source citation."
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

# BC-2.21.006: `parse_s7comm_header` Extracts ROSCTR, PDU Reference, Parameter Length, and Data Length from a Valid 10-byte Common Header (Happy Path)

## Description

Given `data.len() >= 10` and `data[0] == 0x32` (BC-2.21.004/005 passed),
`parse_s7comm_header` extracts the common S7comm header fields: `data[1]` is the
ROSCTR byte (`0x01` Job, `0x02` Ack, `0x03` Ack_Data, `0x07` Userdata — the four
recognized values, BC-2.21.007 covers all others); `data[2..4]` is a 2-byte Reserved
field (read but not semantically interpreted); `data[4..6]` is the PDU Reference
(`u16`, big-endian); `data[6..8]` is the Parameter Length (`u16`, big-endian);
`data[8..10]` is the Data Length (`u16`, big-endian). For ROSCTR ∈ {Job, Userdata},
this 10-byte common header is the complete header (`header_len == 10`); for
ROSCTR ∈ {Ack, Ack_Data}, two additional bytes (Error Class + Error Code) follow
(BC-2.21.008) — Ack_Data is NOT part of this BC's 10-byte happy-path group (corrected
2026-09-24, STORY-187 canonical-frame holdout ruling, DF-CANONICAL-FRAME-HOLDOUT-001;
see BC-2.21.008's `modified:` entry for the source citations).

## Preconditions

1. `data.len() >= 10`.
2. `data[0] == 0x32`.
3. `data[1] ∈ {0x01, 0x02, 0x03, 0x07}` (a recognized ROSCTR value).

## Postconditions

1. `parse_s7comm_header(data)` returns `Some(S7commHeader { rosctr, pdu_reference,
   param_length, data_length, error_class: None, error_code: None, header_len: 10 })`
   for `rosctr ∈ {Job, Userdata}` (i.e. `data[1] ∈ {0x01, 0x07}`); for
   `data[1] ∈ {0x02, 0x03}` (Ack, Ack_Data), control passes to BC-2.21.008 instead
   (this BC's Postconditions apply only to the two non-Ack/non-Ack_Data values).
2. `pdu_reference = u16::from_be_bytes([data[4], data[5]])`.
3. `param_length = u16::from_be_bytes([data[6], data[7]])`.
4. `data_length = u16::from_be_bytes([data[8], data[9]])`.
5. `data[2..4]` (Reserved) is read for header-length bookkeeping but never compared,
   matched, or branched on.
6. No bounds-consistency check between `param_length`/`data_length` and the actual
   remaining bytes in `data` occurs in this function — that check is BC-2.21.009's
   responsibility, applied by the caller after this function returns `Some`.

## Invariants

1. **Field order is fixed**: Protocol ID, ROSCTR, Reserved, PDU Reference, Parameter
   Length, Data Length — this ordering is a structural fact of the S7comm wire format
   (per free-to-read prose sources, ADR-014 Decision 4), not a design choice.
2. **Big-endian multi-byte fields**: PDU Reference, Parameter Length, and Data Length
   are all big-endian `u16` — consistent across all ROSCTR values.
3. **Purity**: no state mutation; deterministic; no panic for any `data.len() >= 10`.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | `param_length == 0` and `data_length == 0` (e.g. a minimal Job PDU (ROSCTR `0x01`) with an empty parameter block and no data — see `tests/fixtures/mk_s7comm_pcap.py`'s `minimal_job_pdu`) | Extracted normally; a zero-length parameter/data block is not itself an error at this layer |
| EC-002 | `pdu_reference == 0x0000` | Extracted verbatim; PDU reference `0` is not treated as invalid at the header-parse layer (used only for later request/response correlation, out of B1 scope) |
| EC-003 | Reserved bytes (`data[2..4]`) are non-zero (spec typically expects `0x0000`) | Extracted and discarded; no rejection — the field is documented as read-but-unvalidated by Postcondition 5 |

## Canonical Test Vectors

| Input (`data`, hex bytes) | Expected `S7commHeader` | Category |
|---|---|---|
| `32 01 00 00 00 01 00 02 00 00` | `{rosctr: Job, pdu_reference: 1, param_length: 2, data_length: 0, header_len: 10}` | happy-path: Job, minimal |
| `32 07 00 00 00 05 00 08 00 00` | `{rosctr: Userdata, pdu_reference: 5, param_length: 8, data_length: 0, header_len: 10}` | happy-path: Userdata |

> Ack_Data (`0x03`) is **not** covered by this table — as of the 2026-09-24
> canonical-frame holdout ruling (DF-CANONICAL-FRAME-HOLDOUT-001), Ack_Data requires
> the 12-byte header documented in BC-2.21.008 (Error Class + Error Code at
> `data[10..12]`, parameter block starting at `data[12]`), not the 10-byte header this
> BC's Postconditions describe. See BC-2.21.008's Canonical Test Vectors for the
> Ack_Data happy-path row.

## Verification Properties

| Property | Proof Method (planned) |
|----------|-------------------------|
| Field extraction is correct (matches byte-for-byte expected values) for all 10-byte-minimum inputs with a recognized ROSCTR; no panic for any symbolic `data` of length ≥ 10 | VP-051 (Kani P0) — "S7comm Header Bounds-Before-Slice Safety," asserts this BC's Postconditions 1-4 (`header_len == 10` field extraction for `rosctr ∈ {Job, Userdata}`; big-endian `pdu_reference`/`param_length`/`data_length` reads), joint with BC-2.21.004, BC-2.21.007, BC-2.21.008, BC-2.21.009 (see VP Anchors below); registered F2 INTEGRATE sub-burst per VP-INDEX.md (this BC registered to VP-051's `source_bc`, F-49); cargo-fuzz P1 (VP-055) provides complementary combined-chain no-panic coverage |

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| Capability Anchor Justification | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 — this is the happy-path common-header extraction that every downstream function-code classification BC (010–023) depends on |
| L2 Domain Invariants | None directly |
| Architecture Module | SS-21 (`src/analyzer/s7comm.rs`) |
| ADR | ADR-014 Decision 9 |
| Stories | STORY-187 |
| Feature | feature-s7comm |
| MITRE Techniques | (none — pure extraction, no finding emission) |

## Related BCs

- BC-2.21.004 — composes with (length-reject precedes this path)
- BC-2.21.005 — composes with (protocol-ID guard precedes this path)
- BC-2.21.007 — composes with (unrecognized-ROSCTR sibling reject path)
- BC-2.21.008 — composes with (the accept-path sibling for `data[1] ∈ {0x02, 0x03}` requiring `len ≥ 12` — Ack/Ack_Data's additional 2-byte requirement; Ack_Data moved fully to BC-2.21.008's 12-byte header as of the 2026-09-24 canonical-frame holdout ruling; together with BC-2.21.007 these three BCs jointly specify `parse_s7comm_header`'s length-conditional totality, F-27)
- BC-2.21.009 — depends on (this BC's `param_length`/`data_length` are consumed by the bounds check there)
- BC-2.21.010 through BC-2.21.023 — depend on (all function-code and Userdata classification is downstream of this extraction)

## Architecture Anchors

- `src/analyzer/s7comm.rs` — `pub fn parse_s7comm_header`, common-header field extraction (implemented, STORY-187)
- `pub struct S7commHeader { pub rosctr: Rosctr, pub pdu_reference: u16, pub param_length: u16, pub data_length: u16, pub error_class: Option<u8>, pub error_code: Option<u8>, pub header_len: usize }` (implemented, this BC's design)
- `pub enum Rosctr { Job, Ack, AckData, Userdata }` (implemented, this BC's design)
- `tests/s7comm_analyzer_tests.rs` — Tests anchor: 4 `test_BC_2_21_006_*` functions (re-counted by direct grep, verified 2026-09-25 against worktree HEAD 38ff7ee1): `test_BC_2_21_006_canonical_setup_communication_job_frame_on_data`, `test_BC_2_21_006_common_header_field_extraction`, `test_BC_2_21_006_nonzero_reserved_bytes_do_not_reject`, `test_BC_2_21_006_byte_asymmetric_big_endian_decode`; plus the joint `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional` (shared with BC-2.21.008).

## Story Anchor

STORY-187

## VP Anchors

- VP-051 (Kani P0) — S7comm Header Bounds-Before-Slice Safety; asserts this BC's
  Postconditions 1-4 (`header_len == 10` extraction for `rosctr ∈ {Job, Userdata}`;
  big-endian `pdu_reference`/`param_length`/`data_length` field reads); joint with
  BC-2.21.004, BC-2.21.007, BC-2.21.008, BC-2.21.009; architect registered this BC to
  VP-051's `source_bc` in VP-INDEX.md per the F-49 ruling (STORY-187 per-story
  adversarial pass 7, 2026-09-24; `source_bc` is now `{BC-2.21.004, BC-2.21.006,
  BC-2.21.007, BC-2.21.008, BC-2.21.009}` — previously `{BC-2.21.004, BC-2.21.008,
  BC-2.21.009}`, this BC and BC-2.21.007 had been omitted despite VP-051's harness
  already covering their postconditions)
- VP-055 (cargo-fuzz P1) — S7comm/ISO-on-TCP combined parse-chain no-panic fuzz
  (`fuzz_s7comm_parser`); complementary combined-chain coverage for this
  field-extraction path

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | none |
| **Deterministic** | yes |
| **Thread safety** | Send + Sync |
| **Overall classification** | pure core — VP-051 (Kani P0) joint target (BC-2.21.004/007/008/009); VP-055 (cargo-fuzz P1) complementary |
