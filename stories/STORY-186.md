---
document_type: story
level: ops
story_id: STORY-186
title: "S7comm ISO-on-TCP Carry-Buffer Reassembly, Walk-First Frame Extraction, Resync, and the Frozen SS-20/SS-21 Module Boundary"
epic_id: E-23
version: "1.2"
status: delivered
producer: story-writer
timestamp: 2026-09-06T00:00:00Z
phase: f3
traces_to: .factory/specs/prd.md
points: 5
priority: P1
cycle: feature-s7comm
wave: 89
target_module: analyzer/s7comm
subsystems: [SS-20, SS-21]
estimated_days: null
assumption_validations: []
risk_mitigations: []
tdd_mode: strict
feature_id: feature-s7comm
depends_on: [STORY-185]
blocks: [STORY-187]
behavioral_contracts: [BC-2.20.013, BC-2.20.014, BC-2.20.015, BC-2.20.016, BC-2.21.003]
verification_properties: [VP-050]
inputs:
  - .factory/specs/behavioral-contracts/ss-20/BC-2.20.013.md
  - .factory/specs/behavioral-contracts/ss-20/BC-2.20.014.md
  - .factory/specs/behavioral-contracts/ss-20/BC-2.20.015.md
  - .factory/specs/behavioral-contracts/ss-20/BC-2.20.016.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.003.md
  - .factory/specs/architecture/ARCH-INDEX.md
  - docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md
  - .factory/cycles/feature-s7comm/f2-pcap-fixture-sourcing.md
input-hash: "259af26"
---

> **tdd_mode:** `strict` — full TDD Iron Law enforced.

# STORY-186: S7comm ISO-on-TCP Carry-Buffer Reassembly, Walk-First Frame Extraction, Resync, and the Frozen SS-20/SS-21 Module Boundary

## Narrative

**As a** security analyst using wirerust to inspect S7comm traffic spanning TCP segment
boundaries,
**I want** `S7commAnalyzer` to correctly reassemble TPKT frames split across `on_data`
calls via directional carry buffers, resync on malformed input without ever discarding
an already-complete frame, and enforce the architectural boundary that keeps SS-20
stateless and S7comm-agnostic,
**so that** multi-segment TPKT delivery is handled correctly, carry-overflow DoS attempts
are detected (T0814), and a future IEC 61850 MMS or ICCP/TASE.2 cycle can reuse SS-20
without modification.

This story creates `src/analyzer/s7comm.rs` (SS-21) for the first time: `S7commAnalyzer`,
a minimal `S7commFlowState` (carry fields only — extended with the remaining
classification/dedup fields in STORY-187), and the frame-walk loop in `on_data` that
drives STORY-184/185's `parse_tpkt_header`/`parse_cotp_header`. Protocol-specific
dispatch on the extracted `CotpHeader::protocol_id` (the protocol-ID dispatch contract
delivered by STORY-187) is **not** built here — this story only proves that complete
TPKT frames are correctly extracted, walked, and
carried across calls. STORY-187 wires the extraction output into S7comm-specific
classification.

## Behavioral Contracts

