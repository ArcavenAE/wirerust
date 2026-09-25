---
document_type: behavioral-contract
level: L3
version: "1.8"
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
  - version: "1.8"
    date: 2026-09-25
    change: "STORY-187 per-story adversarial pass 13 (P13-F-1): Architecture Anchor test-count re-verification. The Architecture Anchors 'Tests anchor' entry was stale — it cited 3 tests from pass 3 (F-31), before a test added in passes 12-14. Re-grepped `tests/s7comm_analyzer_tests.rs` directly and replaced with the actual current count (4 `test_BC_2_21_004_*` functions) and the full function-name list, verified 2026-09-25 against worktree HEAD 38ff7ee1. Removed the stale 'no drift found (F-31)' claim, which no longer held. Explicitly traced EC-002 (`data.len() == 9`) to `test_BC_2_21_004_nine_byte_payload_on_data_too_short_evidence`. No change to Preconditions/Postconditions/Invariants/Edge Cases — Architecture Anchors traceability correction only."
  - version: "1.7"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 9 (N-2): Canonical Test Vectors' 8-byte-input row label ('reject: one byte short of common-header minimum minus buffer') was garbled and numerically wrong — 8 bytes is two bytes short of the 10-byte minimum, not one. Corrected to 'reject: two bytes short of the 10-byte common-header minimum'. No change to Preconditions/Postconditions/Invariants — test-vector label correction only."
  - version: "1.6"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 7 (F-49/NIT). F-49 (VP-051 source-set expansion): architect registering BC-2.21.006/BC-2.21.007 to VP-051's source_bc in VP-INDEX.md in parallel — VP-051's Kani harness (S7comm Header Bounds-Before-Slice Safety) asserts BC-2.21.006's Postconditions 1-4 (header_len == 10 field extraction for Job/Userdata, big-endian pdu_reference/param_length/data_length reads) and BC-2.21.007's Postcondition 1 (None for every unrecognized ROSCTR byte) as part of the same bounds-and-extraction proof already covering this BC and BC-2.21.008/009. This BC's Verification Properties table row ('joint with BC-2.21.008, BC-2.21.009'), Architecture Anchors entry ('joint with BC-2.21.008, BC-2.21.009'), and VP Anchors section ('traces BC-2.21.004, BC-2.21.008, BC-2.21.009') — all still naming the three-BC set from pass 6 (F-45) — corrected to the five-BC set (BC-2.21.004, BC-2.21.006, BC-2.21.007, BC-2.21.008, BC-2.21.009). NIT: EC-003 (`data.len() == 10`) cross-reference expanded from 'see BC-2.21.006/007' to 'see BC-2.21.006/007/008' — an exactly-10-byte input also reaches BC-2.21.008's Ack/Ack_Data branch (returning None, truncated, until len >= 12), so all three sibling BCs govern this edge case's downstream behavior, not just two. Swept this BC for stale provenance wording (F-50 sibling check, BC-2.21.008's Kleinmann & Wool attestation-scope defect) — no 'solely'/'only from'/'Kleinmann'/'prose sources' misattribution found here."
  - version: "1.5"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 6 (F-45/N-1): VP-051 source-set sibling sweep. F-45 (HIGH, partial-fix sibling miss): VP-INDEX.md registers VP-051's source_bc as {BC-2.21.004, BC-2.21.008, BC-2.21.009} (pass 5, F-43), but this BC's own Verification Properties table row, Architecture Anchors entry, and VP Anchors section still said 'joint with BC-2.21.009' / 'traces BC-2.21.004, BC-2.21.009' — a two-BC statement predating F-43's registration of BC-2.21.008. All three corrected to name all three source BCs (BC-2.21.004, BC-2.21.008, BC-2.21.009)."
  - version: "1.4"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 4 (F-32/F-33/F-35). F-35: Architecture Anchors' ADR-014 Decision 9 citation and the Purity Classification's 'Overall classification' row both still said 'cargo-fuzz target'/'cargo-fuzz P1 target', contradicting this BC's own Invariant 2 and Verification Properties table (already correctly stating VP-051, Kani P0, as the primary formal-verification target, with cargo-fuzz P1 (VP-055) as a complementary combined-chain harness — corrected there under F-14, pass 1). Both stale spots aligned to name VP-051 (Kani P0) primary / VP-055 (cargo-fuzz P1) complementary, consistent with Invariant 2, the VP table, and this BC's own VP Anchors section."
  - version: "1.3"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 3 (F-31), re-anchor sweep. Traceability 'Stories' field corrected from '(TBD — story-writer assigns in F3)' to 'STORY-187'. Architecture Module and Architecture Anchors' '(planned)' markers removed — `src/analyzer/s7comm.rs` and `pub fn parse_s7comm_header(data: &[u8]) -> Option<S7commHeader>` are implemented, not planned. Added a Tests anchor citing `tests/s7comm_analyzer_tests.rs`'s `mod story_187` BC-2.21.004-labeled test functions (3 tests; verified test-name/BC-ID alignment, no drift found)."
  - version: "1.2"
    date: 2026-09-24
    change: "STORY-187 canonical-frame holdout (DF-CANONICAL-FRAME-HOLDOUT-001), human ruling 2026-09-24: Ack and Ack_Data both 12-byte headers. Description corrected — Ack_Data (0x03) no longer described as part of the 10-byte-common-header group; it now requires the 12-byte header (Ack and Ack_Data require 2 additional bytes, BC-2.21.008), matching the corrected BC-2.21.006/008. No change to this BC's own `len < 10 -> None` requirement (unaffected by the ruling — the 10-byte minimum-length reject applies identically regardless of which ROSCTR ultimately needs 10 or 12 bytes)."
  - version: "1.1"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 1 (F-01/F-02/F-12/F-13/F-14), human ruling 2026-09-24. F-14: Invariant 2's wording corrected — this BC's own VP Anchors section already cited VP-051 (Kani P0) for BC-2.21.004/BC-2.21.009 jointly, but Invariant 2 and the Verification Properties table still said 'cargo-fuzz P1 ... not a Kani P0 target,' inherited stale wording from parse_asdu's fuzz-not-Kani precedent (BC-2.19.015) predating VP-051's registration. Both sections now correctly state VP-051 (Kani P0) as the primary target, with cargo-fuzz P1 (VP-055) as a complementary combined-chain no-panic harness, consistent with VP-INDEX.md. F-14 (second item): EC-001 (`data.len() == 0`) reworded — it described the framing 'payload_offset+1 slice' as if `data` starts AFTER the protocol-ID byte, contradicting this BC's own Description ('data is ... beginning at the already-classified protocol-ID byte'). Since `protocol_id: Some(0x32)` (the only way this function is reached via BC-2.21.002 dispatch) implies `data.len() >= 1`, `data.len() == 0` can never occur through the analyzer — EC-001 and the corresponding Canonical Test Vectors row are now explicitly marked direct-call-only (unit test / fuzz harness), with the `len < 10 -> None` requirement itself unchanged."
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

