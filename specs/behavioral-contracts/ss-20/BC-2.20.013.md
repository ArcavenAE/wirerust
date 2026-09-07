---
document_type: behavioral-contract
level: L3
version: "1.1"
status: draft
producer: product-owner
timestamp: 2026-09-06T00:00:00Z
phase: f2
origin: greenfield
extracted_from: null
traces_to: .factory/specs/domain/domain-spec.md
subsystem: SS-20
capability: CAP-20
lifecycle_status: active
introduced: feature-s7comm
modified:
  - version: "1.1"
    date: 2026-09-07
    change: "Added Reconciliation Note documenting the walk-first-bounds-carry relationship (STORY-186 adversarial gate F-02/F-03, two independent passes): the frame-walk loop's residual-only stash, combined with the u16-capped TPKT `length` field, bounds the directional carry at ≤65,534 bytes by construction. Cross-references BC-2.20.014 v1.1's defense-in-depth reclassification of the carry-overflow/T0814 guard. No change to this BC's own preconditions, postconditions, or invariants — the walk-first design (Invariant 1, anti-evasion rationale) is NOT reopened; this is documentation-only addition per human ruling (Option B — Defense-in-Depth)."
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

# BC-2.20.013: TPKT Frames Spanning TCP Segment Boundaries Are Reassembled via Directional Carry Buffers Using Walk-First, Residual-Bound Semantics

## Description

A TPKT frame's declared `length` (BC-2.20.004) can exceed the bytes delivered in a
single TCP segment, so `S7commAnalyzer::on_data` must reassemble frames across
multiple `on_data` calls. Per ADR-014 Decision 8, this reuses the directional
carry-buffer split established for DNP3 (ADR-007) and IEC-104 (ADR-013):
`carry_c2s: Vec<u8>` and `carry_s2c: Vec<u8>` fields on `S7commFlowState`. The
frame-walk loop runs **unconditionally** on `carry ++ incoming_data`, extracting every
complete TPKT frame (`parse_tpkt_header` succeeds and `length` bytes are available)
before any byte-count bound is applied — the byte bound (BC-2.20.014) is applied only
to the leftover partial-frame residual stashed back into carry. This is the
WALK-FIRST-RESIDUAL-BOUND discipline (ADR-013 Decision 2/3), explicitly *not* an
aggregate carry-plus-delivery pre-check.

## Preconditions

1. `S7commFlowState` exists for the flow (created on first `on_data` call for a
   port-102-classified flow).
2. `on_data` is called with a direction (`c2s` or `s2c`) and new bytes `incoming_data`.
3. `working = carry[direction] ++ incoming_data` (concatenation, not a separate
   pre-check on the combined length).

## Postconditions

1. The frame-walk loop repeatedly calls `parse_tpkt_header(&working[cursor..])`:
   - If `Some(header)` and `working.len() - cursor >= header.length as usize`: a
     complete TPKT frame is extracted; `parse_cotp_header` is invoked on its COTP
     payload; `cursor += header.length as usize`; the loop continues.
   - If `Some(header)` but `working.len() - cursor < header.length as usize`: the frame
     is declared-but-incomplete; the loop breaks, and `working[cursor..]` (the entire
     partial frame, including its already-parsed TPKT header) is stashed to
     `carry[direction]`.
   - If `None` (BC-2.20.001/002/003 reject paths): the loop breaks and
     `working[cursor..]` is stashed to `carry[direction]` unchanged, OR — if the reject
     reason is a bad version byte — the resync walk (BC-2.20.015) advances 1 byte and
     retries before giving up and stashing.
2. **No aggregate pre-check**: at no point does the implementation compare
   `carry[direction].len() + incoming_data.len()` against any bound before running the
   frame-walk loop. The walk always runs first.
3. After the loop, `carry[direction]` holds only the trailing partial-frame residual —
   never a complete frame that was available to extract.

## Invariants

1. **Anti-evasion rationale (Ptacek/Newsham-class evasion channel, rejected
   alternative)**: an aggregate carry-plus-delivery pre-check would let an attacker pad
   a burst to push the total over a bound, causing the monitor to drop an
   already-complete malicious frame at the head while the endpoint (which reassembles
   at the TCP layer) processes it normally. wirerust has already closed this exact hole
   twice (IEC-104 F-172-001, DNP3 F-B-002); ADR-014 Decision 8 explicitly reuses the
   fix rather than reopening it for S7comm.
2. **Progress guarantee**: each frame-walk iteration either extracts a complete frame
   (advancing `cursor` by at least 4 bytes — the TPKT minimum) or terminates the loop
   (incomplete frame or unparseable start) — the loop cannot spin without making
   progress or breaking.
3. **Directional isolation**: `carry_c2s` and `carry_s2c` are independent; a
   partial frame in one direction never affects parsing in the other.

