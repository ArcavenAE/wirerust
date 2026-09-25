---
document_type: behavioral-contract
level: L3
version: "1.7"
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
  - version: "1.7"
    date: 2026-09-25
    change: "STORY-187 per-story adversarial pass 13 (P13-F-1): Architecture Anchor test-count re-verification. The Architecture Anchors 'Tests anchor' entry was stale — it cited 7 tests from pass 3 (F-31), before tests added in passes 12-14. Re-grepped `tests/s7comm_analyzer_tests.rs` directly and replaced with the actual current count (12 `test_BC_2_21_009_*` functions) and the full function-name list, verified 2026-09-25 against worktree HEAD 38ff7ee1. Removed the stale 'no drift found (F-31)' claim, which no longer held. No change to Preconditions/Postconditions/Invariants/Edge Cases — Architecture Anchors traceability correction only."
  - version: "1.6"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 8 (N-2): dedup-flag reason-class count correction, sibling to BC-2.21.007's same-burst fix. Postcondition 2 stated the per-direction `malformed_header_reported_c2s`/`_s2c` dedup flag is shared with 'BC-2.21.004/007/008 (all four conditions collectively answer ...)' — undercounted: BC-2.21.008 alone covers two distinct reason classes (truncated Ack, truncated Ack_Data; per its own Postcondition 1), so the flag is actually shared by five reason classes across the four BCs (BC-2.21.004/007/008/009), not four. Corrected to enumerate all five: header too short (BC-2.21.004), unrecognized ROSCTR (BC-2.21.007), truncated Ack and truncated Ack_Data (BC-2.21.008), and this BC's own declared-lengths-exceed-available condition — matching BC-2.21.001's Postcondition 1, which already correctly cites the full four-BC set. No change to Preconditions, Postconditions 1/3, Invariants, Edge Cases, Canonical Test Vectors, or Verification Properties — wording correction confined to Postcondition 2's dedup-sharing count."
  - version: "1.5"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 7 (F-49): VP-051 source-set sibling sweep. Verification Properties table row ('joint with BC-2.21.004, BC-2.21.008') and VP Anchors section ('traces BC-2.21.004, BC-2.21.008, BC-2.21.009') corrected to name all five of VP-051's registered source BCs (BC-2.21.004, BC-2.21.006, BC-2.21.007, BC-2.21.008, BC-2.21.009), reflecting the architect's parallel registration of BC-2.21.006/BC-2.21.007 into VP-051's source_bc under the F-49 ruling."
  - version: "1.4"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 6 (F-45/N-1): VP-051 source-set sibling sweep. F-45 (HIGH, partial-fix sibling miss): this BC's Verification Properties table row and VP Anchors section still said 'joint with BC-2.21.004' / 'traces BC-2.21.004, BC-2.21.009' — a two-BC statement predating pass 5's (F-43) registration of BC-2.21.008 into VP-051's source_bc (now {BC-2.21.004, BC-2.21.008, BC-2.21.009} per VP-INDEX.md). Both corrected to name all three source BCs."
  - version: "1.3"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 4 (F-32/F-33/F-35). F-35: the Verification Properties table row said 'Kani P0 candidate ... VP-NNN allocation deferred', contradicting this BC's own VP Anchors section (already citing the registered VP-051, Kani P0, joint with BC-2.21.004, per VP-INDEX.md v2.48) and BC-2.21.004's own VP table entry (which correctly names VP-051 as registered). Row corrected to cite VP-051 (Kani P0) as registered and VP-055 (cargo-fuzz P1) as complementary, matching the VP Anchors section and BC-2.21.004's sibling entry."
  - version: "1.2"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 3 (F-31), re-anchor sweep. Traceability 'Stories' field corrected from '(TBD — story-writer assigns in F3)' to 'STORY-187 (also a formal-hardening re-verification anchor for STORY-194)', matching the Story Anchor section below. Architecture Module and Architecture Anchors' '(planned)' markers removed — `src/analyzer/s7comm.rs` and the bounds check (implemented as the extracted pure helper `pub fn s7comm_bounds_ok(header: &S7commHeader, data_len: usize) -> bool`, called from `S7commAnalyzer::dispatch_classic_s7comm`) are implemented, not planned. Added a Tests anchor citing `tests/s7comm_analyzer_tests.rs`'s `mod story_187` BC-2.21.009-labeled test functions (7 tests; verified test-name/BC-ID alignment, no drift found)."
  - version: "1.1"
    date: 2026-09-24
    change: "STORY-187 canonical-frame holdout (DF-CANONICAL-FRAME-HOLDOUT-001), human ruling 2026-09-24: Ack and Ack_Data both 12-byte headers. Related BCs wording corrected — the `header_len: 12` bounds-check applicability was stated as 'Ack's `header_len: 12`' only; generalized to 'Ack/Ack_Data's `header_len: 12`', since Ack_Data now also carries `header_len == 12` per BC-2.21.008 v1.2. No change to this BC's own Preconditions/Postconditions/Invariants — the bounds-check obligation was already parametrized over `header.header_len` generically and required no correction."
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

