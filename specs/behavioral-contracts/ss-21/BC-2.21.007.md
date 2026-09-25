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
    date: 2026-09-25
    change: "STORY-187 per-story adversarial pass 13 (P13-F-1): Architecture Anchor test-count re-verification. Re-grepped `tests/s7comm_analyzer_tests.rs` directly and confirmed the count (3 `test_BC_2_21_007_*` functions plus 2 proptests) is unchanged since pass 3 (F-31); listed all function names explicitly and removed the stale 'no drift found (F-31)' phrasing in favor of an explicit re-verification timestamp, verified 2026-09-25 against worktree HEAD 38ff7ee1. No change to Preconditions/Postconditions/Invariants/Edge Cases — Architecture Anchors traceability correction only."
  - version: "1.3"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 8 (N-2): dedup-flag reason-class count correction. Postcondition 3 stated the per-direction malformed-header dedup flag (`malformed_header_reported_c2s`/`_s2c`) is shared by 'the two `None`-producing conditions (too-short, unrecognized-ROSCTR)' — stale: BC-2.21.001's own Postcondition 1 already correctly cites the full four-BC set (BC-2.21.004/007/008/009) sharing this flag, and BC-2.21.008 alone covers two distinct reason classes (truncated Ack, truncated Ack_Data), so the flag is actually shared by five reason classes across those four BCs, not two. Corrected Postcondition 3 to enumerate all five: header too short (BC-2.21.004), unrecognized ROSCTR (this BC), truncated Ack and truncated Ack_Data (BC-2.21.008), and declared lengths exceeding available bytes (BC-2.21.009). BC-2.21.009 amended in the same burst for the symmetric 'all four conditions' undercount in its own Postcondition 2 (same root cause — both files predate the full five-reason-class accounting even though BC-2.21.001 and this BC's own pass-7/F-49 VP-051 five-BC sibling set already reflect it). No change to Preconditions, Postconditions 1-2, Invariants, Edge Cases, Canonical Test Vectors, or Verification Properties — wording correction confined to Postcondition 3's dedup-sharing count."
  - version: "1.2"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 7 (F-49): VP-051 source-set expansion. This BC's Postcondition 1 (parse_s7comm_header returns None for every data[1] not in {0x01, 0x02, 0x03, 0x07} — i.e. only a recognized ROSCTR byte can ever reach a Some(..) result) is asserted by VP-051's Kani harness (S7comm Header Bounds-Before-Slice Safety) as the ROSCTR-recognition half of its bounds-and-extraction proof, joint with BC-2.21.004/006/008/009 — architect registering this BC to VP-051's source_bc in VP-INDEX.md in parallel (five-BC set: BC-2.21.004, BC-2.21.006, BC-2.21.007, BC-2.21.008, BC-2.21.009). Verification Properties table row and VP Anchors section corrected from 'proptest P1 ... VP-NNN allocation still deferred' / '(None dedicated — no VP-NNN was registered ...)' to cite VP-051 (Kani P0) as the primary registered target for Postcondition 1, with the existing proptest coverage (`proptest_bc_2_21_007_rosctr_byte_totality_over_all_256_values`, joint `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional`) and cargo-fuzz P1 (VP-055) retained as complementary, exhaustive-over-all-256-values coverage that the bounded Kani proof does not itself need to re-derive. Purity Classification's 'Overall classification' row corrected from 'pure core — proptest P1 target' to 'pure core — VP-051 (Kani P0) joint target (BC-2.21.004/006/008/009); proptest P1 and cargo-fuzz P1 (VP-055) complementary'. No change to Preconditions/Postconditions/Invariants themselves — verification-anchoring correction only."
  - version: "1.1"
    date: 2026-09-24
    change: "STORY-187 per-story adversarial pass 3 (F-27/F-31), human ruling 2026-09-24 (DF-CANONICAL-FRAME-HOLDOUT-001: Ack (0x02) and Ack_Data (0x03) both require a 12-byte header; Job (0x01)/Userdata (0x07) require 10 bytes, BC-2.21.006 v1.2's accept path). F-27: this BC's v1.0 Verification Properties row stated a length-UNconditional totality ('None for exactly the 252 u8 values not in {0x01,0x02,0x03,0x07}, Some for exactly those 4') — that framing predates the canonical-frame holdout ruling and is no longer accurate: a RECOGNIZED ROSCTR byte (e.g. 0x02/0x03) still returns `None` when `data.len() < 12`, so 'recognized-ROSCTR implies Some' is false in general. Rewritten to state the joint, length-conditional totality property actually implemented and tested: `parse_s7comm_header` returns `Some` iff (`data[1] ∈ {0x01, 0x07}` and `data.len() ≥ 10`) or (`data[1] ∈ {0x02, 0x03}` and `data.len() ≥ 12`); `None` for all 252 other `data[1]` byte values, at every length, and no value ever panics. This BC's own Postconditions 1-3 are UNCHANGED by this correction — they only ever asserted `None` for `data[1] ∉ {0x01,0x02,0x03,0x07}`, independent of length, which remains true — the correction is confined to the joint VP-property framing and the Related BCs/Canonical Test Vectors context around it. Related BCs updated: BC-2.21.006 is the accept-path sibling for `{0x01, 0x07}` (`len ≥ 10`); added BC-2.21.008 as the accept-path sibling for `{0x02, 0x03}` (`len ≥ 12`) — omitted from v1.0's Related BCs entirely. Canonical Test Vectors gained an accept-path row for `0x02`/`0x03` pointing to BC-2.21.008 (v1.0 only illustrated the `0x01` accept path). F-31 re-anchor sweep (same burst): Traceability 'Stories' field corrected from '(TBD — story-writer assigns in F3)' to 'STORY-187' (Story Anchor section already correctly said STORY-187). Architecture Module and Architecture Anchors' '(planned)' markers removed — `src/analyzer/s7comm.rs` and `pub fn parse_s7comm_header`'s `_ => None` ROSCTR match-arm fallthrough are implemented, not planned. Added a Tests anchor citing `tests/s7comm_analyzer_tests.rs`'s `mod story_187` BC-2.21.007-labeled test functions, including the two totality proptests that already implement and verify this BC's corrected length-conditional property (`proptest_bc_2_21_007_rosctr_byte_totality_over_all_256_values`, `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional`) — the spec catches up to code and tests that already existed."
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