## Reconciliation Note (STORY-186 Adversarial Gate F-02/F-03, added 2026-09-07)

**Walk-first bounds the directional carry ≤65,534 bytes by construction; this is the
basis for BC-2.20.014 v1.1's defense-in-depth reclassification.**

STORY-186's per-story adversarial gate (confirmed by two independent passes) found that
BC-2.20.014's carry-overflow bound (`residual.len() > 65,535` → clear-and-resync + one
T0814) is unreachable via the real `on_data` data path, because:

1. **Postcondition 3 above** establishes that after the frame-walk loop runs, `carry[direction]`
   holds only the trailing partial-frame residual — never a complete frame that was
   available to extract. The largest possible such residual is a declared-but-incomplete
   frame carrying a TPKT header whose `length` field is at its maximum representable
   value, `65,535` (`u16::MAX`, BC-2.20.004), with at least 1 byte of that declared length
   still missing (otherwise the frame would have been extracted, not stashed as
   incomplete). The residual is therefore bounded at `≤ 65,534` bytes — one byte short of
   BC-2.20.014's `> 65,535` overflow threshold — for every legitimate (declared-but-incomplete)
   residual, on every `on_data` call.
2. **Postcondition 1's `None` branch**, combined with BC-2.20.015's resync sub-routine, means
   un-anchored garbage (bytes that never form a parseable `0x03`-anchored TPKT header) is
   drained 1 byte at a time within the *same* `on_data` call until either a valid frame
   candidate is found or fewer than 4 bytes remain (BC-2.20.015 Postcondition 3(b)). Garbage
   is never carried forward whole from one call to the next and re-appended to on top of
   more incoming garbage without an intervening resync attempt — so it cannot accumulate
   across calls the way BC-2.20.014's original EC-003 premise assumed.
3. Because TPKT `length` is u16-capped (`≤ 65,535`, RFC 1006 §6) and both the
   legitimate-residual path (bound 1) and the adversarial-garbage path (bound 2) are
   independently capped well below the `65,535` overflow threshold, the directional carry
   is bounded at `≤ 65,534` bytes **by construction** — for both conformant and
   adversarial input, on every `on_data` call, without exception.

**Consequence:** BC-2.20.014's `residual.len() > 65,535` precondition can never be
satisfied by any sequence of real `on_data` calls under this BC's walk-first design (or
BC-2.20.015's resync design). Per human ruling (senior architect, 2026-09-07, Option
B — Defense-in-Depth), BC-2.20.014's overflow bound and T0814 emission are RECLASSIFIED
(BC-2.20.014 v1.1) as a defense-in-depth guard — structurally unreachable today, retained
as a safety net against a future design regression in this BC or BC-2.20.015, not as an
assertion that the guard is a live, exercisable runtime detection.

**This walk-first design itself is NOT reopened by this finding or this reconciliation
note.** The anti-evasion rationale above (Invariant 1) — rejecting an aggregate
carry-plus-delivery pre-check in favor of always running the walk first — remains the
settled ruling, mirroring IEC-104 F-172-001 and DNP3 F-B-002. The adversarial finding
concerned only BC-2.20.014's reachability claim, not this BC's reassembly discipline.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | A single `on_data` call delivers exactly one complete TPKT frame, no carry before or after | Frame extracted; `carry[direction]` remains empty |
| EC-002 | A TPKT frame's first 4 bytes (header) arrive in one segment, the remaining `length-4` payload bytes arrive in the next segment | First `on_data` call: `parse_tpkt_header` succeeds but insufficient trailing bytes — declared-but-incomplete; entire header+partial-payload stashed to carry. Second `on_data` call: `carry ++ new_bytes` now contains the complete frame; extracted normally |
| EC-003 | A TCP segment delivers two complete TPKT frames back-to-back plus a partial third frame | Both complete frames are extracted in the same `on_data` call; only the partial third frame's bytes are stashed to carry |
| EC-004 | The carry buffer from a previous call already holds a partial frame, and the new segment completes it AND starts a second complete frame | Both frames extracted in one call; loop continues past the first frame's end without requiring a fresh `on_data` call |
| EC-005 | An attacker sends a burst designed to push `carry.len() + incoming.len()` over any conceivable pre-check bound while a complete, small malicious frame sits at the head | The walk-first design extracts the head frame regardless — this is precisely the evasion this design prevents (contrast with the rejected aggregate pre-check alternative) |

## Canonical Test Vectors