# BC-2.21.009: Declared `param_length`/`data_length` Are Bounds-Checked Against Remaining Bytes Before Parameter/Data Block Access (Safe-Reject on Inconsistency)

## Description

`parse_s7comm_header` (BC-2.21.006/008) extracts `param_length` and `data_length` as
raw `u16` values without validating them against the actual number of bytes remaining
in `data` after the header. This BC specifies the caller-side (`S7commAnalyzer`)
obligation: before slicing out the parameter block (`data[header_len..header_len +
param_length]`) or the data block (`data[header_len + param_length..header_len +
param_length + data_length]`), the caller MUST verify
`data.len() >= header_len + param_length as usize + data_length as usize`. If this
check fails, the frame is treated as malformed (declared lengths exceed available
bytes) — safe-reject, no out-of-bounds slice is ever attempted. This mirrors IEC-104's
ASDU minimum-length guard (BC-2.19.015) applied to S7comm's two-length-field header
instead of ASDU's implicit body length.

## Preconditions

1. `parse_s7comm_header(data)` returned `Some(header)` (BC-2.21.006 or BC-2.21.008).
2. `data.len() < header.header_len + header.param_length as usize + header.data_length as usize`
   (declared lengths exceed what is actually present).

## Postconditions

1. No slice into `data` beyond `data.len()` is ever attempted — the bounds check
   happens strictly before any `data[header_len..]` or `data[header_len +
   param_length..]` indexing.