| BC ID | Title | Story Role |
|-------|-------|-----------|
| BC-2.20.013 | TPKT Frames Spanning TCP Segment Boundaries Are Reassembled via Directional Carry Buffers Using Walk-First, Residual-Bound Semantics | Frame-walk loop, no aggregate pre-check |
| BC-2.20.014 | Carry-Overflow Bound (`MAX_S7_ISO_ON_TCP_CARRY_BYTES = 65,535`) and T0814 Guard — Defense-in-Depth, Unreachable by Construction Under Walk-First Design | Overflow reaction + dedup |
| BC-2.20.015 | Resync Anchor Advances Exactly 1 Byte Per Iteration on a Bad TPKT Version Byte (Never 2) | Resync correctness |
| BC-2.20.016 | Frozen `iso_on_tcp.rs` Module Boundary — Pure Free Functions Only, No StreamAnalyzer Impl, No Per-Flow State of Its Own | Architectural boundary contract |
| BC-2.21.003 | `on_flow_close` Removes `S7commFlowState` and Discards All Carry Bytes | Flow lifecycle teardown (packaged here with flow-map creation, not with STORY-187's classification dispatch) |

## Acceptance Criteria

### AC-186-001: Frame-walk loop extracts every complete TPKT frame before any byte-count bound is applied
(traces to BC-2.20.013 postcondition 1)
- Given `working = carry[direction] ++ incoming_data` for a flow's `on_data` call
- When the frame-walk loop runs
- Then it repeatedly calls `parse_tpkt_header(&working[cursor..])`: a complete frame
  (`Some(header)` and `working.len() - cursor >= header.length as usize`) is extracted
  and dispatched to `parse_cotp_header`, `cursor += header.length as usize`, and the loop
  continues; a declared-but-incomplete frame or a `None` result breaks the loop and
  stashes `working[cursor..]` to `carry[direction]` (traces to BC-2.20.013
  postcondition 1, sub-clauses a/b/c)
- No aggregate `carry[direction].len() + incoming_data.len()` pre-check exists anywhere
  in the implementation — the walk always runs first (traces to BC-2.20.013
  postcondition 2, invariant 1)
- **Test:** `test_BC_2_20_013_walk_first_no_aggregate_precheck`

### AC-186-002: An adversarial burst with a complete frame at the head is never dropped
(traces to BC-2.20.013 invariant 1)
- Given one `on_data` call delivering `[complete 7-byte CR frame][60,000 bytes of
  trailing garbage]`
- When `on_data` processes this delivery
- Then the 7-byte CR frame is extracted regardless of the trailing garbage's size — this
  is the anti-evasion property (Ptacek/Newsham-class evasion channel) this design
  prevents, mirroring the IEC-104 F-172-001 and DNP3 F-B-002 rulings
- **Test:** `test_BC_2_20_013_adversarial_burst_head_frame_not_dropped`

### AC-186-003: Split-frame reassembly across two on_data calls
(traces to BC-2.20.013 edge case EC-002)
- Given call 1 delivers `[0x03, 0x00, 0x00, 0x0A]` (4-byte TPKT header declaring
  `length=10`) and call 2 delivers the remaining 6 bytes
- When both calls complete
- Then call 1 stashes the 4-byte header-only partial to carry (declared-but-incomplete);
  call 2's `working = carry ++ new_bytes` contains the complete 10-byte frame, which is
  extracted and `carry[direction]` is empty afterward
- **Test:** `test_BC_2_20_013_split_frame_across_two_calls`

### AC-186-004: Carry buffer bounded at 65,535 bytes — (a) SYNTHETIC strict-`>` boundary check at the literal (unrealizable) 65,535 value, and (b) LIVE near-bound check at the actual maximum reachable residual of 65,534 bytes
(traces to BC-2.20.014 invariant 1) (part (a) traces to BC-2.20.014 edge case EC-006; part (b) traces to BC-2.20.014 edge case EC-001, edge case EC-002)

**CORRECTION (STORY-186 v1.2, consistency audit finding #2, BC-2.20.014 v1.2):** the v1.1
text of this AC claimed the exactly-65,535-byte at-bound case "IS reachable via the real
`on_data` walk-first data path." This was incorrect: under the walk-first design, a
carry residual of exactly 65,535 bytes is UNREALIZABLE via `on_data` — a residual that
reaches the full 65,535 bytes is itself a complete, dispatchable TPKT frame and would be
extracted by the walk, not stashed to carry (BC-2.20.014 v1.2 Invariant 1). The actual
maximum residual reachable via real `on_data` traffic is 65,534 bytes (a declared
`length=65,535` frame missing exactly its final byte). This AC is therefore split into
two explicit parts:

**(a) SYNTHETIC strict-`>` boundary check (literal 65,535 value — NOT reachable via
`on_data`; same synthetic direct-field-injection labeling convention as AC-186-005/006;
traces to BC-2.20.014 edge case EC-006):**
- Given a residual of exactly 65,535 bytes, directly seeded onto
  `S7commFlowState.carry_c2s` via field injection (a complete, conformant max-length TPKT
  frame constructed by `max_length_frame()` and assigned directly to the carry field,
  bypassing the `on_data` walk-first path entirely), followed by an `on_data` call with an
  empty delivery
- When the residual-bound check runs
- Then no overflow is triggered (comparison is strict `>`, not `>=`) — this proves the
  defense-in-depth guard's comparison boundary (BC-2.20.014 v1.2 Invariant 1) sits exactly
  at the TPKT `length` field's maximum representable value (`u16::MAX`), not merely
  "large." This is a SYNTHETIC guard-boundary check via direct field injection, not a
  live `on_data`-reachable scenario — the literal value 65,535 can never actually occur as
  a stashed residual on the real data path
- **Test:** `test_BC_2_20_014_at_bound_residual_no_overflow` (existing test from STORY-186
  v1.0/v1.1, relabeled SYNTHETIC by this amendment; no test code change is made by this
  story-file edit — a follow-up fix PR must update the test's doc comment/labeling to
  match)

**(b) LIVE near-bound check (the actual maximum reachable residual — real `on_data`
traffic; NEW, traces to BC-2.20.014 v1.2 Canonical Test Vectors "Near-bound, legitimate"
row and edge case EC-001; test not yet implemented):**
- Given a declared `length=65,535` TPKT frame delivered via real `on_data` calls, minus
  its final byte (65,534 bytes total — the largest residual the walk-first path can ever
  stash to carry), delivered either in a single call (BC-2.20.014 edge case EC-001) or
  split across multiple segments/calls as progressive accumulation to the same peak
  residual (BC-2.20.014 edge case EC-002)
- When all but the final byte have been delivered
- Then `carry[direction]` holds exactly 65,534 bytes at every observation point,
  including intermediate accumulation steps under the multi-call path; no finding is
  emitted; the carry-overflow dedup flag (`carry_overflow_reported_c2s`/`_s2c`) remains
  unset
- When the final byte is then delivered via a subsequent `on_data` call (traces to
  BC-2.20.014 edge case EC-002's final-byte-completion step)
- Then the frame is extracted as complete and `carry[direction]` is empty afterward
- **Test:** `test_BC_2_20_014_live_near_bound_residual_reachable` (NEW — not yet written;
  this story-file amendment specifies the requirement only. A follow-up fix PR must add
  this test; src/test/docs changes are not made by this amendment)

### AC-186-005: [DEFENSE-IN-DEPTH, unreachable via `on_data`] Carry-overflow guard mechanics — clears carry, resyncs, emits exactly one T0814 per direction, IF the guard's precondition is ever reached
(traces to BC-2.20.014 postcondition 1) (traces to BC-2.20.014 postcondition 3)
- **Reclassification (BC-2.20.014 v1.1, STORY-186 adversarial gate F-02/F-03, two
  independent passes, human ruling 2026-09-07 — Option B: Defense-in-Depth):**
  `residual.len() > 65,535` is provably unreachable via the real `on_data` data path
  under the current BC-2.20.013 walk-first + BC-2.20.015 1-byte-resync design — TPKT
  `length` is u16-capped, so the directional carry is bounded `≤ 65,534` bytes by
  construction for both conformant and adversarial input (BC-2.20.013 Reconciliation
  Note). This AC therefore specifies the guard's mechanics as a **structural
  defense-in-depth safety net** against a future design regression in BC-2.20.013 or
  BC-2.20.015 — **not** as a live runtime detection exercised by real traffic today. The
  tests below exercise the guard by **directly constructing/injecting an oversized
  `S7commFlowState.carry_c2s`/`carry_s2c`** (bypassing the normal `on_data` walk-first/
  resync path entirely) — this is the SYNTHETIC guard-mechanics vector named in
  BC-2.20.014 v1.1's Canonical Test Vectors table, not a scenario reachable by feeding
  bytes through `on_data`.
- Given `residual.len() > MAX_S7_ISO_ON_TCP_CARRY_BYTES` (i.e. `> 65,535`), directly
  constructed on the directional carry field at call-entry — before the current
  delivery is appended and the walk begins — per BC-2.20.014 v1.1's F-03 reconciliation,
  which retains this call-entry check placement as equivalent to (and reconciled with)
  the BC's originally-specified after-walk-same-call precondition, since Invariant 1
  proves neither placement can ever observe a value exceeding the bound on real traffic
- When the overflow check fires (IF reached)
- Then `carry[direction]` is cleared (not truncated); the walk resyncs (BC-2.20.015); and
  exactly one T0814 finding (`ThreatCategory::Anomaly`, `Verdict::Possible`,
  `Confidence::Medium`) is emitted for this direction, guarded by
  `carry_overflow_reported_c2s`/`_s2c` (traces to BC-2.20.014 postcondition 4 — this
  dedup flag is distinct from any malformed-header dedup flag introduced in STORY-187).
  These mechanics (clear-not-truncate, resync, one-T0814-per-direction, dedup) remain the
  BINDING SPECIFICATION for the guard's behavior IF it is ever reached (BC-2.20.014 v1.1
  Postconditions 1-4 / Invariants 2-4)
- A second overflow event, directly constructed in the same direction on the same flow,
  does not re-emit (traces to BC-2.20.014 edge case EC-004 — same SYNTHETIC/
  direct-injection reachability caveat as above)
- **Test:** `test_BC_2_20_014_overflow_clear_resync_one_t0814_per_direction`,
  `test_BC_2_20_014_repeated_overflow_dedup_same_direction` (both exercise the guard via
  direct flow-state construction, not via `on_data`)

- **NEW positive assertion — on_data-driven unreachability (BC-2.20.014 v1.2 Invariant 1
  / VP-050 clause (c) REACHABLE-BOUND INVARIANT, VP-INDEX.md v2.49):** feeding a real
  garbage flood through `on_data` (e.g.
  200,000 bytes of non-`0x03`-anchored garbage delivered across one or many calls, with
  no valid TPKT frame ever presented) does **NOT** emit a T0814 for either direction, and
  the directional carry stays bounded `≤ 65,534` bytes at every observation point — the
  resync sub-routine (BC-2.20.015) drains un-anchored garbage below 4 remaining bytes
  before each call's walk terminates, so garbage never accumulates carry-to-carry across
  calls (traces to BC-2.20.014 postcondition-negation via Invariant 1 / BC-2.20.013
  Reconciliation Note — this is the positive, on-`on_data`-path counterpart to the
  SYNTHETIC guard-mechanics tests above)
- **Test:** `test_BC_2_20_014_overflow_unreachable_via_on_data`

### AC-186-006: [DEFENSE-IN-DEPTH, unreachable via `on_data`] Overflow dedup flags are independent per direction — guard mechanics, IF reached
(traces to BC-2.20.014 edge case EC-005)
- **Reclassification note:** as with AC-186-005, this AC exercises guard mechanics only
  reachable via SYNTHETIC direct flow-state construction (BC-2.20.014 v1.1 Canonical Test
  Vectors), not via the real `on_data` data path — see AC-186-005's reclassification
  preamble for the full rationale
- Given an overflow event directly constructed in the `c2s` direction on a flow
- When an independent overflow event is subsequently directly constructed in the `s2c`
  direction on the same flow
- Then the `s2c` overflow emits its own T0814 finding — the `c2s` dedup flag has no
  bearing on `s2c`. This per-direction independence remains the binding specification for
  the guard's dedup mechanics IF the guard is ever reached (BC-2.20.014 v1.1
  Postcondition 4 / Invariant 4)
- **Test:** `test_BC_2_20_014_overflow_dedup_independent_per_direction` (exercises the
  guard via direct flow-state construction, not via `on_data`)

### AC-186-007: Resync advances exactly 1 byte per iteration on a bad version byte, never 2
(traces to BC-2.20.015 postcondition 1) (traces to BC-2.20.015 invariant 1)
- Given bytes `[0x01, 0x03, 0x00, 0x00, 0x07]` (a spurious `0x01` immediately followed by
  a valid frame at offset 1, `length=7` — BC-2.20.015's canonical vector; corrected from
  an earlier `length=4` example, which `parse_tpkt_header` would reject outright since
  BC-2.20.003 requires `length >= 7`)
- When the frame-walk loop's resync sub-routine runs
- Then the valid frame at offset 1 is found; a 2-byte advance would have skipped it
  entirely (landing at offset 2, `0x00`)
- **Test:** `test_BC_2_20_015_resync_advances_exactly_one_byte`

### AC-186-008: Resync sub-routine is reused verbatim for both bad-version-byte and post-overflow conditions
(traces to BC-2.20.015 invariant 3)
- Given a bad-version-byte condition encountered mid-stream (per STORY-184's version-byte
  reject path) and a post-carry-overflow resync (BC-2.20.014)
- When either condition triggers a resync
- Then both invoke the same 1-byte-advance resync sub-routine — there is exactly one
  resync implementation, not two
- **Test:** `test_BC_2_20_015_single_resync_implementation_shared`

### AC-186-009: Resync always terminates for finite input
(traces to BC-2.20.015 invariant 2)
- Given a long run of non-`0x03` garbage bytes (e.g. 200 bytes) with no valid frame
  anywhere in the remaining input
- When the resync sub-routine runs
- Then it advances to the end of the input without an infinite loop; the (now sub-4-byte)
  remainder is stashed to carry per the ordinary incomplete-frame path
- **Test:** `test_BC_2_20_015_resync_terminates_no_valid_anchor`

### AC-186-010: `iso_on_tcp.rs` contains no `impl StreamAnalyzer` block
(traces to BC-2.20.016 postcondition 1)
- Given `src/analyzer/iso_on_tcp.rs`
- When a static regression-guard test inspects the module
- Then it contains zero `impl StreamAnalyzer for ...` blocks and zero
  `DispatchTarget::IsoOnTcp`-shaped references
- **Test:** `test_BC_2_20_016_iso_on_tcp_has_no_stream_analyzer_impl` (grep-equivalent
  static assertion)

### AC-186-011: TPKT/COTP carry buffers live on `S7commFlowState`, not a separate `IsoOnTcpFlowState`
(traces to BC-2.20.016 postcondition 3)
- Given `S7commFlowState` (created in this story)
- When its fields are inspected
- Then `carry_c2s: Vec<u8>`, `carry_s2c: Vec<u8>`, `carry_overflow_reported_c2s: bool`,
  `carry_overflow_reported_s2c: bool` are fields on `S7commFlowState` (SS-21); no
  `IsoOnTcpFlowState` type exists anywhere in the tree
- **Test:** `test_BC_2_20_016_no_iso_on_tcp_flow_state_type_exists` (grep-equivalent
  static assertion)

### AC-186-012: `on_flow_close` removes `S7commFlowState` and discards carry bytes with no finding
(traces to BC-2.21.003 postconditions 1-4, forward-referenced here as flow-lifecycle
infrastructure this story's `on_data`/`on_flow_close` pair requires to exist; the full
`S7commFlowState` struct is completed in STORY-187)
- Given a flow with active `S7commFlowState` (possibly non-empty carry buffers)
- When `S7commAnalyzer::on_flow_close(flow_key)` is called
- Then the flow's state is removed from the analyzer's per-flow map; carry bytes are
  dropped with no finding emitted; calling `on_flow_close` for an unknown `flow_key` is a
  no-op
- **Test:** `test_s7comm_on_flow_close_removes_state_discards_carry`

## Architecture Mapping

| Component | Module | File | Pure/Effectful |
|-----------|--------|------|---------------|
| `S7commAnalyzer` struct | SS-21 analyzer | `src/analyzer/s7comm.rs` | Effectful (owns `flows: HashMap<FlowKey, S7commFlowState>`) |
| `S7commFlowState` (minimal: carry fields only) | SS-21 per-flow state | `src/analyzer/s7comm.rs` | Mutable state |
| `MAX_S7_ISO_ON_TCP_CARRY_BYTES` const | SS-21 constants | `src/analyzer/s7comm.rs` | N/A |
| `S7commAnalyzer::on_data` (frame-walk loop only) | SS-21 effectful shell | `src/analyzer/s7comm.rs` | Effectful |
| `S7commAnalyzer::on_flow_close` | SS-21 lifecycle | `src/analyzer/s7comm.rs` | Effectful |
| `parse_tpkt_header`, `parse_cotp_header` (consumed, unchanged) | SS-20 | `src/analyzer/iso_on_tcp.rs` | Pure (unchanged by this story) |

Subsystem anchors:
- SS-21 owns `S7commAnalyzer`/`S7commFlowState`/`on_data`/`on_flow_close` per
  ARCH-INDEX.md §SS-21 — this is the first story to create `s7comm.rs`.
- SS-20 owns the frozen module-boundary contract (BC-2.20.016) this story verifies:
  `iso_on_tcp.rs` remains untouched, stateless, and the sole consumer relationship
  (SS-21 imports SS-20's pure functions) is established here for the first time.

## Purity Classification

| Module | Classification | Justification |
|--------|---------------|---------------|
| `parse_tpkt_header`, `parse_cotp_header` (from STORY-184/185) | pure-core | Unchanged; consumed by value |
| `S7commAnalyzer::on_data` | effectful-shell | Mutates `S7commFlowState` (carry buffers, dedup flags), calls `emit_finding`-equivalent for T0814 on overflow |
| `S7commAnalyzer::on_flow_close` | effectful-shell | Removes map entry |

## VP-050 Proptest Obligation

**Harnesses:** `proptest_vp050_walk_first_residual_bound`,
`proptest_vp050_direction_isolation`, `proptest_vp050_resync_one_byte_advance`
(anchored in this story)
**Method:** proptest
**Priority:** P1

Skeleton (in `tests/s7comm_analyzer_tests.rs`):

```rust
proptest! {
    #[test]
    fn proptest_vp050_direction_isolation(
        c2s_data in prop::collection::vec(any::<u8>(), 0..300),
        s2c_data in prop::collection::vec(any::<u8>(), 0..300),
    ) {
        let mut analyzer = S7commAnalyzer::new();
        let flow_key = FlowKey::new(
            "127.0.0.1".parse().unwrap(), 1234,
            "127.0.0.2".parse().unwrap(), 102,
        );
        analyzer.on_data(flow_key.clone(), &c2s_data, 0, Direction::ClientToServer);
        analyzer.on_data(flow_key.clone(), &s2c_data, 0, Direction::ServerToClient);
        // carry_c2s must only ever contain bytes routed via C2S; carry_s2c only S2C;
        // each carry stays <= 65,534 bytes (BC-2.20.014 v1.2 Invariant 1 -- the maximum
        // reachable via real `on_data` traffic; the guard constant
        // MAX_S7_ISO_ON_TCP_CARRY_BYTES = 65,535 is itself unreachable as a residual).
    }
}
```

Full proptest run (including the walk-first equivalence property: splitting a byte
sequence into `carry + incoming` yields the identical result as running the walk once on
the concatenated bytes) is executed in STORY-194.

## Tasks

- [ ] Create `src/analyzer/s7comm.rs` with a module-level doc comment citing ADR-014
      Decisions 1, 2, 8 (SS-21 owns the frame-walk loop and all per-flow state; SS-20
      remains stateless)
- [ ] Define `const MAX_S7_ISO_ON_TCP_CARRY_BYTES: usize = 65_535;`
- [ ] Define a minimal `S7commFlowState` with exactly: `carry_c2s: Vec<u8>`,
      `carry_s2c: Vec<u8>`, `carry_overflow_reported_c2s: bool`,
      `carry_overflow_reported_s2c: bool` (STORY-187 extends this struct with the
      classification/dedup fields its own scope requires — do not pre-add those fields
      here)
- [ ] Implement `S7commAnalyzer` struct with `flows: HashMap<FlowKey, S7commFlowState>`
- [ ] Implement `S7commAnalyzer::on_data(&mut self, flow_key: FlowKey, data: &[u8],
      ts: u32, direction: Direction)`:
  - overflow check at entry on the directional carry (before appending/walking) per
    BC-2.20.014 walk-first-residual-bound semantics
  - frame-walk loop: extract complete TPKT frames via `iso_on_tcp::parse_tpkt_header`
    then `iso_on_tcp::parse_cotp_header`; for this story, dispatch is a no-op placeholder
    (classification lands in STORY-187) — the loop's job here is proven extraction and
    carry management only
  - bad-version-byte / post-overflow resync: 1-byte advance, shared sub-routine
    (BC-2.20.015)
- [ ] Implement `S7commAnalyzer::on_flow_close(&mut self, flow_key: FlowKey)`:
  `self.flows.remove(&flow_key)`
- [ ] Write `proptest_vp050_*` skeletons in `tests/s7comm_analyzer_tests.rs`
- [ ] Write unit tests: one per AC, named `test_BC_2_20_013_*` .. `test_BC_2_20_016_*`
- [ ] Write the two static regression-guard tests (AC-186-010, AC-186-011)
- [ ] Verify `cargo test` passes
- [ ] Add a CHANGELOG entry under `[Unreleased] > Added` describing the new
      `S7commAnalyzer` skeleton and carry-buffer reassembly, before creating the PR

## Edge Cases

| ID | Source BC | Description | Expected Behavior |
|----|-----------|-------------|-------------------|
| EC-001 | BC-2.20.013 | Single `on_data` call delivers exactly one complete frame, no carry before/after | Frame extracted; carry remains empty |
| EC-002 | BC-2.20.013 | TCP segment delivers two complete frames back-to-back plus a partial third | Both complete frames extracted in the same call; only the partial third stashed |
| EC-003 | BC-2.20.014 | `residual.len() == 65,535` exactly — SYNTHETIC direct field injection only (literal boundary value; UNREALIZABLE via real `on_data` traffic, since a residual reaching the full 65,535 bytes would itself be a complete, dispatchable frame and would be extracted rather than stashed — BC-2.20.014 v1.2 Invariant 1; AC-186-004(a)) | No overflow; comparison is strict `>`, not `>=` |
| EC-004 | BC-2.20.014 | `residual.len() == 65,534` exactly — the actual maximum residual reachable via real `on_data` traffic (a declared `length=65,535` frame missing exactly its final byte; BC-2.20.014 v1.2 Invariant 1 / EC-001; AC-186-004(b)) | No overflow; carry retains all 65,534 bytes; no finding |
| EC-005 | BC-2.20.014 | `residual.len() == 65,536` (one over bound) — SYNTHETIC direct field injection only, UNREALIZABLE via real `on_data` traffic (same reachability caveat as AC-186-005/006) | Carry cleared; resync; exactly one T0814 |
| EC-006 | BC-2.20.015 | `0x03` byte exists but starts a frame with an invalid length field (`< 7`, BC-2.20.003) | Resync finds this `0x03`, `parse_tpkt_header` returns `None` again (different reject reason), walk continues advancing 1 byte past it — never stuck retrying the same offset |
| EC-007 | BC-2.20.016 | A future MMS/ICCP cycle wants to reuse `parse_tpkt_header`/`parse_cotp_header` | Imports them directly; defines its own analogous flow-state fields — zero lines of `iso_on_tcp.rs` change |

## Token Budget Estimate

| Context Source | Estimated Tokens |
|---------------|-----------------|
| This story spec | ~5,200 |
| BC-2.20.013-016 (4 BCs, higher density) | ~6,000 |
| ADR-014 (Decisions 1, 2, 8, 9) | ~10,000 |
| src/analyzer/iso_on_tcp.rs (from STORY-184/185) | ~4,000 |
| Test file (new `s7comm_analyzer_tests.rs`) | ~2,500 |
| **Total** | **~27,700** |
| Agent context window | 200K for Sonnet |
| **Budget usage** | **~14%** |

## Previous Story Intelligence

| Story | Key Decisions | Patterns Established | Gotchas Discovered |
|-------|--------------|---------------------|-------------------|
| STORY-184/185 | `TpktHeader`/`CotpHeader` frozen structs; `parse_tpkt_header`/`parse_cotp_header` pure free fns | SS-20 is genuinely stateless and protocol-agnostic | `parse_cotp_header`'s `protocol_id` extraction is a total identity mapping — this story's frame-walk loop must not add any `0x32`/`0x72` interpretation; that belongs entirely to STORY-187's `S7commAnalyzer::on_data` dispatch extension |

This story is the S7comm/ISO-on-TCP analogue of IEC-104's STORY-172 (carry buffers +
frame-walk loop), but positioned *earlier* in the sequence relative to classification
work (IEC-104's STORY-172 came after its classification stories) because the SS-20/SS-21
split means TPKT/COTP frame extraction is architecturally prior to any S7comm-specific
dispatch. `S7commAnalyzer::on_data` is therefore built incrementally: this story proves
extraction/carry/resync; STORY-187 adds the four-way `protocol_id` dispatch; STORY-188/189
add classification; STORY-190 completes the dispatch table; STORY-191/192 add MITRE
emission.

## Architecture Compliance Rules

Extracted from `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md`:
- **ADR-014 Decision 1**: SS-20 (`iso_on_tcp.rs`) is deliberately stateless. The
  directional carry buffers required for TPKT reassembly are fields on `S7commFlowState`
  (SS-21), never on a hypothetical `IsoOnTcpFlowState`.
- **ADR-014 Decision 2**: No `DispatchTarget::IsoOnTcp` variant is introduced at any
  point — SS-20 is a parsing library consumed by `S7commAnalyzer`, not an independent
  dispatch target.
- **ADR-014 Decision 8 (WALK-FIRST-RESIDUAL-BOUND)**: `MAX_S7_ISO_ON_TCP_CARRY_BYTES =
  65,535` is derived from the TPKT `length` field's own maximum (`u16::MAX`), not COTP's
  254-byte LI maximum. The frame-walk loop runs unconditionally on carry + incoming data;
  the byte bound applies only to the leftover partial-frame residual. No aggregate
  carry-plus-delivery pre-check may exist anywhere (anti-evasion, mirrors IEC-104
  F-172-001 / DNP3 F-B-002).
- Resync anchor is the TPKT version byte (`0x03`); advance exactly 1 byte per iteration,
  never 2, on a bad-version-byte or post-overflow condition.
- Pure/effectful boundary: `parse_tpkt_header`/`parse_cotp_header` remain pure; `on_data`
  and `on_flow_close` are the effectful shell.

## Library & Framework Requirements

| Tool | Version | Purpose |
|------|---------|---------|
| Rust stdlib | 1.91+ (2024 edition) | `Vec<u8>`, `HashMap`, bounds arithmetic |
| proptest | 1 (pinned in `Cargo.toml`) | VP-050 direction-isolation and walk-first-equivalence skeletons |

No new external crate dependencies.

## File Structure Requirements

| File | Action | Purpose |
|------|--------|---------|
| `src/analyzer/s7comm.rs` | CREATE | `S7commAnalyzer`, minimal `S7commFlowState` (carry fields only), `MAX_S7_ISO_ON_TCP_CARRY_BYTES`, `on_data` frame-walk loop, `on_flow_close` |
| `src/analyzer/mod.rs` | MODIFY | Add `pub mod s7comm;` (module created but not yet registered with the dispatcher — that is STORY-193) |
| `tests/s7comm_analyzer_tests.rs` | CREATE | Unit tests for BC-2.20.013-016 + VP-050 proptest skeletons + static regression-guard tests |

## Forbidden Dependencies

- `rusty-cotp`, `rusty-tpkt`, `tpkt`, `copt`, `s7`, `s7-comm`, `s7-client`, Wireshark,
  Snap7, libnodave source — banned/avoid per ADR-014 Decision 4
- `src/analyzer/s7comm.rs` MUST NOT define a type named `IsoOnTcpFlowState` (BC-2.20.016
  postcondition 3) — carry buffers belong on `S7commFlowState` exclusively
- `carry_c2s` and `carry_s2c` MUST NOT be merged into a single shared `Vec<u8>` — this
  would violate RULING-DNP3-SIBLING-001's directional-isolation requirement

## Changelog

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.2 | 2026-09-24 | story-writer | STORY-187 spec pass; consistency audit finding #2; BC-2.20.014 v1.2. AC-186-004 was corrected: it previously claimed the exactly-65,535-byte at-bound residual case was "LIVE, reachable via real `on_data` traffic," but the existing test (`test_BC_2_20_014_at_bound_residual_no_overflow`) actually seeds `S7commFlowState.carry_c2s` directly via `max_length_frame()` injection, then calls `on_data` with an empty delivery — this is SYNTHETIC field injection, not `on_data`-reachable, exactly like AC-186-005/006. Under the walk-first design, a residual of exactly 65,535 bytes is UNREALIZABLE via `on_data`: it would itself be a complete, dispatchable frame and would be extracted, not stashed to carry (BC-2.20.014 v1.2 Invariant 1). AC-186-004 is split into two explicit parts: (a) the existing test, relabeled SYNTHETIC strict-`>` boundary check (same labeling convention as AC-186-005/006) — no test code change; (b) a NEW LIVE near-bound requirement (not yet implemented) asserting that feeding a declared `length=65,535` TPKT frame minus its final byte (65,534 bytes, possibly across multiple segments/calls) via real `on_data` traffic leaves carry holding exactly 65,534 bytes with no finding and an unset overflow dedup flag, and that delivering the final byte then extracts the frame and empties carry. Status remains `delivered`; this is a spec-precision correction only — no behavioral change to shipped code. A follow-up fix PR is required to (i) add the new AC-186-004(b) `test_BC_2_20_014_live_near_bound_residual_reachable` test, and (ii) relabel/re-comment the existing `test_BC_2_20_014_at_bound_residual_no_overflow` test and any demo evidence as SYNTHETIC — those src/test/docs changes are explicitly NOT made by this story-file amendment. **Same-pass follow-up (coordinator-requested scan for residual-65,535-reachable claims):** the Edge Cases table carried the identical defect twice over — EC-003 stated `residual.len() == 65,535` exactly as "at bound, legitimate" with no synthetic label, and EC-004 stated `residual.len() == 65,536` as "one over bound, adversarial" with no synthetic label; both are UNREALIZABLE via real `on_data` traffic for the same reason as AC-186-004(a). EC-003 is now explicitly labeled SYNTHETIC direct field injection; a NEW EC-004 was inserted for the actual `on_data`-reachable maximum (`residual.len() == 65,534` exactly, mirroring AC-186-004(b)); the former EC-004 (65,536 case) is renumbered EC-005 and now explicitly labeled SYNTHETIC; EC-005/EC-006 (0x03-invalid-length, MMS/ICCP reuse) are renumbered EC-006/EC-007 with no content change. The VP-050 proptest skeleton's inline comment (`// each carry stays <= MAX_S7_ISO_ON_TCP_CARRY_BYTES (65,535)`) was also tightened to the precise reachable bound, `<= 65,534` bytes, with the same Invariant-1 citation. No other residual-65,535-reachable claims were found elsewhere in this story (Tasks and Architecture Compliance Rules reference the `65,535` guard constant only as the defense-in-depth bound derivation, not as a reachability claim — left unchanged). **Further same-pass follow-up (BC-summary table H1 refresh):** the Behavioral Contracts table's BC-2.20.014 row still carried its pre-v1.1 H1 ("Carry Buffer Bounded at MAX_S7_ISO_ON_TCP_CARRY_BYTES=65,535; Overflow Triggers Clear-and-Resync With One T0814 Per Direction") instead of the current H1; refreshed verbatim to "Carry-Overflow Bound (`MAX_S7_ISO_ON_TCP_CARRY_BYTES = 65,535`) and T0814 Guard — Defense-in-Depth, Unreachable by Construction Under Walk-First Design". The BC-2.20.013 row was checked against its current H1 and also did not match verbatim ("TPKT Frames Spanning TCP Segments Reassembled via Directional Carry Buffers, Walk-First Residual-Bound Semantics" vs. the source H1's "TPKT Frames Spanning TCP Segment Boundaries Are Reassembled via Directional Carry Buffers Using Walk-First, Residual-Bound Semantics") — corrected verbatim as well. Both refreshed per `bc_h1_is_title_source_of_truth`. **Third same-pass follow-up (product-owner's final BC-2.20.014 v1.2 Edge Cases re-map):** product-owner finalized BC-2.20.014's Edge Cases table as EC-001 = live single-call 65,534 residual, EC-002 = live multi-call progressive accumulation to 65,534 then final-byte completion empties carry, EC-003/004/005 unchanged (counterfactual/synthetic), EC-006 = NEW SYNTHETIC literal-65,535-boundary via direct field injection. AC-186-004's trace line was split accordingly: part (a) (SYNTHETIC literal-65,535 check) now traces to BC-2.20.014 edge case EC-006 (previously cited the now-repurposed EC-001); part (b) (LIVE near-bound check) now traces to edge case EC-001 for the single-call/split-delivery case and edge case EC-002 specifically for the final-byte-completion step, with inline citations added at each corresponding Given/When bullet. AC-186-005's EC-004 citation (second overflow event, same-direction non-re-emission) and AC-186-006's EC-005 citation (independent per-direction dedup) were checked against the new map and require no change — both edge cases are unchanged under the product-owner's re-map. AC-186-005's "NEW positive assertion" paragraph was also updated: its stale "BC-2.20.014 v1.1 Invariant 1 / VP-050 reachability property" citation is now "BC-2.20.014 v1.2 Invariant 1 / VP-050 clause (c) REACHABLE-BOUND INVARIANT, VP-INDEX.md v2.49" to match VP-050's rescoped registered text; all other VP-050 references in this story (the VP-050 Proptest Obligation section header, Library table, File Structure table) were checked and are generic/non-claim-bearing, requiring no change. `behavioral_contracts:`, file list, and input-hash are unchanged. |
| 1.1 | 2026-09-07 | story-writer | Adversarial-review spec reconciliation (BC-2.20.013/014 v1.1, STORY-186 gate F-02/F-03/F-04, human ruling Option B — Defense-in-Depth): reframed AC-186-004/005/006 — the carry-overflow bound + T0814 emission is now specified as a defense-in-depth guard, unreachable-by-construction via `on_data` under the walk-first (BC-2.20.013) + 1-byte-resync (BC-2.20.015) design given the u16 TPKT length cap (carry provably `≤ 65,534`); AC-186-004 retained as the guard's live-reachable at-bound comparison-boundary case; AC-186-005/006 reframed as SYNTHETIC direct-flow-state-injection guard-mechanics tests (test-fn names unchanged); added new AC-186-005 positive assertion + test `test_BC_2_20_014_overflow_unreachable_via_on_data` asserting a real `on_data` garbage flood emits no T0814 and keeps carry `≤ 65,534`. Fixed F-04 story-text defect: AC-186-007's inline byte example corrected from the invalid `[0x01,0x03,0x00,0x00,0x04]` (length=4, rejected by BC-2.20.003's `length >= 7` floor) to BC-2.20.015's canonical `[0x01,0x03,0x00,0x00,0x07]`; corrected Edge Case EC-005's length floor from `< 4` to `< 7`. No change to `behavioral_contracts:`, file list, or guard IF-reached mechanics (clear-not-truncate, one T0814/direction, per-direction dedup — AC-186-005's call-entry framing stands). |
| 1.0 | 2026-09-06 | story-writer | Initial authorship — `s7comm.rs` created, carry-buffer reassembly, walk-first frame extraction, resync, frozen SS-20/SS-21 boundary regression guards, VP-050 skeleton, AC-186-001..012. |