| Scenario | Input sequence | Expected Behavior | Category |
|----------|----------------|-------------------|---------|
| Split-frame across carry/delivery | Call 1: `[0x03,0x00,0x00,0x0A]` (4-byte TPKT header declaring length=10); Call 2: `[remaining 6 bytes]` | Call 1: `carry_c2s` = 4 bytes (header only, incomplete). Call 2: `working` = 10 bytes total; frame extracted; `carry_c2s` empty after | legit: split frame |
| Single delivery, complete-frame-plus-tail | One `on_data` call: `[complete 10-byte frame][3 header-only bytes of a second frame]` | First frame extracted; 3-byte tail stashed to carry | legit: complete + partial tail |
| Adversarial burst, complete frame at head | One `on_data` call: `[complete 7-byte CR frame][60,000 bytes of garbage designed to look large]` | The 7-byte CR frame is extracted regardless of the trailing garbage's size; the garbage bytes are handled independently by subsequent walk iterations (malformed/resync per BC-2.20.015), never causing the head frame to be dropped | non-conformant: evasion-resistance regression guard |

## Verification Properties

| Property | Proof Method (planned) |
|----------|-------------------------|
| The frame-walk loop makes progress every iteration (advances `cursor` or terminates); no infinite loop for any finite input | proptest P1 (directional carry-buffer isolation per ADR-014 Decision 9) — VP-NNN allocation deferred to the F2 INTEGRATE sub-burst |
| No aggregate pre-check exists: for any split of a byte sequence into `carry` + `incoming`, the walk-first result is identical to running the walk once on the concatenated bytes | proptest P1 — VP-NNN allocation deferred |

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-20 ("ISO-on-TCP Framing (TPKT/COTP)") per domain/capabilities/cap-20-iso-on-tcp-framing.md §CAP-20 |
| Capability Anchor Justification | CAP-20 ("ISO-on-TCP Framing (TPKT/COTP)") per domain/capabilities/cap-20-iso-on-tcp-framing.md §CAP-20 — TPKT frame reassembly across TCP segments is a core framing responsibility of the ISO-on-TCP layer, directly named in the F2 authoring scope |
| L2 Domain Invariants | INV-2 (Content-First Dispatch Precedence); reassembly ordering invariant (in-order TCP delivery assumed per SS-04 reassembly contract) |
| Architecture Module | SS-20/SS-21 boundary (`src/analyzer/iso_on_tcp.rs` pure parse fns consumed by `S7commAnalyzer::on_data`, SS-21); `S7commFlowState.carry_c2s`/`carry_s2c` |
| ADR | ADR-014 Decision 8 (WALK-FIRST-RESIDUAL-BOUND, reusing ADR-013 Decision 2/3 and RULING-DNP3-SIBLING-001 / ADR-007 Decision 2) |
| Stories | (TBD — story-writer assigns in F3) |
| Feature | feature-s7comm |
| MITRE Techniques | (none directly for the reassembly mechanism itself; carry-overflow reaction is BC-2.20.014) |

## Related BCs

- BC-2.20.001..003 — depends on (the `None` reject paths whose bytes get stashed to carry)
- BC-2.20.004 — depends on (the accept path whose `length` field drives the completeness check)
- BC-2.20.014 — composes with (the residual-only byte bound applied to whatever this BC leaves in carry; see Reconciliation Note above — this BC's walk-first residual bound is one of the two premises proving BC-2.20.014's overflow guard unreachable-by-construction, v1.1)
- BC-2.20.015 — composes with (resync anchor reused when a bad-version-byte reject occurs mid-walk)

## Architecture Anchors

- `src/analyzer/iso_on_tcp.rs` (planned) — pure `parse_tpkt_header`/`parse_cotp_header` consumed by the frame-walk loop
- `S7commFlowState.carry_c2s: Vec<u8>` / `S7commFlowState.carry_s2c: Vec<u8>` (planned, SS-21) — directional carry buffers, per ADR-014 Decision 8
- `S7commAnalyzer::on_data` (planned, SS-21) — frame-walk loop implementation
- `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md §Decision 8` — full WALK-FIRST-RESIDUAL-BOUND rationale and anti-evasion argument

## Story Anchor

STORY-186 (also a formal-hardening re-verification anchor for STORY-194)

## VP Anchors

- VP-050 (proptest P1) — TPKT/COTP Carry-Buffer Residual-Bound Reassembly, Overflow
  Isolation, and 1-Byte Resync; registered F2 INTEGRATE sub-burst per VP-INDEX.md
  v2.48; traces BC-2.20.013..015
- VP-055 (cargo-fuzz P1) — S7comm/ISO-on-TCP combined parse-chain no-panic fuzz
  (`fuzz_s7comm_parser`); registered representative-subset source_bc includes this BC

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | per-flow mutable state (`S7commFlowState.carry_c2s`/`carry_s2c`) — the frame-walk loop itself is stateful at the flow level, though it is built from the pure `parse_tpkt_header`/`parse_cotp_header` primitives |
| **Deterministic** | yes — given the same sequence of `on_data` calls and byte contents |
| **Thread safety** | flow state is per-flow, not shared across threads within a single flow's processing |
| **Overall classification** | stateful orchestration built on a pure-core parsing kernel |