2. The frame is treated as malformed: `S7commAnalyzer` emits one T0814
   (Anomaly/Possible/Medium) per flow direction, guarded by the same
   `malformed_header_reported_c2s`/`_s2c` dedup flag as BC-2.21.004/007/008 (five
   reason classes share this one per-direction flag, not four: header too short
   (BC-2.21.004), unrecognized ROSCTR (BC-2.21.007), truncated Ack and truncated
   Ack_Data (BC-2.21.008 — two distinct reason classes sharing one BC file), and this
   BC's declared-lengths-exceed-available condition — all five collectively answer
   "was this frame's declared structure internally consistent with its actual byte
   length?").
3. No function-code or Userdata classification (BC-2.21.010 onward) is attempted for a
   frame that fails this check — classification always requires a successfully
   bounds-validated parameter block.

## Invariants

1. **`u16 + u16` cannot overflow `usize`**: on any platform wirerust targets (32-bit or
   64-bit), `header_len (10 or 12) + param_length (max 65,535) + data_length (max
   65,535)` fits comfortably within `usize::MAX`; no arithmetic overflow is possible
   in this bounds computation.
2. **Bounds check precedes every downstream slice**: this is the single choke point
   through which all classification logic (Groups 3 and 4) must pass; no BC in this
   feature ever slices the parameter or data block without this check having already
   succeeded.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | `param_length == 0` and `data_length == 0` and `data.len() == header_len` exactly | Bounds check passes trivially (`data.len() >= header_len + 0 + 0`); an empty parameter block is legitimate (e.g. some Ack_Data responses) — classification proceeds to the empty-parameter-block case (BC-2.21.017's "no FC byte present" edge) |
| EC-002 | `param_length == 65,535` (maximum representable `u16`) but only 20 bytes actually follow the header | Bounds check fails; malformed-header T0814 (dedup-guarded) |
| EC-003 | `data_length` is large and plausible (e.g. a multi-kilobyte Download Block payload) and the TPKT frame's declared length (SS-20) was large enough to carry it | Bounds check passes; this is the expected shape for large block-download traffic (ADR-014 Decision 8's rationale for the 65,535-byte carry-buffer ceiling) |

## Canonical Test Vectors

| `header.param_length` / `header.data_length` / `data.len() - header_len` | Expected outcome | Category |
|---|---|---|
| `2` / `0` / `2` (exact match) | Bounds check passes | happy-path |
| `2` / `0` / `1` (one byte short) | Bounds check fails; malformed-header T0814 | reject: declared length exceeds available bytes |
| `0` / `0` / `0` (empty parameter and data blocks) | Bounds check passes trivially | edge-case: no function code present |

## Verification Properties

| Property | Proof Method (planned) |
|----------|-------------------------|
| No out-of-bounds slice is ever constructed from `header_len`, `param_length`, and `data_length` for any combination of `u16` values and any `data.len()` | VP-051 (Kani P0) — "S7comm Header Bounds-Before-Slice Safety," joint with BC-2.21.004, BC-2.21.006, BC-2.21.007, BC-2.21.008 (see VP Anchors below); registered F2 INTEGRATE sub-burst per VP-INDEX.md (arithmetic/bounds safety over the full `u16 × u16` space is small enough for exhaustive symbolic proof); cargo-fuzz P1 (VP-055) provides complementary combined-chain no-panic coverage (F-35) |

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| Capability Anchor Justification | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 — the bounds gate that makes all downstream function-code classification memory-safe |
| L2 Domain Invariants | None directly (bounds-safety contract) |
| Architecture Module | SS-21 (`src/analyzer/s7comm.rs`) |
| ADR | ADR-014 Decision 9 |
| Stories | STORY-187 (also a formal-hardening re-verification anchor for STORY-194) |
| Feature | feature-s7comm |
| MITRE Techniques | T0814 (Denial of Service) — malformed-header anomaly signal only; emission wiring is a B2 responsibility |

## Related BCs

- BC-2.21.006 — depends on (`param_length`/`data_length` values this BC validates)
- BC-2.21.008 — depends on (same validation applies to Ack/Ack_Data's `header_len: 12`)
- BC-2.21.010 through BC-2.21.023 — depend on (this bounds check is a precondition for every classification BC)
- BC-2.19.015 — composes with (IEC-104 ASDU minimum-length guard precedent this BC mirrors)

## Architecture Anchors

- `src/analyzer/s7comm.rs` — `pub fn s7comm_bounds_ok(header: &S7commHeader, data_len: usize) -> bool` pure-core free-function bounds check (implemented, STORY-187), called from the private helper `fn dispatch_classic_s7comm` immediately after `parse_s7comm_header` returns `Some`, before any parameter/data-block slicing — extracted as a standalone `pub fn` (rather than inlined at the call site) so the VP-051 Kani harness can call it directly
- `tests/s7comm_analyzer_tests.rs` — Tests anchor: 12 `test_BC_2_21_009_*` functions (re-counted by direct grep, verified 2026-09-25 against worktree HEAD 38ff7ee1): `test_BC_2_21_009_dissection_bounded_to_own_tpkt_frame`, `test_BC_2_21_009_bounds_check_before_parameter_data_slice`, `test_BC_2_21_009_bounds_check_dedup_s2c`, `test_BC_2_21_009_s7comm_bounds_ok_helper_matches_bounds_decision`, `test_BC_2_21_009_bounds_check_passes_exact_match`, `test_BC_2_21_009_empty_parameter_and_data_blocks_trivial_pass`, `test_BC_2_21_009_overflow_free_arithmetic_max_values`, `test_BC_2_21_009_ack_header_len_12_bounds_check`, `test_BC_2_21_009_data_length_overrun_on_data_emits_t0814`, `test_BC_2_21_009_s7comm_bounds_ok_data_length_only_overrun`, `test_BC_2_21_009_bounds_failure_evidence_reports_declared_and_available`, `test_BC_2_21_009_bounds_failure_evidence_ack_data_header_len_12`.

## Story Anchor

STORY-187 (also a formal-hardening re-verification anchor for STORY-194)

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
| **Global state access** | reads header fields only; the emit-on-failure consequence touches per-flow dedup state |
| **Deterministic** | yes |
| **Thread safety** | Send + Sync (the bounds-check arithmetic itself; caller-side finding emission follows the analyzer's single-flow-owner pattern) |
| **Overall classification** | pure core (bounds arithmetic) — Kani P0 target |
