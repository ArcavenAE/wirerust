---
document_type: behavioral-contract
level: L3
version: "1.9"
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
  - version: "1.9"
    date: 2026-09-25
    change: "STORY-187 final adversarial batch (passes 19-21): anchor count 8→9, EC-007 trace scope. Architecture Anchors 'Tests anchor' entry was stale again — a ninth `test_BC_2_21_008_*` function, `test_BC_2_21_008_canonical_ack_vector_verbatim` (parses BC-2.21.008's own canonical Ack 12-byte happy-path vector, `32 02 00 00 00 01 00 00 00 00 00 00`, byte-for-byte with no substitution, asserting the exact `S7commHeader` including `error_class: Some(0)`/`error_code: Some(0)`), was added since the v1.8/pass-13 count of 8. Re-grepped `tests/s7comm_analyzer_tests.rs` directly and replaced the Tests anchor with the actual current count (9 `test_BC_2_21_008_*` functions plus the joint proptest) and the full function-name list, verified 2026-09-25 against worktree HEAD b4fce34a. EC-007's trace to `test_BC_2_21_008_ack_data_nonzero_error_fields_with_parameter_block` is now annotated: that test covers only the extraction half of EC-007 (verifying `error_class`/`error_code` are correctly extracted as `Some` alongside a present, bounds-satisfying parameter block, and that `on_data` emits no finding) — the Group-3 function-code classification EC-007's Expected-Behavior column describes as proceeding 'independently at `data[12]`' (BC-2.21.010 onward) is STORY-188 scope and is untested by this file's BC-2.21.008 suite. Also re-grepped the other seven STORY-187 SS-21 BCs' (001, 002, 004, 005, 006, 007, 009) Tests-anchor lists against the same HEAD (b4fce34a): all seven counts (13/13/4/2/4/3/12 respectively) confirmed unchanged from their existing anchor text — no drift found, no edits made to those files. No change to Preconditions/Postconditions/Invariants/Edge Cases themselves — Architecture Anchors traceability correction only."
  - version: "1.8"
    date: 2026-09-25
    change: "STORY-187 per-story adversarial pass 13 (P13-F-1): Architecture Anchor test-count re-verification. The Architecture Anchors 'Tests anchor' entry was stale — it cited 7 tests from pass 3 (F-31), before a test added in passes 12-14. Re-grepped `tests/s7comm_analyzer_tests.rs` directly and replaced with the actual current count (8 `test_BC_2_21_008_*` functions plus the joint proptest) and the full function-name list, verified 2026-09-25 against worktree HEAD 38ff7ee1. Removed the stale 'no drift found (F-31)' claim, which no longer held. Explicitly traced EC-007 to `test_BC_2_21_008_ack_data_nonzero_error_fields_with_parameter_block`. No change to Preconditions/Postconditions/Invariants/Edge Cases — Architecture Anchors traceability correction only."
  - version: "1.7"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 7 (F-49/F-50). F-49 (VP-051 source-set expansion): architect registering BC-2.21.006/BC-2.21.007 to VP-051's source_bc in VP-INDEX.md in parallel — the registered set becomes {BC-2.21.004, BC-2.21.006, BC-2.21.007, BC-2.21.008, BC-2.21.009}, previously {BC-2.21.004, BC-2.21.008, BC-2.21.009} (F-43/pass 5). VP Anchors section, Verification Properties table row, and Purity Classification's joint-target parenthetical corrected from the three-BC set to the five-BC set. F-50 (HIGH, source-attestation overreach): the Description's provenance paragraph stated Kleinmann & Wool 2014 documents the error block 'only for ROSCTR 3' (Ack_Data), 'in addition to the analogous Ack (ROSCTR 2) case' — this misreads K&W, whose Figure 2/§3.2 attest the error block for ROSCTR 3 (Ack_Data) only; K&W's own captured traffic sample observed only ROSCTR 1 (Job) and ROSCTR 3 (Ack_Data), so K&W says nothing about the Ack (0x02) case at all. Reworded: K&W now cited as attesting the Ack_Data (0x03) error-block layout exclusively; the Ack (0x02) 12-byte layout's grounding rests solely on the two permitted open-source design references (cisagov/icsnpp-s7comm's shared ROSCTR_ACK/ROSCTR_ACK_Data S7Comm_Error record, python-snap7's ACK/ACK_DATA parse_response handling), with kprovost/libs7comm independently consistent for both ROSCTR values. EC-008's cross-reference ('the cited permitted design references — Kleinmann & Wool 2014, icsnpp-s7comm, python-snap7, libs7comm') incorrectly listed K&W (a Decision 4 prose source) among the 'permitted design references' (a distinct Decision 4 category covering icsnpp-s7comm/python-snap7/libs7comm only) — reworded to cite K&W separately as the prose source (Ack_Data only) and the three implementations as the permitted design references. No change to Preconditions, Postconditions, or Invariants — provenance/citation correction only; postcondition semantics unchanged."
  - version: "1.1"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 1 (F-01/F-02/F-12/F-13/F-14), human ruling 2026-09-24. F-13: Postcondition 4's 'S7commAnalyzer logs the observed error class/code' clause is out of scope for STORY-187's parse-only contract. Re-anchored (not deleted) to STORY-188 (\"S7comm Job/Ack_Data Function-Code Classification\"), the next SS-21 story in the classic-S7comm dissection chain — STORY-189 (Userdata parse/classification) does not touch ROSCTR=Ack at all, so STORY-188 is the correct target. STORY-188's current ACs do not yet name this obligation explicitly; story-writer must add an AC before that story is considered to close it. Traceability 'Stories' field and Story Anchor section updated to record the split (parse/extraction stays STORY-187; consumption/logging deferred to STORY-188)."
  - version: "1.2"
    date: 2026-09-24
    change: "STORY-187 canonical-frame holdout (DF-CANONICAL-FRAME-HOLDOUT-001), human ruling 2026-09-24: Ack and Ack_Data both 12-byte headers. Four independent real-world sources (cnblogs \"西门子S7通讯协议引用整理\" https://www.cnblogs.com/crcce-dncs/p/10659087.html; Yiqisoft 2023-03-22 https://www.yiqisoft.cn/blogs/IoT-Gateway/363.html; Inductive Automation KB \"Loggers - Device Connections: Siemens\"; Kleinmann & Wool 2014 prose) show an Ack_Data (0x03) Setup Communication response ALSO carries Error Class (data[10]) + Error Code (data[11]), with the parameter block starting at data[12] — contradicting this BC's v1.0/v1.1 claim that Ack_Data uses the plain 10-byte common header with no error fields. H1 retitled to name both ROSCTR values; Preconditions/Postconditions/Invariants/Edge Cases/Canonical Test Vectors generalized to `data[1] ∈ {0x02, 0x03}` and `header_len == 12` for both; added an Ack_Data 12-byte happy-path vector (cnblogs Setup Communication response bytes) and Ack_Data 10/11-byte truncated→None vectors. Postcondition 4's STORY-188 deferral (error-class/code consumption/logging) now explicitly covers both Ack and Ack_Data. BC-2.21.004/006/009 amended in the same burst for consistency (see their own `modified:` entries)."
  - version: "1.3"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 3 (F-31), re-anchor sweep. Traceability 'Stories' field and Story Anchor section were already correctly split between STORY-187 (parse/extraction) and STORY-188 (error-class/code consumption, PC4 deferral) as of v1.2 — no change needed there. Architecture Module and Architecture Anchors' '(planned)' markers removed — `src/analyzer/s7comm.rs` and `pub fn parse_s7comm_header`'s shared `0x02 | 0x03` match arm (`if data.len() < 12 { return None; }` followed by `Rosctr::Ack`/`Rosctr::AckData` construction) are implemented, not planned. Added a Tests anchor citing `tests/s7comm_analyzer_tests.rs`'s `mod story_187` BC-2.21.008-labeled test functions (7 tests, plus the joint `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional` and the totality proptest shared with BC-2.21.007); verified test-name/BC-ID alignment, no drift found."
  - version: "1.4"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 4→5 findings (F-39/F-42/F-43). F-43 (orchestrator decision): VP-051's Kani harness (S7comm Header Bounds-Before-Slice Safety) already asserts this BC's Postconditions 2/3 (the `header_len == 12` selection for `rosctr ∈ {Ack, AckData}` and the `error_class`/`error_code` `Some`-iff-`rosctr ∈ {Ack, AckData}` non-vacuity check, per VP-INDEX.md's VP-051 HARNESS REQUIREMENTS clause) — architect is adding this BC to VP-051's registered `source_bc` (previously `{BC-2.21.004, BC-2.21.009}`) in a parallel, separately-scoped burst. VP Anchors section updated from '(None dedicated — not in VP-051's registered source_bc {BC-2.21.004, BC-2.21.009}...)' to cite VP-051 (Kani P0) for Postconditions 2/3, joint with BC-2.21.004/BC-2.21.009, with VP-055 (cargo-fuzz P1) as complementary combined-chain coverage — mirroring BC-2.21.004/BC-2.21.009's own VP Anchors phrasing. Verification Properties table row corrected from 'cargo-fuzz P1 (combined harness) — VP-NNN allocation deferred' to cite VP-051 (Kani P0) as the primary registered target for the Ack/Ack_Data 12-byte-minimum and error-field-presence property, with VP-055 (cargo-fuzz P1) complementary — consistent with BC-2.21.004/BC-2.21.009's already-corrected rows (F-14/F-35, prior passes). Purity Classification's 'Overall classification' row corrected from 'pure core — cargo-fuzz P1 target' to 'pure core — VP-051 (Kani P0) joint target (BC-2.21.004/BC-2.21.009); VP-055 (cargo-fuzz P1) complementary', matching BC-2.21.004/BC-2.21.009's sibling wording. No change to Preconditions/Postconditions/Invariants themselves — this is a verification-anchoring correction only, consistent with the architect's parallel VP-INDEX/verification-architecture/verification-coverage-matrix update under `vp_index_is_vp_catalog_source_of_truth`."
  - version: "1.5"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 6 (F-45/N-1): VP-051 source-set sibling sweep. F-45 (HIGH, partial-fix sibling miss): the Verification Properties table row and VP Anchors section still described the F-43 registration in present/future tense ('architect adding this BC to VP-051's `source_bc`') even though VP-INDEX.md now registers VP-051's source_bc as {BC-2.21.004, BC-2.21.008, BC-2.21.009} (pass 5) — corrected to past tense ('registered'). N-1: Postcondition 1's malformed-header dedup-flag citation ('shares the dedup flag with BC-2.21.004/007') generalized to 'BC-2.21.004/007/009' — BC-2.21.009 shares the same `malformed_header_reported_c2s`/`_s2c` dedup flag via its own Postcondition 2 (which already cites BC-2.21.004/007/008 reciprocally); this BC's own Postcondition 1 had omitted 009 from that shared-flag set. Description section (lines ~49-56, the four-source Ack_Data provenance citation) intentionally left untouched — F-46 provenance fix is a separate, pending research item."
  - version: "1.6"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 6 (F-46): re-ground Ack/Ack_Data 12-byte layout in ADR-014 Decision 4 permitted sources. The normative Description's field-semantics claim (12-byte Ack/Ack_Data header with 1-byte `error_class` at `data[10]` + 1-byte `error_code` at `data[11]`) previously cited 'four independent real-world capture sources' as its grounding — three of those (cnblogs, Yiqisoft, Inductive Automation KB) are wire-capture pages that ADR-014 Decision 4's 2026-09-24 permitted-sources note authorizes as canonical-frame TEST VECTORS only, not as field-semantics provenance. Rewrote the Description to ground the field-semantics claim in permitted prose/design-reference sources instead: Kleinmann & Wool 2014 (JDFSL 9(2) §3.2, Figure 2 — error block documented 'only for ROSCTR 3'/Ack_Data), cisagov/icsnpp-s7comm (BSD-3; `src/s7comm-protocol.pac` `ROSCTR_ACK`/`ROSCTR_ACK_Data` records each embed `S7Comm_Error {error_class: uint8, error_code: uint8}`), and python-snap7 (MIT; `snap7/s7protocol.py` `parse_response` — ACK/ACK_DATA 12-byte header, error_class/error_code at offsets 10/11), with kprovost/libs7comm (BSD-2) independently consistent (ROSCTR types 2 and 3 both carry 2 extra header bytes). Reviewed 2026-09-24; these are reverse-engineered implementations and an academic analysis, not a Siemens spec. The cnblogs/Yiqisoft/Inductive Automation KB wire captures remain cited, but relabeled as the canonical-frame TEST VECTORS that surfaced the defect and corroborate it at the byte level (DF-CANONICAL-FRAME-HOLDOUT-001) — test-vector use only, per the ADR-014 Decision 4 2026-09-24 note. The Canonical Test Vectors table's Ack_Data happy-path row (cnblogs bytes) is unchanged in content and is kept as a test-vector citation (permitted use); its citation label was clarified accordingly. Edge Case EC-008's cross-reference to 'all four cited sources' was reworded to distinguish the test-vector captures (real-world shape corroboration) from the permitted design references (independent implementation consistency). No change to Preconditions, Postconditions, or Invariants — this is a provenance/citation correction only; postcondition semantics are unchanged."
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

# BC-2.21.008: `parse_s7comm_header` for ROSCTR=Ack (0x02) and Ack_Data (0x03) Requires 12 Bytes (Error Class + Error Code)

## Description

The Ack ROSCTR (`0x02`) is a bare acknowledgment carrying no parameter or data block —
only the 10-byte common header (BC-2.21.006) plus two additional bytes: Error Class
(`data[10]`) and Error Code (`data[11]`). Ack_Data (`0x03`) carries the **same** two
additional bytes at the same offsets, immediately followed by its parameter/data block
starting at `data[12]` (not `data[10]`, as this BC's v1.0/v1.1 previously assumed). This
12-byte Ack/Ack_Data header — a 1-byte `error_class` field at `data[10]` and a 1-byte
`error_code` field at `data[11]` — is grounded in permitted prose and design-reference
sources per ADR-014 Decision 4, split by ROSCTR value. For Ack_Data (ROSCTR 3),
Kleinmann & Wool 2014 (JDFSL 9(2) §3.2, Figure 2) documents the S7comm error block as
present "only for ROSCTR 3" — the authors' own captured traffic sample contained only
ROSCTR 1 (Job) and ROSCTR 3 (Ack_Data) frames, so K&W attests the Ack_Data (0x03)
error-block layout exclusively and says nothing about the Ack (0x02) case (corrected
2026-09-24, STORY-187 per-story adversarial pass 7, F-50 — v1.6 read K&W as attesting
Ack "in addition to" Ack_Data, which overstates what Figure 2/§3.2 actually cover). The
Ack (ROSCTR 2) 12-byte layout's grounding rests instead solely on two permitted
open-source design references — cisagov/icsnpp-s7comm (BSD-3; `src/s7comm-protocol.pac`),
whose `ROSCTR_ACK` and `ROSCTR_ACK_Data` record definitions each embed an `S7Comm_Error {
error_class: uint8, error_code: uint8 }` field pair, and python-snap7 (MIT;
`snap7/s7protocol.py` `parse_response`), whose ACK and ACK_DATA handling reads a 12-byte
header with `error_class`/`error_code` at byte offsets 10/11 — with kprovost/libs7comm
(BSD-2) independently consistent for both ROSCTR values (ROSCTR types 2 and 3 both carry
2 extra header bytes beyond the common 10-byte header). These three are reverse-engineered
community implementations, not a Siemens specification; K&W is an independent academic
analysis, also not a Siemens specification; reviewed 2026-09-24. The
cnblogs "西门子S7通讯协议引用整理" (https://www.cnblogs.com/crcce-dncs/p/10659087.html),
Yiqisoft 2023-03-22 (https://www.yiqisoft.cn/blogs/IoT-Gateway/363.html), and the
Inductive Automation KB "Loggers - Device Connections: Siemens" wire captures are the
canonical-frame **test vectors** that originally surfaced this defect and corroborate the
field layout at the byte level (STORY-187 canonical-frame holdout finding,
DF-CANONICAL-FRAME-HOLDOUT-001, 2026-09-24) — they are cited for test-vector use only,
per the ADR-014 Decision 4 permitted-sources note (2026-09-24), and do not themselves
ground the normative field-semantics claim above. (Human ruling, STORY-187 per-story
adversarial pass 6, F-46, 2026-09-24; attestation-scope correction, pass 7, F-50,
2026-09-24.)
When `data[1] ∈ {0x02, 0x03}` (Ack or Ack_Data) and `data.len() < 12`,
`parse_s7comm_header` returns `None` (truncated); when `data.len() >= 12`, it returns
`Some(S7commHeader { rosctr: Ack | AckData, error_class: Some(data[10]), error_code:
Some(data[11]), header_len: 12, .. })`.

## Preconditions

1. `data.len() >= 10`, `data[0] == 0x32`, `data[1] ∈ {0x02, 0x03}` (Ack or Ack_Data
   ROSCTR).

## Postconditions

1. If `data.len() < 12`: returns `None`. `S7commAnalyzer` treats this as a
   malformed-header condition (shares the dedup flag with BC-2.21.004/007/009) — applies
   identically whether `data[1] == 0x02` (Ack) or `data[1] == 0x03` (Ack_Data).
2. If `data.len() >= 12`: returns
   `Some(S7commHeader { rosctr: Ack | AckData, pdu_reference, param_length, data_length,
   error_class: Some(data[10]), error_code: Some(data[11]), header_len: 12 })`, where
   `pdu_reference`/`param_length`/`data_length` are extracted identically to
   BC-2.21.006 (the common-header fields are present at the same offsets regardless of
   ROSCTR value).
3. `error_class`/`error_code` are `Some` **only** when `rosctr ∈ {Ack, AckData}`; every
   other `S7commHeader` (Job, Userdata) has both fields `None` (BC-2.21.006
   Postcondition 1).
4. No function-code classification (Group 3/4, BC-2.21.010 onward) is attempted for a
   bare Ack-ROSCTR header — Ack (`0x02`) carries no parameter block to classify.
   Ack_Data (`0x03`), by contrast, DOES carry a parameter block: Group 3 function-code
   classification (BC-2.21.010 through BC-2.21.017) proceeds normally for Ack_Data once
   BC-2.21.009's bounds check passes, reading `data[header_len]` at the corrected
   `header_len == 12` offset (not `10`, per this BC's v1.2 correction) — this is a
   behavior change from v1.0/v1.1, which (incorrectly) implied Ack_Data's parameter
   block began at `data[10]`. This BC's own scope (STORY-187, parse-only) ends at
   successfully extracting and returning `error_class`/`error_code` (Postcondition 2);
   it does NOT itself require `S7commAnalyzer` to log, surface, or otherwise act on the
   extracted values. That consumption obligation is out of scope for STORY-187's
   parse-only contract and is **deferred to STORY-188** ("S7comm Job/Ack_Data
   Function-Code Classification"), the next SS-21 story in the classic-S7comm
   dissection chain that extends `S7commAnalyzer::on_data`'s ROSCTR-conditional
   handling beyond parsing (STORY-189 covers Userdata, ROSCTR `0x07`, and does not
   touch ROSCTR ∈ {Ack, AckData} at all, so STORY-188 is the correct target).
   STORY-188's acceptance criteria as currently written cover Job/Ack_Data
   function-code classification only, not Ack/Ack_Data (`0x02`/`0x03`) error-class/code
   consumption — story-writer must add an explicit AC (or a follow-on task) to
   STORY-188 covering "log/surface the observed error class/code for an Ack- or
   Ack_Data-ROSCTR header" before that story is considered to close this requirement.
   (Deferred per human ruling, STORY-187 per-story adversarial pass 1, F-13,
   2026-09-24, re-scoped to explicitly include Ack_Data per the canonical-frame holdout
   ruling, DF-CANONICAL-FRAME-HOLDOUT-001, 2026-09-24 — this requirement is NOT
   deleted, only re-anchored.)

## Invariants

1. **Ack is structurally minimal; Ack_Data is not**: 12 bytes is the complete Ack
   (`0x02`) frame length. An Ack_Data (`0x03`) frame's 12-byte header is instead
   followed by its parameter/data block (`param_length`/`data_length` bytes), exactly
   as for Job/Userdata, just at the `header_len == 12` offset instead of `10`. Any
   `param_length`/`data_length` value extracted for a bare-Ack header is expected to be
   `0` in conformant traffic but is not validated as such by this function (a non-zero
   value on a bare-Ack frame is a downstream anomaly-detection concern, out of B1
   scope).
2. **`error_class`/`error_code` presence is exhaustively tied to `rosctr ∈ {Ack,
   AckData}`**: no other ROSCTR value (Job, Userdata) ever produces `Some` for either
   field.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | `data[1] == 0x02` (Ack), `data.len() == 10` (only the common header present) | Returns `None` — truncated Ack, malformed-header dedup |
| EC-002 | `data[1] == 0x02` (Ack), `data.len() == 11` (one byte short of the 12-byte minimum) | Returns `None` — truncated Ack |
| EC-003 | `data[1] == 0x02` (Ack), `data.len() == 12` exactly | `Some(...)` with `header_len: 12`; no parameter block (bare Ack) |
| EC-004 | `error_class == 0x00` and `error_code == 0x0000`-equivalent (no error reported), either ROSCTR | Extracted verbatim; a zero error class/code is a normal successful-Ack(_Data) value, not itself flagged |
| EC-005 | `data[1] == 0x03` (Ack_Data), `data.len() == 10` (only the common header present) | Returns `None` — truncated Ack_Data, same malformed-header dedup flag as EC-001 |
| EC-006 | `data[1] == 0x03` (Ack_Data), `data.len() == 11` (one byte short of the 12-byte minimum) | Returns `None` — truncated Ack_Data |
| EC-007 | `data[1] == 0x03` (Ack_Data), `data.len() >= 12`, non-zero `error_class`/`error_code` alongside a non-empty parameter block (real-world shape, e.g. a Setup Communication response) | `Some(...)` with `header_len: 12`; error fields extracted AND Group 3 function-code classification proceeds independently at `data[12]` (BC-2.21.010 onward). Traced to `test_BC_2_21_008_ack_data_nonzero_error_fields_with_parameter_block` — that test covers the extraction half only (error fields correctly `Some`, bounds-satisfying parameter block, no spurious finding); Group-3 FC classification at `data[12]` is STORY-188 scope (untested here). |
| EC-008 | `data[1] == 0x03` (Ack_Data), `error_class == 0x00`/`error_code == 0x00` alongside a populated parameter block (the common real-world shape corroborated by the cited canonical-frame test vectors — cnblogs/Yiqisoft/Inductive Automation KB — and consistent with the field layout documented by the cited prose source, Kleinmann & Wool 2014 (Ack_Data/ROSCTR 3 only), and permitted design references — icsnpp-s7comm, python-snap7, libs7comm (corrected 2026-09-24, F-50 — v1.6 listed K&W among the "permitted design references," a distinct ADR-014 Decision 4 category K&W does not belong to)) | Both the zero error fields and the parameter-block FC classification are extracted/performed independently; a zero error class/code on Ack_Data does not suppress FC classification |

## Canonical Test Vectors

| Input (`data`, hex bytes) | Expected result | Category |
|---|---|---|
| `32 02 00 00 00 01 00 00 00 00` (10 bytes) | `None` (truncated Ack) | reject: missing error class/code |
| `32 02 00 00 00 01 00 00 00 00 00` (11 bytes) | `None` (truncated Ack) | reject: one byte short |
| `32 02 00 00 00 01 00 00 00 00 00 00` (12 bytes) | `Some({rosctr: Ack, error_class: Some(0), error_code: Some(0), header_len: 12})` | happy-path: minimal Ack |
| `32 03 00 00 FF FF 00 08 00 00 00 00` (12 bytes) | `Some({rosctr: AckData, pdu_reference: 65535, param_length: 8, data_length: 0, error_class: Some(0), error_code: Some(0), header_len: 12})` | happy-path: Ack_Data Setup Communication response header (test-vector citation: cnblogs "西门子S7通讯协议引用整理", https://www.cnblogs.com/crcce-dncs/p/10659087.html — permitted for canonical-frame test-vector use only per ADR-014 Decision 4 (2026-09-24), not as field-semantics provenance (see Description) — full 27-byte frame `03 00 00 1B 02 F0 80 32 03 00 00 FF FF 00 08 00 00 00 00 F0 00 00 01 00 01 00 F0`; this row is the S7 header slice `data[0..12]` of that frame; the following `F0 00 00 01 00 01 00 F0` (8 bytes, matching `param_length`) is the Setup Communication parameter block starting at `data[12]`, classified by BC-2.21.010) |
| `32 03 00 00 00 01 00 00 00 00` (10 bytes) | `None` (truncated Ack_Data) | reject: Ack_Data missing error class/code |
| `32 03 00 00 00 01 00 00 00 00 00` (11 bytes) | `None` (truncated Ack_Data) | reject: Ack_Data one byte short |

## Verification Properties

| Property | Proof Method (planned) |
|----------|-------------------------|
| Ack/Ack_Data-specific 12-byte minimum is correctly enforced; `error_class`/`error_code` extraction is correct and produced exactly for ROSCTR ∈ {Ack, AckData}, never for Job/Userdata | VP-051 (Kani P0) — "S7comm Header Bounds-Before-Slice Safety," joint with BC-2.21.004/BC-2.21.006/BC-2.21.007/BC-2.21.009 (see VP Anchors below); registered F2 INTEGRATE sub-burst per VP-INDEX.md (this BC registered to VP-051's `source_bc`, F-43; sibling set expanded to five BCs under F-49); cargo-fuzz P1 (VP-055) provides complementary combined-chain no-panic coverage |

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| Capability Anchor Justification | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 — completes the ROSCTR-conditional header-length contract for Ack AND Ack_Data alongside BC-2.21.006/007 |
| L2 Domain Invariants | None directly |
| Architecture Module | SS-21 (`src/analyzer/s7comm.rs`) |
| ADR | ADR-014 Decision 9 |
| Stories | STORY-187 (parse/extraction of `error_class`/`error_code` for both Ack and Ack_Data, Postconditions 1-3); STORY-188 (consumption/logging of the extracted values for both ROSCTR values, Postcondition 4 — deferred per F-13 ruling 2026-09-24, rescoped to explicitly include Ack_Data per the canonical-frame holdout ruling DF-CANONICAL-FRAME-HOLDOUT-001, 2026-09-24; story-writer must add an explicit AC) |
| Feature | feature-s7comm |
| MITRE Techniques | T0814 (Denial of Service) — malformed-header (truncated-Ack/Ack_Data) anomaly signal only; emission wiring is a B2 responsibility |

## Related BCs

- BC-2.21.006 — composes with (shares common-header field extraction for `pdu_reference`/`param_length`/`data_length`; also composes with BC-2.21.010 onward for Ack_Data's now-`header_len == 12`-offset parameter block)
- BC-2.21.004 — composes with (shares malformed-header dedup flag)
- BC-2.21.009 — composes with (the bounds check applies identically at `header_len == 12` for both Ack and Ack_Data)
- BC-2.21.010 through BC-2.21.017 — composes with (Group 3 function-code classification reads Ack_Data's parameter block at the corrected `data[header_len]` == `data[12]` offset)

## Architecture Anchors

- `src/analyzer/s7comm.rs` — `pub fn parse_s7comm_header`, shared `0x02 | 0x03` match arm implementing the Ack/Ack_Data `header_len == 12` branch (implemented, STORY-187)
- `tests/s7comm_analyzer_tests.rs` — Tests anchor: 9 `test_BC_2_21_008_*` functions (re-counted by direct grep, verified 2026-09-25 against worktree HEAD b4fce34a): `test_BC_2_21_008_canonical_ack_data_setup_communication_response_on_data`, `test_BC_2_21_008_ack_rosctr_12_byte_minimum_and_error_fields`, `test_BC_2_21_008_canonical_ack_vector_verbatim`, `test_BC_2_21_008_ack_data_12_byte_header_and_error_fields`, `test_BC_2_21_008_truncated_ack_data_returns_none`, `test_BC_2_21_008_error_fields_none_for_job_and_userdata_rosctr`, `test_BC_2_21_008_truncated_ack_on_data_emits_t0814_once`, `test_BC_2_21_008_truncated_ack_data_on_data_emits_t0814_once`, `test_BC_2_21_008_ack_data_nonzero_error_fields_with_parameter_block`; plus the joint `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional` (shared with BC-2.21.006/007). The ninth, `test_BC_2_21_008_canonical_ack_vector_verbatim`, is newly added since the prior anchor count (8, pass 13/v1.8); it parses this BC's own canonical Ack 12-byte happy-path vector (Canonical Test Vectors table) verbatim — no byte substituted — and asserts `Some({rosctr: Ack, error_class: Some(0), error_code: Some(0), header_len: 12})` exactly. EC-007 (`data.len() >= 12`, non-zero error fields alongside a non-empty parameter block) is traced to `test_BC_2_21_008_ack_data_nonzero_error_fields_with_parameter_block` — extraction half only; Group-3 FC classification at `data[12]` is STORY-188 scope (untested here; see Edge Cases table).

## Story Anchor

STORY-187 (parse/extraction for both Ack and Ack_Data, Postconditions 1-3).
Postcondition 4's error-class/code consumption (logging) obligation is re-anchored to
STORY-188 per human ruling, F-13, 2026-09-24, rescoped to explicitly include Ack_Data
per DF-CANONICAL-FRAME-HOLDOUT-001, 2026-09-24 — see Postcondition 4.

## VP Anchors

- VP-051 (Kani P0) — S7comm Header Bounds-Before-Slice Safety; asserts this BC's
  Postconditions 2/3 (`header_len == 12` selection for `rosctr ∈ {Ack, AckData}`;
  `error_class`/`error_code` `Some` iff `rosctr ∈ {Ack, AckData}`, non-vacuity per
  DF-KANI-NONVACUITY-001); joint with BC-2.21.004, BC-2.21.006, BC-2.21.007,
  BC-2.21.009; architect registered this BC to VP-051's `source_bc` in VP-INDEX.md per
  the F-43 ruling (STORY-187 per-story adversarial pass 5, 2026-09-24), with
  BC-2.21.006/BC-2.21.007 subsequently added under the F-49 ruling (pass 7,
  2026-09-24); `source_bc` is now `{BC-2.21.004, BC-2.21.006, BC-2.21.007,
  BC-2.21.008, BC-2.21.009}` — previously `{BC-2.21.004, BC-2.21.009}` (pass 4), then
  `{BC-2.21.004, BC-2.21.008, BC-2.21.009}` (pass 5, F-43); this BC's own siblings
  BC-2.21.006/007 had been omitted despite VP-051's harness already covering their
  postconditions too
- VP-055 (cargo-fuzz P1) — S7comm/ISO-on-TCP combined parse-chain no-panic fuzz
  (`fuzz_s7comm_parser`); complementary combined-chain coverage for this Ack-length path

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | none |
| **Deterministic** | yes |
| **Thread safety** | Send + Sync |
| **Overall classification** | pure core — VP-051 (Kani P0) joint target (BC-2.21.004/006/007/009); VP-055 (cargo-fuzz P1) complementary |