# BC-2.21.004: `parse_s7comm_header` Returns None for Input Shorter Than 10 Bytes

## Description

`parse_s7comm_header(data: &[u8]) -> Option<S7commHeader>` is the pure-core entry
function for classic S7comm (protocol-ID `0x32`) header parsing (ADR-014 Decision 9
item 3). Per this BC's design decision, `data` is the COTP DT payload slice beginning
**at** the already-classified protocol-ID byte (`&tpkt_payload[payload_offset..]` from
BC-2.20.009), so `data[0]` is expected to equal `0x32` (re-validated defensively, see
BC-2.21.005). The classic S7comm common header — Protocol ID (1) + ROSCTR (1) +
Reserved (2) + PDU Reference (2, big-endian) + Parameter Length (2, big-endian) + Data
Length (2, big-endian) — is exactly 10 bytes for Job/Userdata ROSCTR values (Ack and
Ack_Data both require 2 additional bytes — Error Class + Error Code — BC-2.21.008;
corrected 2026-09-24, STORY-187 canonical-frame holdout ruling,
DF-CANONICAL-FRAME-HOLDOUT-001 — Ack_Data was previously, incorrectly, grouped with
the 10-byte-only ROSCTR values). When `data.len() < 10`, the function returns `None`
immediately without accessing any field beyond the length check.

## Preconditions

1. `data` is the slice `S7commAnalyzer::on_data` passes for a DT frame already
   classified `protocol_id == Some(0x32)` (BC-2.21.002 Postcondition 3).
2. `data.len() < 10`.

## Postconditions

1. `parse_s7comm_header(data)` returns `None`.
2. No bytes beyond the length check are accessed; no panics are possible for any
   `data.len()` in `[0, 9]`.