# BC-2.21.007: `parse_s7comm_header` Returns None for an Unrecognized ROSCTR Byte (Safe-Reject, No Force-Fit)

## Description

`Rosctr` (this BC's design, ADR-014 Decision 9 item 3) models exactly the four ROSCTR
values classic S7comm's steady-state traffic uses: `0x01` Job, `0x02` Ack, `0x03`
Ack_Data, `0x07` Userdata. When `data[1]` (the ROSCTR byte) is none of these four
values, `parse_s7comm_header` returns `None` rather than guessing a classification —
mirroring SS-20's `CotpTpduType` exhaustive-but-bounded design (BC-2.20.011) and
IEC-104's reserved-TypeID handling (BC-2.19.022). This is the primary safe-reject path
for malformed or non-conformant classic S7comm traffic at the ROSCTR layer.

## Preconditions

1. `data.len() >= 10`, `data[0] == 0x32` (BC-2.21.004/005 passed).
2. `data[1] ∉ {0x01, 0x02, 0x03, 0x07}`.

## Postconditions

1. `parse_s7comm_header(data)` returns `None`.
2. No panic occurs for any of the 252 remaining `u8` values not covered by the four
   recognized ROSCTR values.
3. `S7commAnalyzer` treats this `None` as a malformed-header condition, subject to the
   same per-direction T0814 dedup-and-emit treatment as BC-2.21.004's length-reject
   path (`malformed_header_reported_c2s`/`_s2c`, BC-2.21.001) — the per-direction
   malformed-header dedup flag is shared by five reason classes, not two: header too
   short (BC-2.21.004), unrecognized ROSCTR (this BC), truncated Ack and truncated
   Ack_Data (BC-2.21.008 — two distinct reason classes sharing one BC file), and
   declared lengths exceeding available bytes (BC-2.21.009). All five collectively
   answer "this frame's S7comm header could not be parsed," not five distinct anomaly
   classes each emitting its own finding.

## Invariants

1. **No force-fit**: an unrecognized ROSCTR byte is never coerced into one of the four
   modeled variants — this is a deliberate scope decision matching the "never
   force-fit" language used throughout ADR-014 (Decision 2's protocol_id table, this
   feature's dispatch philosophy generally).
2. **Exhaustive-but-bounded ROSCTR set**: `Rosctr` models exactly the values S7comm's
   steady-state protocol uses in practice; it is not exhaustive over all 256 `u8`
   values by design, symmetric with `CotpTpduType`'s three-of-many-ISO-8073-types
   scope (BC-2.20.011 Invariant 1).

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | `data[1] == 0x00` | Returns `None`; malformed-header T0814 (first occurrence per direction) |
| EC-002 | `data[1] == 0x04` (plausible off-by-one confusion with the Read Var function code, which is a parameter-block value, not a ROSCTR value) | Returns `None` — the ROSCTR field and function-code field are structurally distinct positions; no cross-field confusion is possible in a correct implementation |
| EC-003 | `data[1] == 0xFF` | Returns `None` |

## Canonical Test Vectors

| Input `data[1]` | Expected result | Category |
|---|---|---|
| `0x00` | `None` | reject: unrecognized ROSCTR |
| `0x04` | `None` | reject: unrecognized ROSCTR (not to be confused with FC 0x04) |
| `0xFF` | `None` | reject: unrecognized ROSCTR |
| `0x01` | `Some(...)` with `rosctr: Job` (requires `len ≥ 10`) | accept — see BC-2.21.006 |
| `0x02` | `Some(...)` with `rosctr: Ack` (requires `len ≥ 12`; `None` if `len < 12` even though `0x02` is a recognized ROSCTR byte) | accept — see BC-2.21.008 |

## Verification Properties

| Property | Proof Method (planned) |
|----------|-------------------------|
| `parse_s7comm_header(data)` returns `Some` **iff** (`data[1] ∈ {0x01, 0x07}` and `data.len() ≥ 10`) **or** (`data[1] ∈ {0x02, 0x03}` and `data.len() ≥ 12`); it returns `None` for all 252 remaining `data[1]` byte values, at every length, and never panics for any input. A recognized ROSCTR byte (`0x02`/`0x03`) does NOT by itself imply `Some` — it also requires the ROSCTR-specific minimum length (BC-2.21.006/008); this is the joint, length-conditional totality property across BC-2.21.006/007/008 (corrected 2026-09-24, F-27, DF-CANONICAL-FRAME-HOLDOUT-001 — supersedes this BC's v1.0 length-UNconditional framing, "`None` for exactly the 252 `u8` values not in `{0x01,0x02,0x03,0x07}`, `Some` for exactly those 4," which is false for `data[1] ∈ {0x02, 0x03}` with `data.len() < 12`) | VP-051 (Kani P0) — "S7comm Header Bounds-Before-Slice Safety," asserts this BC's Postcondition 1 (`None` for every `data[1] ∉ {0x01, 0x02, 0x03, 0x07}`, at every length, no panic), joint with BC-2.21.004, BC-2.21.006, BC-2.21.008, BC-2.21.009 (see VP Anchors below); registered F2 INTEGRATE sub-burst per VP-INDEX.md (this BC registered to VP-051's `source_bc`, F-49). Complemented by proptest P1, exhaustive over all 256 `data[1]` values and a representative length span (`[9, 10, 11, 12, 13]`) — implemented as `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional` (`tests/s7comm_analyzer_tests.rs` `mod story_187`) and `proptest_bc_2_21_007_rosctr_byte_totality_over_all_256_values`, which isolates this BC's own recognized-vs-unrecognized-ROSCTR concern at a single length (12 bytes); cargo-fuzz P1 (VP-055) provides further complementary combined-chain no-panic coverage |

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 |
| Capability Anchor Justification | CAP-21 ("S7comm Analysis") per domain/capabilities/cap-21-s7comm-analysis.md §CAP-21 — safe-reject discipline for the ROSCTR field, directly analogous to CAP-20's protocol_id no-force-fit guarantee (BC-2.20.012) |
| L2 Domain Invariants | None directly |
| Architecture Module | SS-21 (`src/analyzer/s7comm.rs`) |
| ADR | ADR-014 Decision 9 |
| Stories | STORY-187 |
| Feature | feature-s7comm |
| MITRE Techniques | T0814 (Denial of Service) — malformed-header anomaly signal only; emission wiring is a B2 responsibility |

## Related BCs

- BC-2.21.004 — composes with (shares the malformed-header dedup flag)
- BC-2.21.006 — composes with (the accept-path sibling for `data[1] ∈ {0x01, 0x07}`, requiring `data.len() ≥ 10`)
- BC-2.21.008 — composes with (the accept-path sibling for `data[1] ∈ {0x02, 0x03}`, requiring `data.len() ≥ 12`; added 2026-09-24, F-27 — omitted from v1.0's Related BCs, which predates BC-2.21.008's canonical-frame holdout correction splitting the accept paths by length; together with BC-2.21.006 these two BCs are this BC's complete set of accept-path siblings, jointly exhausting the length-conditional totality property this BC's Verification Properties table specifies)
- BC-2.20.011 — composes with (the SS-20 `CotpTpduType` no-force-fit precedent this BC mirrors)

## Architecture Anchors

- `src/analyzer/s7comm.rs` — `pub fn parse_s7comm_header`, ROSCTR `match data[1] { .. }` with a `_ => None` fallthrough arm (implemented, STORY-187)
- `tests/s7comm_analyzer_tests.rs` — Tests anchor: 3 `test_BC_2_21_007_*` functions (re-counted by direct grep, verified 2026-09-25 against worktree HEAD 38ff7ee1): `test_BC_2_21_007_unrecognized_rosctr_returns_none`, `test_BC_2_21_007_shares_dedup_flag_with_004_malformed_header`, `test_BC_2_21_007_unrecognized_rosctr_emits_t0814_once_s2c`; plus `proptest_bc_2_21_007_rosctr_byte_totality_over_all_256_values` and the joint `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional` (shared with BC-2.21.006/008; F-27's corrected VP property is already implemented and passing here).

## Story Anchor

STORY-187

## VP Anchors

- VP-051 (Kani P0) — S7comm Header Bounds-Before-Slice Safety; asserts this BC's
  Postcondition 1 (`None` for every unrecognized `data[1]`, at every length, no panic);
  joint with BC-2.21.004, BC-2.21.006, BC-2.21.008, BC-2.21.009; architect registered
  this BC to VP-051's `source_bc` in VP-INDEX.md per the F-49 ruling (STORY-187
  per-story adversarial pass 7, 2026-09-24; `source_bc` is now `{BC-2.21.004,
  BC-2.21.006, BC-2.21.007, BC-2.21.008, BC-2.21.009}` — previously `{BC-2.21.004,
  BC-2.21.008, BC-2.21.009}`, this BC and BC-2.21.006 had been omitted despite
  VP-051's harness already covering their postconditions)
- proptest P1 — exhaustive ROSCTR-byte totality over all 256 `data[1]` values
  (`proptest_bc_2_21_007_rosctr_byte_totality_over_all_256_values`, joint
  `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional`); complementary to
  VP-051's bounded Kani proof
- VP-055 (cargo-fuzz P1) — S7comm/ISO-on-TCP combined parse-chain no-panic fuzz
  (`fuzz_s7comm_parser`); further complementary combined-chain coverage

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | none |
| **Deterministic** | yes |
| **Thread safety** | Send + Sync |
| **Overall classification** | pure core — VP-051 (Kani P0) joint target (BC-2.21.004/006/008/009); proptest P1 and cargo-fuzz P1 (VP-055) complementary |