3. The function is pure: no I/O, no global state mutation, no side effects.
4. `S7commAnalyzer` treats this `None` identically to an incomplete-TPKT-frame
   condition at the SS-20 layer: the already-extracted TPKT frame is presumed
   internally inconsistent (a TPKT frame whose declared `length` was large enough to
   be accepted by BC-2.20.004, yet whose COTP+S7comm payload does not contain a full
   10-byte S7comm header) — this is a malformed-frame condition, not a
   carry-buffer-incomplete condition (the frame-walk loop already confirmed the full
   TPKT frame was delivered; the shortfall is *within* the delivered frame). The first
   occurrence per flow direction emits one T0814 (Anomaly/Possible/Medium) finding via
   `malformed_header_reported_c2s`/`_s2c` (BC-2.21.001), mirroring IEC-104's
   malformed-ASDU-length treatment (BC-2.19.026).

## Invariants

1. **Minimum-header guard**: 10 bytes is the smallest slice from which any S7comm
   ROSCTR-driven header can be identified; not configurable.
2. **Purity**: `parse_s7comm_header` is a pure-core free function. Per VP-INDEX.md
   (VP-051, Kani P0, registered F2 INTEGRATE sub-burst), this function's `len < 10`
   reject (this BC) and BC-2.21.009's caller-side bounds check are together a Kani P0
   formal-verification target — "S7comm Header Bounds-Before-Slice Safety" — not merely
   a cargo-fuzz P1 candidate. cargo-fuzz P1 (VP-055) remains a complementary,
   later-stage harness covering the combined TPKT→COTP→S7comm parse chain's no-panic
   property under arbitrary byte input; it does not substitute for the Kani proof.
   (Reconciled with VP-INDEX.md per human ruling, STORY-187 per-story adversarial pass
   1, F-14, 2026-09-24 — this Invariant's prior wording, "not a Kani P0 target,"
   inherited from `parse_asdu`'s fuzz-only precedent (BC-2.19.015 sibling) before
   `parse_s7comm_header` had its own registered VP, is superseded now that VP-051 is
   registered Kani P0 and this BC's own VP Anchors section already cites it.)
3. **Distinct from SS-20's incomplete-frame path**: this `None` occurs on an
   *already-complete* TPKT frame per BC-2.20.004's accept path — it is a
   malformed-content finding, not a reassembly-in-progress condition, and is
   therefore dedup-flagged and finding-emitting, unlike SS-20's carry-stash `None`
   paths.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | `data.len() == 0` — **direct-call-only edge case**: exercises `parse_s7comm_header`'s standalone length guard when the function is invoked directly (unit test / fuzz harness), not reachable via `S7commAnalyzer::on_data`'s BC-2.21.002 dispatch. Per this BC's Description, `data == &tpkt_payload[payload_offset..]`, so `data[0]` IS the protocol-ID byte; the only way this function is reached in production is `protocol_id: Some(0x32)` (BC-2.21.002 Postcondition 3), which by construction implies `tpkt_payload.len() > payload_offset`, i.e. `data.len() >= 1` always holds when the analyzer calls this function — `data.len() == 0` can never occur through the dispatcher | Returns `None`; no T0814 is emitted in production for this exact input, since the analyzer can never construct it — this row validates the function's own bounds guard in isolation only (Reconciled per human ruling, STORY-187 per-story adversarial pass 1, F-14, 2026-09-24) |
| EC-002 | `data.len() == 9` (one byte short of the 10-byte minimum) | Returns `None`; T0814 emitted (first occurrence per direction) |
| EC-003 | `data.len() == 10` (exactly minimum) | Proceeds to ROSCTR/field validation; see BC-2.21.006/007/008 |
| EC-004 | A second malformed-length frame arrives on the same flow direction after the first triggered the dedup flag | Returns `None`; **no** second T0814 emitted (dedup flag already set) |

## Canonical Test Vectors

| Input (`data`, hex bytes, length) | Expected result | Category |
|---|---|---|
| `[]` (0 bytes) | `None` (direct-call-only — see EC-001; not reachable via `on_data` dispatch, no T0814 in production) | reject: empty, unit-test-level bounds check only |
| `[0x32, 0x01, 0x00, 0x00, 0x00, 0x01, 0x00, 0x02]` (8 bytes) | `None` + T0814 (first occurrence) | reject: two bytes short of the 10-byte common-header minimum |
| `[0x32, 0x01, 0x00, 0x00, 0x00, 0x01, 0x00, 0x02, 0x00]` (9 bytes) | `None` + T0814 (first occurrence) | reject: exactly one byte short |
| `[0x32, 0x01, 0x00, 0x00, 0x00, 0x01, 0x00, 0x02, 0x00, 0x00]` (10 bytes) | `Some(S7commHeader{..})` | accept: exact minimum — see BC-2.21.006 |

## Verification Properties

| Property | Proof Method (planned) |
|----------|-------------------------|
| `parse_s7comm_header` returns `None` for all inputs with `len < 10`; never panics for any symbolic input up to a bounded length | VP-051 (Kani P0) — "S7comm Header Bounds-Before-Slice Safety," joint with BC-2.21.006, BC-2.21.007, BC-2.21.008, BC-2.21.009 (see VP Anchors below); cargo-fuzz P1 (VP-055) provides complementary combined-chain no-panic coverage |

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| Capability Anchor Justification | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 — this BC is the length-reject path for the classic S7comm header parser, the entry function for all classic-S7comm dissection |
| L2 Domain Invariants | None directly (bounds-safety contract; findings-cap concerns belong to B2/INV-6) |
| Architecture Module | SS-21 (`src/analyzer/s7comm.rs`); ADR-014 Decision 9 |
| ADR | ADR-014 Decisions 2, 9 |
| Stories | STORY-187 |
| Feature | feature-s7comm |
| MITRE Techniques | T0814 (Denial of Service) — malformed-length anomaly signal only; full emission wiring (verdict/confidence/dedup call-site) is a B2 (MITRE technique-emission BC) responsibility; this BC names the obligation, does not author the emission contract |

## Related BCs

- BC-2.20.009 — depends on (the DT payload slice this function receives)
- BC-2.21.001 — depends on (`malformed_header_reported_c2s`/`_s2c` dedup flags)
- BC-2.21.005 — composes with (next rejection: protocol-ID byte mismatch when len ≥ 10)
- BC-2.21.006 — composes with (accept path)
- BC-2.19.026 — composes with (IEC-104 malformed-length EMIT-WITH-DEDUP precedent this BC mirrors)

## Architecture Anchors

- `src/analyzer/s7comm.rs` — `pub fn parse_s7comm_header(data: &[u8]) -> Option<S7commHeader>` pure-core free function; the `if data.len() < 10 { return None; }` guard is this BC's implementation
- `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md §Decision 9` — pure-core free-fn design; VP-051 (Kani P0) is the primary formal-verification target (joint with BC-2.21.006, BC-2.21.007, BC-2.21.008, BC-2.21.009), with cargo-fuzz P1 (VP-055) as a complementary combined-chain no-panic harness (aligned with Invariant 2, F-35; source-set expanded to five BCs, F-49)
- `tests/s7comm_analyzer_tests.rs` — Tests anchor: 4 `test_BC_2_21_004_*` functions (re-counted by direct grep, verified 2026-09-25 against worktree HEAD 38ff7ee1): `test_BC_2_21_004_parse_returns_none_for_len_lt_10`, `test_BC_2_21_004_len_shorter_than_10_returns_none_and_emits_t0814_once`, `test_BC_2_21_004_len_shorter_than_10_emits_t0814_once_s2c`, `test_BC_2_21_004_nine_byte_payload_on_data_too_short_evidence`. The last of these is newly added since the prior anchor count (3, pass 3, F-31); EC-002 (`data.len() == 9`, one byte short of the 10-byte minimum) is traced to `test_BC_2_21_004_nine_byte_payload_on_data_too_short_evidence`.

## Story Anchor

STORY-187

## VP Anchors

- VP-051 (Kani P0) — S7comm Header Bounds-Before-Slice Safety; registered F2
  INTEGRATE sub-burst per VP-INDEX.md; traces BC-2.21.004, BC-2.21.006, BC-2.21.007,
  BC-2.21.008, BC-2.21.009 (source_bc expanded to this five-BC set under the F-49
  ruling, STORY-187 per-story adversarial pass 7, 2026-09-24 — previously
  `{BC-2.21.004, BC-2.21.008, BC-2.21.009}`)
- VP-055 (cargo-fuzz P1) — S7comm/ISO-on-TCP combined parse-chain no-panic fuzz
  (`fuzz_s7comm_parser`); registered representative-subset source_bc includes this BC

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | none (the function itself is pure; the finding-emission/dedup consequence is the caller's responsibility) |
| **Deterministic** | yes |
| **Thread safety** | Send + Sync |
| **Overall classification** | pure core — VP-051 (Kani P0) primary formal-verification target, cargo-fuzz P1 (VP-055) complementary (aligned with Invariant 2, F-35) |
