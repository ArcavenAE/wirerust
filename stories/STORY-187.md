---
document_type: story
level: ops
story_id: STORY-187
title: "S7comm Flow State Completion, Four-Way protocol_id Dispatch Skeleton, and parse_s7comm_header Pure-Core Parser"
epic_id: E-23
version: "1.17"
status: ready
producer: story-writer
timestamp: 2026-09-24T00:00:00Z
phase: f3
traces_to: .factory/specs/prd.md
points: 8
priority: P1
cycle: feature-s7comm
wave: 90
target_module: analyzer/s7comm
subsystems: [SS-21]
estimated_days: null
assumption_validations: []
risk_mitigations: []
tdd_mode: strict
feature_id: feature-s7comm
depends_on: [STORY-186]
blocks: [STORY-188]
behavioral_contracts: [BC-2.21.001, BC-2.21.002, BC-2.21.004, BC-2.21.005, BC-2.21.006, BC-2.21.007, BC-2.21.008, BC-2.21.009]
verification_properties: [VP-051, VP-053]
inputs:
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.001.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.002.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.004.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.005.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.006.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.007.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.008.md
  - .factory/specs/behavioral-contracts/ss-21/BC-2.21.009.md
  - .factory/specs/architecture/ARCH-INDEX.md
  - docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md
input-hash: "315fdf1"
---

> **tdd_mode:** `strict` — full TDD Iron Law enforced.

# STORY-187: S7comm Flow State Completion, Four-Way protocol_id Dispatch Skeleton, and parse_s7comm_header Pure-Core Parser

## Narrative

**As a** security analyst using wirerust to inspect classic S7comm traffic,
**I want** `S7commFlowState` completed with classification state, `S7commAnalyzer` to
branch on the extracted COTP `protocol_id` byte, and a bounds-safe pure-core parser for
the classic S7comm (`0x32`) common header,
**so that** classic S7comm PDUs (ROSCTR, PDU reference, parameter/data length) are
correctly and safely parsed as the foundation for function-code classification
(STORY-188/189) and MITRE technique emission (STORY-191/192).

This story completes the `S7commFlowState` struct definition (fields not yet needed by
STORY-186's frame-extraction-only scope) and extends `S7commAnalyzer::on_data` with the
four-way dispatch on `CotpHeader::protocol_id`. The `Some(0x32)` (classic) branch is
fully wired to `parse_s7comm_header`; the `Some(0x72)` (S7comm-plus) and
unclassified/unrecognized branches are completed structurally in STORY-190 — this story
routes them to a placeholder no-op so the four-way match is total and compiles, without
yet implementing their observable behavior.

## Behavioral Contracts

| BC ID | Title | Story Role |
|-------|-------|-----------|
| BC-2.21.001 | `S7commFlowState` Owns TPKT/COTP Carry Buffers, S7comm Classification State, and Per-Direction Dedup Flags | Completes the flow-state struct started in STORY-186 |
| BC-2.21.002 | `S7commAnalyzer::on_data` Four-Way Dispatch on `CotpHeader::protocol_id` | Dispatch skeleton (classic branch fully wired; plus/unclassified branches placeholder) |
| BC-2.21.004 | `parse_s7comm_header` Returns None for Input Shorter Than 10 Bytes | Reject path: length < 10 |
| BC-2.21.005 | `parse_s7comm_header` Defensively Rejects `data[0] != 0x32` | Defense-in-depth reject path |
| BC-2.21.006 | `parse_s7comm_header` Extracts ROSCTR, PDU Reference, Parameter Length, and Data Length from a Valid 10-byte Common Header (Happy Path) | Accept path for Job/Userdata ROSCTR only — Ack_Data moved to BC-2.21.008's 12-byte header per the 2026-09-24 canonical-frame holdout ruling (DF-CANONICAL-FRAME-HOLDOUT-001) |
| BC-2.21.007 | `parse_s7comm_header` Returns None for an Unrecognized ROSCTR Byte (Safe-Reject, No Force-Fit) | Safe-reject, no force-fit |
| BC-2.21.008 | `parse_s7comm_header` for ROSCTR=Ack (0x02) and Ack_Data (0x03) Requires 12 Bytes (Error Class + Error Code) | Ack/Ack_Data-specific 12-byte header extension (both ROSCTR values require the same header shape per the 2026-09-24 canonical-frame holdout ruling) |
| BC-2.21.009 | Declared `param_length`/`data_length` Are Bounds-Checked Against Remaining Bytes Before Parameter/Data Block Access (Safe-Reject on Inconsistency) | Safe-reject on inconsistency |

## Acceptance Criteria

### AC-187-001: `S7commFlowState` carries the full field set required by this story's scope
(traces to BC-2.21.001 postcondition 1)
- Given `S7commFlowState` after this story
- When its fields are inspected
- Then, in addition to the carry fields from STORY-186, it now also carries:
  `session_established: bool`, `cr_observed_dir: Option<Direction>` (or an
  equivalently-purposed field — exact name not mandated, per BC-2.21.001 Postcondition 1
  "at minimum" field-set permission) hosting the pending-CR-direction tracking state
  that makes the F-01 opposite-direction-CC matching rule (AC-187-003) testable,
  `classified_protocol: Option<S7Protocol>`, `malformed_header_reported_c2s: bool`,
  `malformed_header_reported_s2c: bool`
- No field on `S7commFlowState` duplicates a field SS-20 owns (traces to BC-2.21.001
  postcondition 2)
- **Test:** `test_BC_2_21_001_flow_state_field_set`

### AC-187-002: `S7commFlowState` is created lazily on first `on_data` call
(traces to BC-2.21.001 postcondition 3)
- Given a newly classified flow with no prior `on_data` call
- When `on_data` is called for the first time
- Then `S7commFlowState` is created and stored in the analyzer's per-flow map, keyed by
  `FlowKey`
- **Test:** `test_BC_2_21_001_lazy_flow_state_creation`

### AC-187-003: session_established is set only by a CC observed opposite-direction from a prior CR; CR-only, CC-only, CC-before-CR, same-direction CC, and repeated same-direction CR do not set it
(traces to BC-2.21.001 postcondition 1; BC-2.21.002 postcondition 2)
- Given a CR TPDU observed in direction A, followed by a CC TPDU observed in direction B
  where B is opposite A, on the same flow
- When `on_data` dispatches both frames in order
- Then `S7commFlowState.session_established` becomes `true`; no protocol classification
  occurs — classification remains deferred to the first DT frame regardless of this
  flag's value (F-01 ruling)
- **Test:** `test_BC_2_21_001_cr_then_opposite_cc_sets_session_established`
- Given any of the following five negative sequences, on a flow with no other CR/CC
  activity: (a) a CR observed, with no CC ever following (CR-only — this sub-case is not
  independently enumerated as a distinct BC-2.21.001 EC-NNN; it is the trivial
  no-matching-CC-yet state implied by Postcondition 1); (b) the flow's first observed
  COTP frame is a CC with no prior CR (CC-only, e.g. mid-flow capture start,
  BC-2.21.001 EC-004); (c) a CC observed before any CR (out-of-order, BC-2.21.001
  EC-005); (d) a CR in direction A followed by a CC also in direction A (same-direction,
  BC-2.21.001 EC-006); (e) a CR observed in direction A, followed by a second CR also
  observed in direction A with no intervening CC (repeated same-direction CR,
  BC-2.21.001 EC-007)
- When `on_data` dispatches each sequence
- Then `S7commFlowState.session_established` remains `false` in every one of the five
  cases — a CC with no preceding opposite-direction CR never "matches" (F-01 ruling); for
  case (e) specifically, `cr_observed_dir` (or equivalent) is asserted to still equal
  `Some(A)` after the second CR — the repeated CR does not clear or corrupt the pending
  direction, and does not itself set `session_established` (BC-2.21.001 v1.4+ EC-007's
  deterministic most-recent-CR-wins overwrite rule: `cr_observed_dir` is unconditionally
  overwritten by every CR, not merely retained-if-absent — here the overwrite is
  unobservable only because both CRs share direction A, not because either choice was
  permitted)
- **Tests:** `test_BC_2_21_001_cr_only_session_not_established` (case (a) only — asserts
  `session_established == false`; this test does NOT claim BC-2.21.001 EC-007, which is
  covered exclusively by the new repeated-CR test below),
  `test_BC_2_21_001_cc_only_no_prior_cr_session_not_established` (case (b), EC-004),
  `test_BC_2_21_001_cc_before_cr_session_not_established` (case (c), EC-005),
  `test_BC_2_21_001_same_direction_cc_session_not_established` (case (d), EC-006),
  `test_BC_2_21_001_repeated_same_direction_cr_session_not_established` (case (e),
  EC-007 — asserts both `session_established == false` and
  `cr_observed_dir == Some(A)`)
- **Most recent CR direction wins (pass-11 addition):** the pending `cr_observed_dir` is
  overwritten by every CR observed, not only the first — this is exercised across two CRs
  of *different* directions (EC-007 itself only exercises two CRs of the *same*
  direction). Given `CR(A)`, then `CR(B)` (A != B), with no intervening CC, on the same
  flow: (i) a subsequent `CC(A)` leaves `session_established == true` — because the CC's
  direction (A) is opposite the *most recent* CR's direction (B), not the stale first CR
  (A); (ii) a subsequent `CC(B)` (a fresh flow with the same `CR(A)`, `CR(B)` prefix)
  leaves `session_established == false` — the CC's direction (B) matches the most recent
  CR's direction (B), so it is a same-direction CC, not opposite-direction
- **Test:** `test_BC_2_21_001_most_recent_cr_direction_wins` (covers both the `CC(A)` ->
  `established == true` and `CC(B)` -> `established == false` sub-cases against the same
  `CR(A)`, `CR(B)` prefix; traces to BC-2.21.001 EC-008 for the `CC(A)` ->
  `established == true` sub-case and EC-009 for the `CC(B)` -> `established == false`
  sub-case)
- **session_established is monotonic (pass-13/14 addition, P14-F-3):** once
  `session_established` becomes `true`, no subsequent frame on the same flow — a
  same-direction CC, a further CR in either direction, or any other COTP/DT traffic —
  ever reverts it to `false`; the flag is a one-way latch driven by the opposite-direction
  CC match, not a condition re-evaluated on every frame (traces to BC-2.21.001
  postcondition 1). **N-4 (exact sequences, confirmed via grep of
  `tests/s7comm_analyzer_tests.rs`):** the test exercises exactly two sequences, each on
  a fresh flow: (a) `CR(c2s)`, `CC(s2c)` [establishes], `CC(c2s)` [same-direction as the
  still-recorded CR — must not clear the flag]; (b) `CR(c2s)`, `CC(s2c)` [establishes],
  `CR(s2c)` [overwrites `cr_observed_dir` but must not clear `session_established`].
  **P23-F-1 addition (confirmed via grep):** the test now also directly asserts
  `cr_observed_dir` at each step, not merely `session_established` — after sequence (a)'s
  establishing CC, `cr_observed_dir == Some(ClientToServer)` is unchanged (the matching
  `ConnectConfirm` arm only ever reads `cr_observed_dir`, it never clears it, even on the
  establishing CC); after sequence (b)'s further `CR(s2c)`, `cr_observed_dir ==
  Some(ServerToClient)` (the `ConnectRequest` arm unconditionally overwrites
  `cr_observed_dir` with every CR's direction regardless of whether
  `session_established` is already `true`) — both per BC-2.21.001 postcondition 1's
  overwritten-by-every-CR, never-cleared-on-CC rule
- **Test:** `test_BC_2_21_001_session_established_is_monotonic`

### AC-187-004: `Some(0x32)` DT frames dispatch to classic S7comm dissection when the flow's sticky classified_protocol is (or becomes) Classic
(traces to BC-2.21.002 postcondition 3)
- Given `parse_cotp_header` returns `Some(CotpHeader { tpdu_type: DataTransfer,
  protocol_id: Some(0x32), .. })` and the flow's sticky `classified_protocol` is (or, by
  this very frame's own first-classification, becomes) `Some(S7Protocol::Classic)`
- When `on_data` dispatches this frame
- Then `parse_s7comm_header` is called on the slice beginning at `payload_offset`, and
  after dispatch `S7commFlowState.classified_protocol == Some(S7Protocol::Classic)`
- Given a malformed (too-short, `data.len() < 10`) `0x32`-leading DT frame arrives on a
  flow whose sticky `classified_protocol` is already `Some(S7Protocol::Classic)`
- When `on_data` dispatches it
- Then exactly one T0814 finding is emitted for the malformed-header condition
  (BC-2.21.004 Postcondition 4) — not more than one, and not zero
- **Tests:** `test_BC_2_21_002_classic_s7comm_dispatch_asserts_classified_protocol_classic`,
  `test_BC_2_21_002_malformed_0x32_frame_on_classic_flow_emits_exactly_one_t0814`

### AC-187-005: First DT frame carrying protocol_id: Some(byte) sets classified_protocol exactly once (sticky first-classification-wins); protocol_id: None never classifies
(traces to BC-2.21.002 postcondition 6)
- Given a flow's first DT frame carries `protocol_id: Some(byte)` (`0x32`->Classic,
  `0x72`->Plus, any other byte->Unclassified)
- When `on_data` processes it
- Then `S7commFlowState.classified_protocol` is set exactly once from that frame;
  subsequent DT frames on the same flow — including ones whose `protocol_id` differs, is
  a different `Some(other)`, or is `None` — never overwrite it
- **Test:** `test_BC_2_21_002_sticky_first_classification`
- Given a flow's first DT frame carries `protocol_id: None` (empty DT payload)
- When `on_data` processes it
- Then `classified_protocol` remains `None` — a `None`-`protocol_id` DT frame carries no
  protocol evidence, is never treated as a classifying event, and does not consume the
  flow's "first DT frame" status; classification remains deferred to a later DT frame (if
  any) that carries `Some(byte)` (F-02 ruling)
- **Test:** `test_BC_2_21_002_none_protocol_id_dt_first_then_0x32_dt_classifies_classic`
  (sequence: first DT frame `protocol_id: None`, second DT frame `protocol_id:
  Some(0x32)` -> `classified_protocol == Some(S7Protocol::Classic)`; traces to
  BC-2.21.002 Edge Case EC-004)
- Given two DT frames with different `protocol_id` values arrive back-to-back within a
  single `on_data` call (multiple frames in one delivery, mirrors BC-2.20.013's
  multi-frame walk)
- When `on_data` processes the single delivery
- Then `classified_protocol`'s first-write-wins rule applies across the pair in arrival
  order — the first frame within the delivery sets it, the second does not overwrite it
  (traces to BC-2.21.002 Edge Case EC-003)
- **Test:** `test_BC_2_21_002_two_frames_one_delivery_first_write_wins`
- Given a single `on_data` delivery carries an empty DT frame (`protocol_id: None`)
  immediately followed, within that same delivery, by a second frame that also carries no
  classifying evidence (e.g. a second `protocol_id: None` DT, or a non-DT session TPDU)
- When `on_data` processes the whole delivery
- Then `classified_protocol` remains `None` after the entire delivery is processed — a
  delivery containing zero classifying frames leaves the flow's sticky classification
  untouched regardless of how many non-classifying frames it contains (pass-11 addition,
  complements the None-then-`Some(0x32)` and two-frames-first-write-wins tests above,
  which both include at least one classifying frame)
- **Test:** `test_BC_2_21_002_empty_dt_followed_by_frame_same_delivery_stays_unclassified`
- Given a COTP frame that `parse_cotp_header` cannot parse at all (e.g. an unrecognized
  TPDU-type high nibble) arrives on a flow with no prior classification
- When `on_data` processes it
- Then `classified_protocol` remains `None`, `session_established` remains `false`, and
  `cr_observed_dir` remains `None` — an unparseable COTP frame carries no classifying
  evidence of any kind and must never classify the flow, set the session flag, or
  populate the pending-CR direction (pass-13/14 addition, P14-F-2; traces to BC-2.21.002
  postcondition 1)
- **Test:** `test_BC_2_21_002_unparseable_cotp_does_not_classify`

### AC-187-012: Classic S7comm dissection is gated on the flow's sticky classified_protocol == Classic, never on the current frame's raw protocol_id byte alone
(traces to BC-2.21.002 postcondition 3; invariant 4)
- Given a flow whose sticky `classified_protocol` is already `Some(S7Protocol::Plus)` or
  `Some(S7Protocol::Unclassified)` (set by an earlier DT frame per AC-187-005)
- When a later DT frame on the same flow carries `protocol_id: Some(0x32)`
- Then `parse_s7comm_header` is NOT called for this frame; no dissection of any kind is
  attempted; no `Finding` is emitted for it — per ADR-014 Decision 2's no-misattribution
  guarantee applied at the flow level (F-12 ruling; traces to BC-2.21.002 Edge Case
  EC-005)
- **Test:** `test_BC_2_21_002_0x32_dt_frame_not_dissected_when_sticky_classified_plus_or_unclassified`
- **The gate is a conjunction, not either condition alone (pass-13/14 addition):**
  classic dissection requires BOTH the current frame's `protocol_id == Some(0x32)` AND
  the flow's sticky `classified_protocol == Some(Classic)` — neither condition alone is
  sufficient
- Given a flow whose sticky `classified_protocol` is already `Some(S7Protocol::Classic)`
- When a later DT frame on the same flow carries a `protocol_id` other than `Some(0x32)`
  (e.g. `Some(0x72)`, `Some(other)`, or `None`)
- Then `parse_s7comm_header` is NOT called for this frame — a `0x32`-leading frame's own
  `protocol_id` byte is not sufficient on its own without the sticky-Classic gate
  (AC-187-004's positive case), and, symmetrically, a sticky-Classic flow does not
  dissect every subsequent frame regardless of that frame's own `protocol_id` byte
- **Test:** `test_BC_2_21_002_non_0x32_dt_frame_on_classic_flow_not_dissected`

### AC-187-013: A canonical, independently-sourced classic S7comm frame pair (Job request and Ack_Data response) validates the parser and dispatch against a real-world byte layout, per policy DF-CANONICAL-FRAME-HOLDOUT-001
(traces to BC-2.21.002 postcondition 3; BC-2.21.006; BC-2.21.008 postconditions 1-2)
- Given a canonical, independently-sourced classic S7comm Setup Communication (Job,
  function code `0xF0`) PDU, wrapped in real ISO-on-TCP framing — an RFC 1006 TPKT
  header followed by a standard class-0 COTP Data (DT) TPDU header (`02 F0 80`)
- When `on_data` dispatches this frame
- Then `S7commFlowState.classified_protocol == Some(S7Protocol::Classic)` and no
  `Finding` is emitted
- The TPKT length field, the `02 F0 80` COTP DT header bytes, and the S7comm
  common-header/Setup-Communication-parameter bytes MUST NOT be derived from this
  story's own BCs, ADR-014, or any other project artifact — per policy
  DF-CANONICAL-FRAME-HOLDOUT-001, they MUST be sourced independently from an
  authoritative reference. Sourcing is complete (pass-3 F-28): primary citation is
  cnblogs, "西门子S7通讯协议引用整理",
  https://www.cnblogs.com/crcce-dncs/p/10659087.html (Setup Communication
  request/response byte sequence), corroborated by Yiqisoft
  (https://www.yiqisoft.cn/blogs/IoT-Gateway/363.html) and the Inductive Automation
  KB ("Loggers - Device Connections: Siemens") — permitted as test-vector sources
  per ADR-014 Decision 4's 2026-09-24 reconciliation note (STORY-187 per-story
  adversarial pass 5, F-40, human ruling): publicly posted wire-capture byte
  examples are permitted as test-vector sources only, satisfying
  DF-CANONICAL-FRAME-HOLDOUT-001; parser design and field semantics for this story
  continue to derive from Decision 4's prose sources (Wireshark wiki, Kleinmann &
  Wool 2014, Orange-Cyberdefense catalog) and permitted design references
  (cisagov/icsnpp-s7comm BSD-3, kprovost/libs7comm BSD-2, gijzelaerr/python-snap7
  MIT)
- The test's doc comment MUST cite the authoritative source (document title/URL and
  section) the canonical byte sequence was taken from
- Given the canonical, independently-sourced classic S7comm Setup Communication
  **Ack_Data (`0x03`) response** frame now documented in BC-2.21.008's Canonical Test
  Vectors (source: cnblogs "西门子S7通讯协议引用整理",
  https://www.cnblogs.com/crcce-dncs/p/10659087.html — full 27-byte frame `03 00 00 1B
  02 F0 80 32 03 00 00 FF FF 00 08 00 00 00 00 F0 00 00 01 00 01 00 F0`, wrapped in the
  same RFC 1006 TPKT + standard class-0 COTP DT (`02 F0 80`) framing as the Job request
  above; corroborated by two further independent sources per BC-2.21.008's
  Description — Yiqisoft 2023-03-22 and the Inductive Automation KB)
- When `on_data` dispatches this Ack_Data response frame
- Then `parse_s7comm_header` returns `Some(header)` with `header.rosctr ==
  Rosctr::AckData`, `header.header_len == 12`, `header.error_class == Some(0)`, and
  `header.error_code == Some(0)` — NOT the pre-ruling (v1.0/v1.1) assumption of a
  10-byte Ack_Data header with any parameter block starting at `data[10]`; the
  parameter block instead begins at `data[12]` (BC-2.21.008 v1.2, corrected 2026-09-24)
  — and `S7commFlowState.classified_protocol == Some(S7Protocol::Classic)`; no
  `Finding` is emitted, since the frame is well-formed
- Given the committed `tests/fixtures/s7comm-setup-comm.pcap` capture (synthetic,
  generated by `tests/fixtures/mk_s7comm_pcap.py` per ADR-014 Decision 7 — distinct
  from the independently-sourced canonical byte literals above), carrying a full
  COTP CR/CC handshake followed by a Setup Communication Job/Ack_Data PDU pair under
  the corrected 12-byte Ack_Data header (pass-3 F-25)
- When every TCP-payload-bearing packet in the fixture is read via
  `PcapSource::from_file` and fed through `S7commAnalyzer::on_data` in sequence
- Then zero `Finding`s are emitted, `S7commFlowState.session_established == true`
  (from the CR/CC handshake), and `S7commFlowState.classified_protocol ==
  Some(S7Protocol::Classic)` (from the fixture's `0x32` DT frames) — an end-to-end
  regression check that the fixture generator's corrected 12-byte Ack/Ack_Data
  header stays well-formed under the full dispatch path
- **Tests:** `test_BC_2_21_006_canonical_setup_communication_job_frame_on_data`,
  `test_BC_2_21_008_canonical_ack_data_setup_communication_response_on_data`,
  `test_BC_2_21_002_setup_comm_fixture_pcap_well_formed_no_findings`

### AC-187-006: `parse_s7comm_header` returns None for input shorter than 10 bytes
(traces to BC-2.21.004 postcondition 1)
- Given `data.len() < 10`
- When `parse_s7comm_header(data)` is called
- Then returns `None`; no bytes beyond the length check are accessed; no panic for any
  `data.len()` in `[0, 9]` (traces to BC-2.21.004 postcondition 2)
- The first occurrence per flow direction emits one T0814
  (`ThreatCategory::Anomaly`/`Verdict::Possible`/`Confidence::Medium`) via
  `malformed_header_reported_c2s`/`_s2c` (traces to BC-2.21.004 postcondition 4)
- The emitted `Finding`'s evidence text is reason-specific — the test asserts the
  evidence text identifies this condition (header shorter than the 10-byte minimum) with
  wording distinct from the AC-187-009 (unrecognized ROSCTR), AC-187-010 (truncated Ack /
  truncated Ack_Data), and AC-187-011 (bounds-check failure) reasons, so the five
  malformed-header conditions sharing the `malformed_header_reported_c2s`/`_s2c` dedup
  flag remain distinguishable from the emitted `Finding` alone — the shared
  `assert_reason_specific_evidence` test helper (used by this AC's, AC-187-009's,
  AC-187-010's, and AC-187-011's evidence-text tests) additionally asserts that
  `finding.summary` is non-empty and contains the same reason text as `finding.evidence`
  verbatim, since `report_malformed_header` formats `summary` from the identical
  `reason` string it pushes into `evidence`
- Per-direction dedup is verified independently for BOTH the c2s and s2c directions
- **Tests:** `test_BC_2_21_004_len_shorter_than_10_returns_none_and_emits_t0814_once`,
  `test_BC_2_21_004_len_shorter_than_10_emits_t0814_once_s2c`
- Given a classic-S7comm DT frame, on a flow whose sticky `classified_protocol` is
  already `Some(Classic)`, whose payload is exactly 9 bytes (one byte short of the
  10-byte minimum)
- When `on_data` dispatches it
- Then exactly one T0814 `Finding` is emitted, with evidence text containing the exact
  boundary value "header too short: 9" — confirmed via the `on_data` dispatch path
  rather than a direct `parse_s7comm_header` call alone (pass-13/14 addition)
- **Test:** `test_BC_2_21_004_nine_byte_payload_on_data_too_short_evidence`
- **Note (pass-13/14/16, corrected pass-16 P16-F-1):** the shared
  `assert_malformed_header_t0814` test helper (used by this AC's, AC-187-009's, and
  AC-187-011's `on_data`-driven tests) pins `finding.category == ThreatCategory::Anomaly`,
  `finding.verdict == Verdict::Possible`, `finding.confidence == Confidence::Medium`,
  `finding.mitre_techniques == vec!["T0814".to_string()]` EXACTLY — not merely that
  `"T0814"` is present among other entries — and `finding.direction ==
  Some(expected_direction)`; the helper itself does NOT assert `finding.timestamp`.
  Timestamp is asserted directly (not via the shared helper) by exactly two tests
  under this AC (four story-wide; see AC-187-009/011 notes):
  `test_BC_2_21_004_len_shorter_than_10_returns_none_and_emits_t0814_once`
  and `test_BC_2_21_004_len_shorter_than_10_emits_t0814_once_s2c`, each asserting
  `finding.timestamp == chrono::DateTime::from_timestamp(ts as i64, 0)` against a
  non-zero `ts` value (so the assertion cannot vacuously pass against chrono's
  zero-epoch default). This is sufficient coverage for all five malformed-header reason
  classes (too-short, unrecognized ROSCTR, truncated Ack, truncated Ack_Data, bounds-check
  failure) because every one of them funnels through the single
  `report_malformed_header` emission path (`src/analyzer/s7comm.rs`), which performs the
  identical `chrono::DateTime::from_timestamp(ts as i64, 0)` conversion regardless of
  which reason triggered it — so these two tests exercise the shared conversion logic
  itself, not a reason-specific code path.

### AC-187-007: `parse_s7comm_header` defensively rejects `data[0] != 0x32`
(traces to BC-2.21.005 postcondition 1)
- Given `data.len() >= 10` and `data[0] != 0x32`
- When `parse_s7comm_header(data)` is called
- Then returns `None`; no other bytes are accessed once `data[0]` fails the equality
  check; no `Finding` is emitted — this is a defense-in-depth caller-hygiene contract,
  not a wire-observable anomaly (traces to BC-2.21.005 postcondition 3)
- **Test:** `test_BC_2_21_005_defensive_reject_wrong_protocol_id_byte`

### AC-187-008: `parse_s7comm_header` extracts common header fields for Job/Userdata (Ack_Data is handled with Ack — see AC-187-010)
(traces to BC-2.21.006 postcondition 1)
- Given `data.len() >= 10`, `data[0] == 0x32`, `data[1] ∈ {0x01, 0x07}` (Job or Userdata
  only — Ack_Data (`0x03`) is NOT part of this happy path as of the 2026-09-24
  canonical-frame holdout ruling, DF-CANONICAL-FRAME-HOLDOUT-001; Ack_Data now requires
  the 12-byte header covered by AC-187-010/BC-2.21.008)
- When `parse_s7comm_header(data)` is called
- Then returns `Some(S7commHeader { rosctr, pdu_reference, param_length, data_length,
  error_class: None, error_code: None, header_len: 10 })` with fields extracted exactly
  per the byte offsets: `pdu_reference = u16::from_be_bytes([data[4], data[5]])`
  (traces to BC-2.21.006 postcondition 2), `param_length = u16::from_be_bytes([data[6],
  data[7]])` (postcondition 3), `data_length = u16::from_be_bytes([data[8], data[9]])`
  (postcondition 4)
- `data[2..4]` (Reserved) is read for header-length bookkeeping only, never compared or
  branched on (traces to BC-2.21.006 postcondition 5) — a non-zero Reserved value does
  not cause rejection, confirmed by `test_BC_2_21_006_nonzero_reserved_bytes_do_not_reject`
- **Test:** `test_BC_2_21_006_common_header_field_extraction`

### AC-187-009: `parse_s7comm_header` returns None for an unrecognized ROSCTR byte; totality verified across all 256 possible u8 values
(traces to BC-2.21.007 postcondition 1)
- Given `data.len() >= 10`, `data[0] == 0x32`, `data[1] ∉ {0x01, 0x02, 0x03, 0x07}`
- When `parse_s7comm_header(data)` is called
- Then returns `None` for all 252 remaining `u8` values; no panic (traces to BC-2.21.007
  postcondition 2)
- Emits one T0814 via the same `malformed_header_reported_c2s`/`_s2c` dedup flag as
  AC-187-006 (traces to BC-2.21.007 postcondition 3 — this is the same dedup flag, not a
  second, distinct anomaly class); per-direction dedup is verified independently for
  BOTH the c2s and s2c directions — c2s via `test_BC_2_21_007_shares_dedup_flag_with_004_malformed_header`
  (a too-short frame sets the c2s dedup flag first, then a *different* malformed
  condition, unrecognized ROSCTR, arriving on the same c2s direction is suppressed —
  proving the flag is shared across reasons on c2s), s2c via
  `test_BC_2_21_007_unrecognized_rosctr_emits_t0814_once_s2c` (first s2c
  unrecognized-ROSCTR occurrence emits, independent of the c2s flag already being set;
  a repeated s2c occurrence of the SAME reason is then suppressed)
- The emitted `Finding`'s evidence text is reason-specific — the test asserts the
  evidence text identifies this condition (unrecognized ROSCTR byte) with wording
  distinct from the AC-187-006 (too-short header), AC-187-010 (truncated Ack / truncated
  Ack_Data), and AC-187-011 (bounds-check failure) reasons — one of the five
  malformed-header conditions sharing the `malformed_header_reported_c2s`/`_s2c` dedup
  flag; as of commit b4fce34a, `test_BC_2_21_007_unrecognized_rosctr_emits_t0814_once_s2c`
  asserts the exact rendered evidence text "unrecognized ROSCTR byte 0x00" (the frame's
  `data[1] == 0x00`), not merely a generic substring (P20-F-2)
- The test suite exercises all 256 possible `data[1]` byte values (`0x01`/`0x07`
  handled by BC-2.21.006, requiring `data.len() >= 10`; `0x02`/`0x03` handled by
  BC-2.21.008, requiring `data.len() >= 12`; the remaining 252 bytes handled by this
  BC, `None` regardless of length) to establish joint ROSCTR-byte totality across
  BC-2.21.006/007/008 — i.e. `parse_s7comm_header(data)` returns `Some` **iff**
  (`data[1] ∈ {0x01, 0x07}` and `data.len() >= 10`) or (`data[1] ∈ {0x02, 0x03}` and
  `data.len() >= 12`); this length-conditional totality was corrected 2026-09-24 per
  the canonical-frame holdout ruling (DF-CANONICAL-FRAME-HOLDOUT-001) — Ack_Data
  (`0x03`) is no longer grouped with the 10-byte-minimum values
- **Tests:** `test_BC_2_21_007_unrecognized_rosctr_returns_none`,
  `test_BC_2_21_007_shares_dedup_flag_with_004_malformed_header`,
  `test_BC_2_21_007_unrecognized_rosctr_emits_t0814_once_s2c`,
  `proptest_bc_2_21_007_rosctr_byte_totality_over_all_256_values`,
  `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional`
- **Note (pass-13/14/16, corrected pass-16 P16-F-1):** see AC-187-006's note for the
  shared `assert_malformed_header_t0814` helper's exact assertion set (category,
  verdict, confidence, `mitre_techniques == vec!["T0814".to_string()]` exactly,
  direction) — the helper does not assert `finding.timestamp`. For this AC, timestamp
  is asserted by exactly one test, `test_BC_2_21_007_unrecognized_rosctr_emits_t0814_once_s2c`
  (`finding.timestamp == chrono::DateTime::from_timestamp(ts as i64, 0)` against a
  non-zero `ts` value, asserted on its s2c finding); `test_BC_2_21_007_unrecognized_rosctr_returns_none`
  does not assert timestamp — it is a direct `parse_s7comm_header(&data)` call test (no
  `on_data`, no `S7commAnalyzer`, no `Finding` emitted at all), confirmed via grep of
  `tests/s7comm_analyzer_tests.rs`, so "timestamp" does not apply to it. This
  single assertion is sufficient because `report_malformed_header` performs the
  same timestamp conversion for every reason class, and AC-187-006's two tests already
  exercise that conversion directly (see AC-187-006's note).

### AC-187-010: ROSCTR=Ack AND ROSCTR=Ack_Data both require 12 bytes and extract error_class/error_code; truncated Ack_Data (10/11 bytes) returns None
(traces to BC-2.21.008 postcondition 1)
- Given `data[1] ∈ {0x02, 0x03}` (Ack or Ack_Data) and `data.len() < 12`
- When `parse_s7comm_header(data)` is called
- Then returns `None` (malformed-header, shares the dedup flag with AC-187-006/009) —
  this applies identically whether `data[1] == 0x02` (Ack) or `data[1] == 0x03`
  (Ack_Data); per the 2026-09-24 canonical-frame holdout ruling
  (DF-CANONICAL-FRAME-HOLDOUT-001), Ack_Data is NOT a 10-byte-only ROSCTR — a 10- or
  11-byte Ack_Data frame is truncated and returns `None`, exactly like a truncated Ack
- Given `data.len() >= 12`
- Then returns `Some(S7commHeader { rosctr: Ack | AckData, error_class: Some(data[10]),
  error_code: Some(data[11]), header_len: 12, .. })` (traces to BC-2.21.008
  postcondition 2) — for `rosctr == AckData`, the parameter block (if any) begins at
  `data[12]`, NOT `data[10]` (the pre-ruling v1.0/v1.1 assumption); function-code
  classification of that parameter block is STORY-188's scope, not this story's, but
  this story's `header_len == 12` extraction is what makes the corrected offset
  available to STORY-188
- Given the BC-2.21.008 Canonical Test Vectors table's Ack 12-byte happy-path row taken
  verbatim (`32 02 00 00 00 01 00 00 00 00 00 00`, no byte substituted)
- When `parse_s7comm_header(data)` is called
- Then returns `Some(S7commHeader { rosctr: Ack, pdu_reference: 1, param_length: 0,
  data_length: 0, error_class: Some(0), error_code: Some(0), header_len: 12 })` exactly
  as the BC specifies (added commit b4fce34a, `test_BC_2_21_008_canonical_ack_vector_verbatim`
  — closes the gap left by the two 12-byte happy-path tests below, which deliberately
  substitute distinct, non-zero error bytes 0x81/0x04 (P12-F-2 mutation-catching
  improvement) into the canonical-vector SHAPE rather than using the canonical vector's
  own `Some(0)`/`Some(0)` error fields verbatim)
- `error_class`/`error_code` are `Some` only when `rosctr ∈ {Ack, AckData}`; every other
  `S7commHeader` (Job, Userdata) has both fields `None` (traces to BC-2.21.008
  postcondition 3)
- Postcondition 4's obligation to log/surface the extracted `error_class`/`error_code`
  values is OUT OF SCOPE for this story's parse-only contract for BOTH Ack and Ack_Data
  — this story's scope ends at successfully returning them from `parse_s7comm_header`;
  consumption/logging is deferred to STORY-188 (F-13 ruling, human-ratified 2026-09-24,
  rescoped to explicitly include Ack_Data per DF-CANONICAL-FRAME-HOLDOUT-001,
  2026-09-24)
- Given an `on_data` call delivering a classic-S7comm DT frame whose ROSCTR byte
  (`data[1]`) is `0x02` (Ack) and whose S7comm payload length (the slice passed to
  `parse_s7comm_header`) is exactly 10 bytes, and separately a variant whose S7comm
  payload length is exactly 11 bytes (both one and two bytes short of the 12-byte Ack
  minimum)
- When `on_data` dispatches each variant
- Then `parse_s7comm_header` returns `None` and exactly one T0814 `Finding` is emitted
  per occurrence, with evidence text that identifies the reason as "truncated Ack"
  (distinct from the AC-187-006 "too short" and AC-187-009 "unrecognized ROSCTR"
  reasons) — verified for both the 10-byte and the 11-byte case
- Given the same `on_data` scenario but with `data[1] == 0x03` (Ack_Data) instead of
  `0x02`, at exactly 10 bytes and, separately, exactly 11 bytes
- When `on_data` dispatches each variant
- Then `parse_s7comm_header` returns `None` and exactly one T0814 `Finding` is emitted
  per occurrence, with evidence text that identifies the reason as "truncated Ack_Data"
  — distinguishable from the "truncated Ack" reason above even though both share the
  `malformed_header_reported_c2s`/`_s2c` dedup flag
- **Tests:** `test_BC_2_21_008_canonical_ack_vector_verbatim`,
  `test_BC_2_21_008_ack_rosctr_12_byte_minimum_and_error_fields`,
  `test_BC_2_21_008_ack_data_12_byte_header_and_error_fields`,
  `test_BC_2_21_008_truncated_ack_data_returns_none`,
  `test_BC_2_21_008_truncated_ack_on_data_emits_t0814_once`,
  `test_BC_2_21_008_truncated_ack_data_on_data_emits_t0814_once`,
  `test_BC_2_21_008_error_fields_none_for_job_and_userdata_rosctr` (postcondition
  3's Job/Userdata `error_class`/`error_code == None` bullet above; this test name
  supersedes an earlier `..._for_non_ack_rosctr` naming — DF-AC-TEST-NAME-SYNC-001)
- **Note (pass-11, mutation-discriminating):** the `error_class`/`error_code` extraction
  tests (`test_BC_2_21_008_ack_rosctr_12_byte_minimum_and_error_fields` and
  `test_BC_2_21_008_ack_data_12_byte_header_and_error_fields`) MUST use distinct,
  non-zero byte values for `data[10]` (`error_class`) and `data[11]` (`error_code`) — e.g.
  `data[10] = 0x05`, `data[11] = 0x0A` — never a shared value and never `0x00` for both.
  A test using `0x00`/`0x00` or the same byte for both fields cannot distinguish a mutant
  that swaps `error_class`/`error_code`, or one that hard-codes either field to a
  constant, from correct extraction.
- Given a 12-byte Ack_Data (`0x03`) header carrying non-zero, distinct `error_class`
  (`0x81`) and `error_code` (`0x04`) values, followed by a non-empty parameter block
  (`param_length == 2` with the 2 declared parameter bytes present, total frame length
  14 bytes)
- When `parse_s7comm_header(data)` is called directly, and separately when the same
  frame is dispatched via `on_data` on a flow that self-classifies Classic from this
  frame
- Then `parse_s7comm_header` returns `Some(header)` with `header.header_len == 12`
  (unaffected by the trailing parameter block — `header_len` describes the fixed header
  only, not the parameter/data blocks that follow it), `header.error_class ==
  Some(0x81)`, and `header.error_code == Some(0x04)`; via `on_data`, the bounds check
  passes and no `Finding` is emitted (pass-13/14 addition; traces to BC-2.21.008 Edge
  Case EC-007) — **N-2:** this test covers only the header/error-field **extraction**
  half of EC-007's shape (parse succeeds, bounds pass, no finding); it does not parse or
  classify the trailing parameter block's Group-3 function code at `data[12]` in any
  way — that classification half is out of this story's scope and belongs to STORY-188
- **Test:** `test_BC_2_21_008_ack_data_nonzero_error_fields_with_parameter_block`

### AC-187-011: Declared param_length/data_length are bounds-checked before any slice access; dedup verified for both flow directions
(traces to BC-2.21.009 postcondition 1)
- Given `parse_s7comm_header(data)` returned `Some(header)` and
  `data.len() < header.header_len + header.param_length as usize + header.data_length
  as usize`
- When `S7commAnalyzer` attempts to slice out the parameter/data blocks
- Then no slice into `data` beyond `data.len()` is ever attempted; the frame is treated
  as malformed (one T0814 per flow direction, sharing the
  `malformed_header_reported_c2s`/`_s2c` dedup flag — traces to BC-2.21.009
  postcondition 2)
- No function-code or Userdata classification is attempted for a frame that fails this
  check (traces to BC-2.21.009 postcondition 3)
- The emitted `Finding`'s evidence text is reason-specific — the test asserts the
  evidence text identifies this condition (declared `param_length`/`data_length` exceeds
  available data — bounds-check failure) with wording distinct from the AC-187-006
  (too-short header), AC-187-009 (unrecognized ROSCTR), and AC-187-010 (truncated Ack /
  truncated Ack_Data) reasons — one of the five malformed-header conditions sharing the
  `malformed_header_reported_c2s`/`_s2c` dedup flag
- Per-direction dedup (the shared `malformed_header_reported_c2s`/`_s2c` flag) is
  verified independently for BOTH the c2s and s2c directions — as of commit b4fce34a,
  `test_BC_2_21_009_bounds_check_before_parameter_data_slice` now repeats the SAME
  malformed c2s delivery a second time and asserts `findings.len() == 1` (the repeat
  does not re-emit), in addition to `test_BC_2_21_009_bounds_check_dedup_s2c`'s
  independent s2c verification
- Given an `on_data`-delivered classic-S7comm DT frame carrying a 12-byte Ack or
  Ack_Data header (`header.header_len == 12`, per BC-2.21.008 — applies identically for
  `rosctr == Ack` and `rosctr == AckData`) with `header.param_length == 1` and
  `header.data_length == 0`
- When the S7comm payload length (the slice passed to `parse_s7comm_header`) is exactly
  12 bytes (the declared parameter byte is absent)
- Then `s7comm_bounds_ok(&header, data.len())` returns `false` and one T0814 `Finding`
  is emitted (bounds-check-failure reason)
- When, in a second case, the S7comm payload length is exactly 13 bytes (the declared
  parameter byte is present)
- Then `s7comm_bounds_ok(&header, data.len())` returns `true` and no `Finding` is
  emitted for the bounds check — this case verifies the check correctly incorporates the
  Ack/Ack_Data-specific `header.header_len == 12` base rather than assuming the
  10-byte Job/Userdata default (corrected 2026-09-24 per the canonical-frame holdout
  ruling, DF-CANONICAL-FRAME-HOLDOUT-001 — Ack_Data is no longer assumed to use the
  10-byte default)
- Given an `on_data`-delivered classic-S7comm DT frame carrying a valid 10-byte
  Job/Userdata header with `header.param_length == 0` and `header.data_length` alone
  (not `param_length`) exceeding the bytes remaining after the header
- When `on_data` dispatches the frame
- Then `s7comm_bounds_ok(&header, data.len())` returns `false` and one T0814 `Finding` is
  emitted — this isolates the `data_length` term of the bounds sum from the
  `param_length` term already exercised by the AC-187-013/N-3 Ack/Ack_Data cases above, so
  a mutant that drops, zeroes, or swaps either term of `header_len + param_length +
  data_length` independently is caught rather than passing by coincidence
- Given a single TCP delivery carries a well-formed classic-S7comm DT frame — its own
  TPKT-declared length correctly bounds it to exactly the 10-byte common header, whose
  S7comm header declares `param_length == 2` with zero parameter bytes actually present
  within that frame's own boundary — immediately followed, within the same delivery, by
  a complete, distinct COTP CR frame
- When `on_data` dispatches the delivery
- Then the bounds check for the DT frame is computed against that frame's own
  TPKT-declared length only (10 bytes available) — never borrowing bytes from the
  trailing CR frame — so `s7comm_bounds_ok` correctly fails (12 declared > 10 available)
  and exactly one T0814 `Finding` is emitted (evidence text containing "exceed available
  bytes", never more than one and never zero); the trailing CR frame is still walked and
  dispatched within the same delivery — confirmed by `S7commFlowState.cr_observed_dir ==
  Some(Direction::ClientToServer)` — proving the cursor advanced past the DT frame using
  its own declared TPKT length rather than an over-wide slice that would otherwise have
  absorbed the CR frame's bytes too
- **Tests:** `test_BC_2_21_009_bounds_check_before_parameter_data_slice`,
  `test_BC_2_21_009_bounds_check_dedup_s2c`,
  `test_BC_2_21_009_ack_header_len_12_bounds_check`,
  `test_BC_2_21_009_data_length_overrun_on_data_emits_t0814`,
  `test_BC_2_21_009_dissection_bounded_to_own_tpkt_frame`,
  `test_BC_2_21_009_bounds_failure_evidence_reports_declared_and_available` (Job
  header, `header_len=10`, `param_length=3`, `data_length=5` -> declared 18,
  available 12 -- T0814 evidence reports the exact declared and available byte
  counts, F-11 reason-specific evidence, added commit 6705ed8b to kill
  cargo-mutants survivors at `s7comm.rs:779`),
  `test_BC_2_21_009_bounds_failure_evidence_ack_data_header_len_12` (Ack_Data
  header, `header_len=12`, `param_length=3`, `data_length=5` -> declared 20,
  available 14 -- same reason-specific evidence-text obligation as the Job case,
  confirmed across both `header_len` bases, added commit 6705ed8b)
- **Note (pass-13/14/16, corrected pass-16 P16-F-1):** see AC-187-006's note for the
  shared `assert_malformed_header_t0814` helper's exact assertion set (category,
  verdict, confidence, `mitre_techniques == vec!["T0814".to_string()]` exactly,
  direction) — the helper does not assert `finding.timestamp`. For this AC, timestamp
  is asserted by exactly one test, `test_BC_2_21_009_bounds_check_dedup_s2c` (the s2c
  bounds-check-failure dedup test; `finding.timestamp ==
  chrono::DateTime::from_timestamp(ts as i64, 0)` against a non-zero `ts` value);
  `test_BC_2_21_009_bounds_check_before_parameter_data_slice` and
  `test_BC_2_21_009_data_length_overrun_on_data_emits_t0814` do not assert timestamp.
  This single assertion is sufficient because `report_malformed_header` performs the
  same timestamp conversion for every reason class, and AC-187-006's two tests already
  exercise that conversion directly (see AC-187-006's note).
- **Note (N-3, clarity):** `test_BC_2_21_009_ack_header_len_12_bounds_check` exercises
  the Ack (`0x02`) case only; Ack_Data (`0x03`) shares the identical
  `header.header_len == 12` code path and is exercised live (not merely by symmetry
  argument) by `test_BC_2_21_008_canonical_ack_data_setup_communication_response_on_data`
  (AC-187-013), so both ROSCTR values that take the 12-byte bounds-check base are covered
  by a concrete test, not just this AC's Ack-only case.

## Architecture Mapping

| Component | Module | File | Pure/Effectful |
|-----------|--------|------|---------------|
| `S7commFlowState` (completed) | SS-21 per-flow state | `src/analyzer/s7comm.rs` | Mutable state |
| `S7Protocol` enum | SS-21 data model | `src/analyzer/s7comm.rs` | N/A (`Classic`, `Plus`, `Unclassified`; full population in STORY-190) |
| `S7commHeader` struct | SS-21 data model | `src/analyzer/s7comm.rs` | N/A (frozen per ADR-014 Decision 9 item 3) |
| `Rosctr` enum | SS-21 data model | `src/analyzer/s7comm.rs` | N/A (`Job`, `Ack`, `AckData`, `Userdata`) |
| `parse_s7comm_header` | SS-21 S7comm header parser | `src/analyzer/s7comm.rs` | Pure (free fn, VP-051 target) |
| `S7commAnalyzer::on_data` (dispatch extension) | SS-21 effectful shell | `src/analyzer/s7comm.rs` | Effectful |

Subsystem anchor: SS-21 owns this story's scope because `S7commFlowState`,
`parse_s7comm_header`, and the `protocol_id` dispatch skeleton are the core data model
and entry point of the S7comm analyzer per ARCH-INDEX.md §SS-21.

## Purity Classification

| Module | Classification | Justification |
|--------|---------------|---------------|
| `parse_s7comm_header` | pure-core | Returns `Option<S7commHeader>` by value; no mutation, no I/O; defensively re-checks `data[0]` but performs no side effects |
| `S7commHeader`, `Rosctr`, `S7Protocol` | pure-core | Plain data types |
| `S7commAnalyzer::on_data` (dispatch extension) | effectful-shell | Mutates `S7commFlowState`, calls `parse_s7comm_header`, emits T0814 on malformed-header conditions |

## VP-051 Kani Obligation

**Harnesses (corrected pass-16, P16-F-3 — two separate `#[kani::proof]` harnesses, not
one):**
- `verify_parse_s7comm_header_bounds_safety` — the header-extraction half (bounded
  symbolic `data: [u8; 16]` + `len`); covers the `Some`/`None` totality and the
  positive field-extraction obligations below.
- `verify_s7comm_bounds_ok_bounds_safety` — the bounds-check half (F-18; a second,
  independent symbolic `data_len: usize` input against `s7comm_bounds_ok`); covers the
  exact-equality and no-panic obligations in the "Bounds-check half —
  `verify_s7comm_bounds_ok_bounds_safety` (F-18, ...)" paragraph below.

Both harnesses are anchored in this story (`tests/s7comm_analyzer_tests.rs`, `#[cfg(kani)]`
module). STORY-194 (the full non-vacuous VP-051 proof run) MUST execute BOTH harnesses,
not merely the first — the bounds-check half's `kani::cover!` non-vacuity obligations are
independent of the header-extraction half's and are not satisfied by running only
`verify_parse_s7comm_header_bounds_safety`.

**Method:** Kani symbolic execution
**Priority:** P0

Covers BC-2.21.004 (10-byte minimum), BC-2.21.006 (10-byte Job/Userdata common-header
field extraction, postconditions 2-4), BC-2.21.007 (unrecognized-ROSCTR safe-reject),
BC-2.21.008 (the Ack/Ack_Data 12-byte header extension and `error_class`/`error_code`
extraction), and BC-2.21.009 (the caller-side bounds obligation `header_len +
param_length + data_length` cannot overflow `usize`, and no slice beyond `data.len()` is
ever constructed) — VP-051's source BCs are {BC-2.21.004, BC-2.21.006, BC-2.21.007,
BC-2.21.008, BC-2.21.009} per VP-INDEX.md (F-49 — architect registering BC-2.21.006/007
alongside the prior BC-2.21.004/008/009 set). CONFIRMED Kani P0 (not merely
cargo-fuzz P1) per
ADR-014 Decision 9 item 3's reconciliation note (F-14, human-ratified 2026-09-24) and
VP-INDEX.md v2.51 — `parse_s7comm_header` is loop-free and fixed-shape, matching
`parse_tpkt_header`/`parse_cotp_header`'s (VP-048/VP-049) Kani-tractability profile
rather than `parse_asdu`'s cargo-fuzz-only profile.

`header_len` selection is `12` when `rosctr ∈ {Ack, AckData}` (ROSCTR `0x02`/`0x03`) and
`10` otherwise (Job `0x01`, Userdata `0x07`) — corrected per the canonical-frame holdout
ruling (2026-09-24, DF-CANONICAL-FRAME-HOLDOUT-001, per VP-INDEX.md v2.52's VP-051 row):
both Ack AND Ack_Data carry the 12-byte error-field header, not Ack alone.

The harness MUST be **non-vacuous** per DF-KANI-NONVACUITY-001 — it must be checked to
actually exercise both the `Some` and `None` return paths — asserting at minimum:
- `data.len() < 10` implies `parse_s7comm_header(data) == None`
- `parse_s7comm_header(data) == Some(header)` implies `header.header_len ∈ {10, 12}` and
  `data.len() >= header.header_len`
- `header.error_class.is_some() == (header.rosctr == Rosctr::Ack || header.rosctr ==
  Rosctr::AckData)` (identically for `header.error_code`) — NOT the narrower
  `(header.rosctr == Rosctr::Ack)` alone, which would vacuously pass a
  `header_len`/`error_class` mismatch on every Ack_Data input (corrected per the
  canonical-frame holdout ruling, DF-CANONICAL-FRAME-HOLDOUT-001, human-ratified
  2026-09-24; mirrors VP-INDEX.md v2.52's VP-051 row and its non-vacuity requirement)

In addition to the above negative/implication-style assertions, the harness MUST assert
the following positive field-extraction obligations (F-47) — these pin down exactly what
`Some(header)` contains, not merely that it exists:
- `header.header_len == 12` iff `header.rosctr ∈ {Rosctr::Ack, Rosctr::AckData}`, else
  `header.header_len == 10` (restates the header-length selection rule above as a direct
  per-field equality the harness checks, not just an implication)
- `header.error_class == Some(data[10])` and `header.error_code == Some(data[11])` for
  `rosctr ∈ {Rosctr::Ack, Rosctr::AckData}` — the exact byte values, not merely
  `is_some()`
- `header.rosctr` is derived from `data[1]` (the ROSCTR byte maps directly to the
  `Rosctr` enum value at that offset)
- `header.pdu_reference`, `header.param_length`, and `header.data_length` equal the
  big-endian `u16` reads of `data[4..6]`, `data[6..8]`, and `data[8..10]` respectively
  (BC-2.21.006 postconditions 2-4)

The harness MUST use **bounded symbolic input** — e.g. a fixed-size `[u8; 16]` array via
`kani::any()` plus an assumed/bounded `len <= 16`, or an explicit `#[kani::unwind(N)]`
bound — never an unbounded `Vec<u8>`, mirroring `parse_tpkt_header`/`parse_cotp_header`'s
existing harness shape.

Per the F-14 reconciliation, BC-2.21.009's caller-side bounds check (`header_len +
param_length + data_length` vs. `data.len()`) MUST be extracted as a **pure,
public (`pub fn`) helper function** (e.g. `pub fn s7comm_bounds_ok(header: &S7commHeader,
data_len: usize) -> bool`) so the Kani harness can call it directly, rather than only
being exercisable via the effectful `on_data` call site — this keeps VP-051's proof
loop-free and independently callable. (P11-F-1, pass-11: corrected from "crate-visible"
— the harness that exercises this helper lives in `tests/s7comm_analyzer_tests.rs`, an
external integration-test binary outside the crate, so the helper must be `pub`, not
merely `pub(crate)`.)

**Bounds-check half — `verify_s7comm_bounds_ok_bounds_safety` (F-18, corrected pass-16
P16-F-3: a separate `#[kani::proof]` harness from `verify_parse_s7comm_header_bounds_safety`
above, not a second half of the same harness):** in addition to symbolic `data: [u8; 16]`
and its bounded `len` (used to obtain a `Some(header)` via `parse_s7comm_header`), this
harness MUST introduce a second, independent symbolic input
`data_len: usize` (via `kani::any()` with an assumed bound, e.g. `kani::assume(data_len
<= u16::MAX as usize * 3)` or an equivalently justified bound — representing the
length of the hypothetical full delivery the header's declared `param_length`/
`data_length` are checked against, decoupled from the small fixed-size symbolic array
used for header-field extraction) and assert:
- `s7comm_bounds_ok(&h, data_len) == (data_len as u64 >= h.header_len as u64 +
  h.param_length as u64 + h.data_length as u64)` for a header `h` obtained from a
  `Some` result of `parse_s7comm_header` on the symbolic array (the equality asserted
  directly, not merely one direction of the implication, so the helper's boolean result
  is proven exactly right — no false-accept and no false-reject)
- When the right-hand side is `true` (i.e. `data_len` is large enough), the resulting
  parameter/data sub-slice access — `data.get(h.header_len .. h.header_len +
  h.param_length as usize + h.data_length as usize)` against a symbolic slice of length
  `data_len` — is asserted to be `Some(..)`, never `None` and never a construct that
  could panic
- `kani::cover!` MUST be used to confirm the harness actually reaches BOTH the
  `s7comm_bounds_ok == true` outcome and the `s7comm_bounds_ok == false` outcome under
  the symbolic inputs — satisfying non-vacuity (DF-KANI-NONVACUITY-001) for this half of
  the harness specifically, independent of the `Some`/`None` cover obligations already
  stated above for the header-extraction half

Skeleton written here; full proof in STORY-194.

**Deferred (pass-3 F-30):** VP-INDEX's "deliberate-flip negative check" for VP-051
— mutating a known-good `header_len`/`error_class` mapping to confirm the harness
would actually catch the injected defect, rather than merely covering both outcomes
per `kani::cover!` above — is out of scope for this story's skeleton and is
deferred to STORY-194's full formal-hardening pass.

## VP-053 Proptest Obligation (partial — completed in STORY-190)

**Harness:** `proptest_vp053_protocol_id_dispatch_totality` (skeleton started here)
**Method:** proptest
**Priority:** P0

This story wires the CR/CC and `Some(0x32)` branches of the four-way dispatch
(BC-2.21.002), including the amended `protocol_id: None`-never-classifies semantics
(F-02) and the sticky-classification-gates-dissection semantics (F-12). Stated once,
here, for both deferrals (N-2): the `Some(0x72)` and unclassified/unrecognized branches
— required for VP-053's full totality proof — are completed in STORY-190, and the full
non-vacuous VP-053 run (exercising all four dispatch branches together) is deferred to
STORY-194. The dispatch branches for the not-yet-implemented `Some(0x72)`/unclassified
cases are `todo!()`-free structural no-ops, NOT a `todo!()`/placeholder stub — they
compile cleanly under `tdd_mode: strict`, and this story's proptest skeleton runs under
`cargo test` (no `cargo kani` or nightly toolchain required for this skeleton).

Per VP-INDEX.md v2.51's reworded VP-053 row ("Sticky-Classification Gating"), the
proptest input-generation strategy for this skeleton MUST include `protocol_id: None` as
a distinct, explicitly generated case (not merely an implicit absence/default) — the
property under test ("`protocol_id: None` does not participate in sticky
first-classification") is untestable unless the strategy can generate it alongside
`Some(0x32)`, `Some(0x72)`, and `Some(other)`.

## Tasks

- [ ] Extend `S7commFlowState` (from STORY-186) with: `session_established: bool`,
      `cr_observed_dir: Option<Direction>` (or an equivalently-purposed field) hosting
      pending-CR-direction tracking state (F-01), `classified_protocol:
      Option<S7Protocol>`, `malformed_header_reported_c2s: bool`,
      `malformed_header_reported_s2c: bool`
- [ ] Define `pub enum S7Protocol { Classic, Plus, Unclassified }` (skeleton; `Plus` and
      `Unclassified` variants are fully driven starting in STORY-190)
- [ ] Define `pub enum Rosctr { Job, Ack, AckData, Userdata }`
- [ ] Define `pub struct S7commHeader { pub rosctr: Rosctr, pub pdu_reference: u16,
      pub param_length: u16, pub data_length: u16, pub error_class: Option<u8>,
      pub error_code: Option<u8>, pub header_len: usize }`
- [ ] Implement `pub fn parse_s7comm_header(data: &[u8]) -> Option<S7commHeader>`:
  - `data.len() < 10` guard -> `None` (BC-2.21.004)
  - `data[0] != 0x32` defensive guard -> `None` (BC-2.21.005)
  - ROSCTR match: `0x01`/`0x07` -> common-header extraction, `header_len: 10`
    (BC-2.21.006); `0x02`/`0x03` -> Ack/Ack_Data extension requiring 12 bytes
    (BC-2.21.008, corrected 2026-09-24 canonical-frame holdout ruling
    DF-CANONICAL-FRAME-HOLDOUT-001 — Ack_Data is no longer part of the 10-byte group);
    any other byte -> `None` (BC-2.21.007)
- [ ] Implement the CR/CC session-tracking rule (F-01): a CC TPDU sets
      `session_established = true` only when `cr_observed_dir` (or equivalent) holds a
      direction OPPOSITE the CC's own direction; a CR TPDU records/updates the pending
      direction; CC-only, CC-before-CR, and same-direction CC leave
      `session_established` at `false`
- [ ] Extend `S7commAnalyzer::on_data`'s frame dispatch (from STORY-186) with the
      four-way `CotpHeader::protocol_id` branch (BC-2.21.002): `None` (session
      TPDU/unparseable) routes to session tracking or a STORY-190 placeholder;
      `Some(byte)` on a DT frame drives sticky first-classification (F-02: `None`
      `protocol_id` never classifies and does not consume "first DT frame" status);
      `Some(0x32)` DT is dissected via `parse_s7comm_header` ONLY when the flow's sticky
      `classified_protocol == Some(Classic)` (F-12 gate) and applies the bounds check
      (BC-2.21.009); `Some(0x72)` and `Some(other)` route to a `todo!()`-free
      placeholder no-op completed in STORY-190
- [ ] Implement the BC-2.21.009 caller-side bounds check before any parameter/data-block
      slice is constructed, as a **pure, public (`pub fn`) helper function** (e.g.
      `pub fn s7comm_bounds_ok`) callable both from `on_data` and directly from the
      VP-051 Kani harness (which lives in `tests/s7comm_analyzer_tests.rs`, external to
      the crate, so `pub(crate)` visibility would not suffice) (F-14, P11-F-1)
- [ ] Write `#[cfg(kani)]` VP-051 skeleton using bounded symbolic input (fixed-size array
      + assumed/bounded length, or `#[kani::unwind]`) and non-vacuous assertions per the
      VP-051 Kani Obligation section above
- [ ] Write `proptest_vp053_protocol_id_dispatch_totality` skeleton (partial, per the VP
      Obligation section above), with a strategy that explicitly generates the
      `protocol_id: None` case
- [ ] Write unit tests: one per AC, named `test_BC_2_21_001_*` .. `test_BC_2_21_009_*`,
      plus `test_BC_2_21_002_*` for AC-187-012 — including the five F-01 negative-case
      session tests (CR-only, CC-only, CC-before-CR, same-direction CC, and the F-22
      repeated-same-direction-CR test
      `test_BC_2_21_001_repeated_same_direction_cr_session_not_established`), the F-02
      None-then-0x32 and two-frames-one-delivery
      tests, the F-12 sticky-gating negative test, s2c-direction dedup tests for
      AC-187-006/009/011, reason-specific evidence-text assertions for the
      AC-187-006/009/011 malformed-header findings (F-21a), the on_data truncated-Ack
      test `test_BC_2_21_008_truncated_ack_on_data_emits_t0814_once` for both the 10-
      and 11-byte cases (F-21b), the Ack-header (`header_len == 12`) bounds test
      `test_BC_2_21_009_ack_header_len_12_bounds_check` for both the param-byte-absent
      and param-byte-present cases (F-21c), and — per the 2026-09-24 canonical-frame
      holdout ruling (DF-CANONICAL-FRAME-HOLDOUT-001) — the Ack_Data-specific tests
      `test_BC_2_21_008_ack_data_12_byte_header_and_error_fields`,
      `test_BC_2_21_008_truncated_ack_data_returns_none`,
      `test_BC_2_21_008_truncated_ack_data_on_data_emits_t0814_once`, and
      `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional`
- [ ] Write the AC-187-013 canonical-frame `on_data` test
      `test_BC_2_21_006_canonical_setup_communication_job_frame_on_data` (F-19, policy
      DF-CANONICAL-FRAME-HOLDOUT-001): source the canonical classic S7comm Setup
      Communication (`0xF0`) frame and its real TPKT + standard class-0 COTP DT
      (`02 F0 80`) framing independently of this project's BCs/ADR-014/tests — per
      research-agent sourcing, permitted as a test-vector source only under ADR-014
      Decision 4's 2026-09-24 reconciliation note (F-40, human ruling; wire-capture
      byte examples are test-vector sources only, not field-semantics sources) —
      and cite the authoritative source (document + section) in the test's doc
      comment
- [ ] Write the AC-187-013 companion canonical-frame `on_data` test
      `test_BC_2_21_008_canonical_ack_data_setup_communication_response_on_data`
      (canonical-frame holdout ruling, DF-CANONICAL-FRAME-HOLDOUT-001, 2026-09-24):
      feed the Ack_Data (`0x03`) Setup Communication response frame from BC-2.21.008's
      Canonical Test Vectors (already sourced from cnblogs, corroborated by Yiqisoft
      and the Inductive Automation KB) and assert `header_len == 12`,
      `error_class == Some(0)`, `error_code == Some(0)`, and
      `classified_protocol == Some(Classic)`
- [ ] Verify `cargo test` passes for this story's tests
- [ ] Extend `tests/fixtures/mk_s7comm_pcap.py` (CREATE, first use in this story) with
      Setup Communication (`0xF0`) and a minimal classic-S7comm Job/Ack_Data frame pair,
      per ADR-014 Decision 7 — synthetic, CC0/MIT, mirrors `mk_modbus_large_pcap.py`;
      extend with a CR/CC-opposite-direction pair (standard class-0 COTP DT `02 F0 80`
      framing). (F-23) The CC-only fixture and the two-DT-frames-in-one-delivery
      fixture needed for the F-01/F-02 test additions are covered by in-memory unit
      tests that construct byte slices directly — no pcap-generator support is required
      for those two cases; the generator's obligation for this story is limited to the
      CR/CC-opposite-direction pair and the Setup Communication classic-S7comm frame(s)
      (including, per AC-187-013, the canonical independently-sourced variant, which may
      be embedded directly in the test as a byte literal rather than routed through the
      generator). Per the 2026-09-24 canonical-frame holdout ruling
      (DF-CANONICAL-FRAME-HOLDOUT-001, pass-3 F-25), the generator's Ack_Data PDUs emit
      the corrected 12-byte header (error class/code at `data[10..12]`, parameter block
      at `data[12]`), not the pre-ruling 10-byte assumption; the committed
      `tests/fixtures/s7comm-setup-comm.pcap` output of this generator is exercised
      end-to-end (CR/CC handshake plus Setup Communication Job/Ack_Data walk) by
      `test_BC_2_21_002_setup_comm_fixture_pcap_well_formed_no_findings` (AC-187-013),
      which asserts zero findings, `session_established`, and
      `classified_protocol == Some(Classic)`
- [ ] Add a CHANGELOG entry under `[Unreleased] > Added` describing the S7comm header
      parser and dispatch skeleton, before creating the PR

**Scoped mutation-testing result (v1.13, commit 6705ed8b):** `cargo mutants` 27.1.0,
serial run (`--jobs 1`, per the PG-MUTANTS-JOBS-001 incident guidance in CLAUDE.md)
over the STORY-187 diff of `src/analyzer/s7comm.rs` — 43 mutants, 37 caught
initially, 3 real survivors in the evidence arithmetic at `s7comm.rs:779` (the
`declared`/`available` byte-count strings in the BC-2.21.009 bounds-check-failure
`Finding`), now killed by the two new tests
`test_BC_2_21_009_bounds_failure_evidence_reports_declared_and_available` (Job,
declared 18 = 10+3+5, available 12) and
`test_BC_2_21_009_bounds_failure_evidence_ack_data_header_len_12` (Ack_Data,
declared 20 = 12+3+5, available 14), added to AC-187-011's test list above; 1
equivalent mutant remains (the empty `Some(0x72)` placeholder arm in
`dispatch_cotp_frame` — this arm becomes killable only once STORY-190 fills it in
with observable behavior); 2 unviable mutants (non-compiling or otherwise excluded
by `cargo mutants`' own filtering).

**Accepted residual (pass-15, P15-F-1, NIT):** `dispatch_classic_s7comm` builds the
malformed-header evidence string (the `format!`-ed "declared"/"available" byte-count
text) BEFORE the per-direction `malformed_header_reported_c2s`/`_s2c` dedup check runs —
so a suppressed duplicate malformed frame on an already-dedup'd direction still
allocates and formats the evidence string even though the resulting `Finding` is
discarded and never emitted. This is bounded (one wasted allocation/format per
suppressed duplicate frame, not unbounded growth) and has no correctness impact — the
dedup guarantee itself (exactly one `Finding` per condition per direction) is unaffected.
Accepted as-is; flagged as a candidate micro-optimization (reorder the dedup check ahead
of the evidence-string construction) for a later story rather than blocking this one.

**Deferred (pass-17, P17-F-1):** VP-053's proptest strategy (`protocol_id_strategy` in
`tests/s7comm_analyzer_tests.rs`, `prop_oneof![Just(None), any::<u8>().prop_map(Some)]`)
does not explicitly weight generation toward `Some(0x32)` (Classic) or `Some(0x72)`
(Plus) — each is reachable only as 1 of 256 equally-likely `u8` values within the
`Some` branch, so proptest's default case count is not guaranteed to exercise either
value on every run. This story's skeleton scope (partial VP-053, N-2) does not require
fixing this; explicitly weighting the strategy toward `Some(0x32)`/`Some(0x72)` (e.g.
via a `prop_oneof!` with dedicated arms for those two bytes alongside the general `u8`
case) is deferred to STORY-190/STORY-194's full non-vacuous VP-053 run, where the
`Some(0x72)` branch is filled in and totality across all four dispatch branches is
proven.

**Accepted residual (pass-17, P17-F-2, NIT):** the truncated-Ack `on_data` test
(`test_BC_2_21_008_truncated_ack_on_data_emits_t0814_once`, AC-187-010) asserts only
that the emitted `Finding`'s evidence contains the substring "truncated Ack header"
(via `assert_reason_specific_evidence`) for both the 10-byte and 11-byte cases — it does
not assert the exact byte count (e.g. "10" vs "11") the way the too-short-header
(AC-187-006, "header too short: 9") and bounds-check-failure (AC-187-011, declared/
available counts) evidence tests do. This is a lesser-coverage gap, not a correctness
gap — the two sub-cases are still distinguished by test structure (separate blocks
asserting `findings.len() == 1` each), just not by evidence-text content. Accepted
as-is; not blocking this story.

**Deferred (pass-21, P21-F-1, LOW):** ADR-014's prose describing the four-way
`protocol_id` dispatch skeleton's placeholder branches uses "explicit unwind" wording
that does not precisely match this story's actual placeholder-no-op implementation
(the `Some(0x72)`/unclassified branches route to a no-op, not an unwind of any kind).
Reconciling the ADR's wording is deferred to STORY-194 rather than corrected here, since
it is an ADR-text clarity issue, not a behavioral gap in this story's code or tests.

**Accepted residual (pass-21, D-P21-1, NIT):** `on_data`'s carry-buffer handling
(`std::mem::take(&mut state.carry_c2s)` / `carry_s2c`, followed by re-combining with the
new delivery and re-walking from the start of the combined buffer each call — pre-
existing STORY-186 code in `src/analyzer/s7comm.rs`, not introduced by this story) is
worst-case quadratic in total bytes carried across many small deliveries, since each
call re-copies and re-walks the accumulated carry residual rather than resuming from a
saved offset. This is a CPU-amplification concern, not a correctness or bounds-safety
one (STORY-186's own bounds-safety tests already cover the residual-growth ceiling at
`MAX_S7_ISO_ON_TCP_CARRY_BYTES = 65,535`). Accepted as-is for this story; flagged as a
performance research item for phase-5 (formal hardening / performance) rather than
blocking TDD delivery here — validation of the actual amplification magnitude under
realistic small-segment traffic is deferred to that phase's research pass.

**Accepted residual (pass-21, N-3, NIT):** BC-2.21.006 EC-002 (`pdu_reference ==
0x0000`, extracted verbatim, not treated as invalid) and BC-2.21.009 EC-003 (a large,
plausible `data_length` — e.g. a multi-kilobyte Download Block payload — passes the
bounds check cleanly) have no dedicated concrete unit test in
`tests/s7comm_analyzer_tests.rs` isolating either case by name (confirmed via grep: the
sole `pdu_reference: 0` occurrence in the file, in
`test_BC_2_21_009_s7comm_bounds_ok_data_length_only_overrun`, is an incidental struct-
literal value in an unrelated `data_length`-isolation test, not a dedicated EC-002
assertion; no test references a multi-kilobyte or otherwise large `data_length`
scenario). Both cases are covered only symbolically, by VP-051's bounded Kani harnesses
(`pdu_reference` and `data_length` both range freely across their symbolic byte/length
inputs, and the harnesses assert the general extraction/bounds-check postconditions
that subsume these specific values as unexceptional points in that range). Accepted as
sufficient for this story given the symbolic coverage; a dedicated example-based test
for either case would be redundant with, not additive to, the Kani proof and is not
required to close this story.

**Convergence decision (2026-09-25, human ruling):** per-story adversarial review for
STORY-187 is closed after passes 22-24 (24 passes total). Pass 22: LOW/NIT wording
findings only — **P22-F-2 (LOW)**: AC-187-011's fifth Given/When/Then described a
generic "second, distinct TPKT frame" scenario that did not match its cited test,
`test_BC_2_21_009_dissection_bounded_to_own_tpkt_frame`, which specifically exercises a
Job header with `param_length == 2` and zero parameter bytes present, followed by a CR
frame, and asserts both the single T0814 ("exceed available bytes" evidence) and the
trailing CR frame's `cr_observed_dir` side effect; corrected to match the test body
(this v1.17 burst). **P22-F-3 (NIT)**: AC-187-010 and AC-187-011 described the truncated-
Ack/Ack_Data and Ack-header bounds test scenarios as "total frame length" when the tests
actually construct the S7comm payload (the slice passed to `parse_s7comm_header`) at the
stated byte counts before wrapping it in TPKT/COTP framing via `dt_frame`; reworded to
"S7comm payload length (the slice passed to `parse_s7comm_header`)" (this v1.17 burst).
Pass 23: one LOW test-adequacy finding, already fixed post-convergence with a
mutation-kill proof (commit b4fce34a and prior bursts — see the pass-16 mutation note
above), plus two NITs — **P23-F-1 (NIT)**:
`test_BC_2_21_001_session_established_is_monotonic` gained direct `cr_observed_dir`
assertions at each step (retained unchanged after the establishing CC; overwritten by a
further CR even once `session_established` is `true`), documented in AC-187-003's N-4
note (this v1.17 burst). **P23-F-3 (NIT)**:
the Tasks list's `test_BC_2_21_001_repeated_same_direction_cr_session_not_established`
test name was wrapped mid-identifier across two lines; rejoined onto one line (this
v1.17 burst; a full-story grep for other split identifiers found none). Pass 24:
NITPICK_ONLY, no further findings. The implementation in `src/analyzer/s7comm.rs` has
been frozen and correct since approximately pass 10; every finding from pass 10 onward
was a documentation, wording, or test-naming correction to this story spec or the test
suite, never a behavioral change to the shipped code. All items in this final trio
(passes 22-24) were fixed in this single post-convergence documentation/test-only batch
without further re-review, per the human ruling closing the review.

## Edge Cases

| ID | Source BC | Description | Expected Behavior |
|----|-----------|-------------|-------------------|
| EC-001 | BC-2.21.001 | Flow receives zero bytes before close | `S7commFlowState` never created |
| EC-002 | BC-2.21.002 | First DT frame classifies `Unclassified`, later frame on same flow carries `Some(0x32)` | `classified_protocol` remains `Unclassified` — sticky-first-classification applies uniformly; the later `0x32` frame is also NOT dissected (F-12 gate, see EC-009 below) |
| EC-003 | BC-2.21.004 | `data.len() == 9` (one byte short) | `None`; T0814 on first occurrence per direction |
| EC-004 | BC-2.21.005 | `data[0] == 0x72` reaching this function (caller-drift simulation) | `None`; no `Finding` — pure defensive hygiene, not a wire anomaly |
| EC-005 | BC-2.21.007 | `data[1] == 0x00` | `None`; shares dedup flag with EC-003 |
| EC-006 | BC-2.21.008 | `data[1] == 0x02`, `data.len() == 11` (one short of the 12-byte Ack minimum) | `None`; malformed-header T0814 |
| EC-012 | BC-2.21.008 EC-005 | `data[1] == 0x03` (Ack_Data), `data.len() == 10` (only the common header present) | `None`; malformed-header T0814 (shares dedup flag with EC-006) — Ack_Data is NOT a 10-byte-only ROSCTR, per the 2026-09-24 canonical-frame holdout ruling |
| EC-013 | BC-2.21.008 EC-006 | `data[1] == 0x03` (Ack_Data), `data.len() == 11` (one short of the 12-byte Ack_Data minimum) | `None`; malformed-header T0814 |
| EC-014 | BC-2.21.008 EC-007/EC-008 | `data[1] == 0x03` (Ack_Data), `data.len() >= 12`, with a non-empty parameter block following the error-class/code bytes (real-world shape, e.g. a Setup Communication response) | `Some(...)` with `header_len: 12`; error fields extracted from `data[10..12]`; the parameter block begins at `data[12]`, NOT `data[10]` (corrected 2026-09-24, canonical-frame holdout ruling DF-CANONICAL-FRAME-HOLDOUT-001) |
| EC-007 | BC-2.21.009 | `header.param_length == 0xFFFF`, `header.data_length == 0xFFFF`, `data.len() == 10` | Bounds check fails cleanly; no overflow in the sum; no slice attempted |
| EC-008 | BC-2.21.001 EC-004..EC-009 (plus the CR-only trivial case, no distinct BC EC number) | (a) CR-only, no CC ever following; (b) CC-only, no prior CR (mid-flow capture start, EC-004); (c) CC observed before any CR (EC-005); (d) CR direction A then CC same direction A (EC-006); (e) repeated CR, direction A twice, no intervening CC (EC-007, F-22); (f) `CR(A)`, then `CR(B)` (A != B), then `CC(A)` (EC-008); (g) `CR(A)`, then `CR(B)` (A != B), then `CC(B)` (EC-009) | `session_established` remains `false` in cases (a)-(e) (F-01 ruling); case (e) additionally asserts `cr_observed_dir == Some(A)` is unchanged by the repeated CR; case (f) sets `session_established` to `true` — `CC(A)` is opposite the most-recent CR's direction B, not the stale first CR's direction A (BC-2.21.001 EC-008); case (g) leaves `session_established` `false` — `CC(B)` matches the most-recent CR's direction B (same-direction, BC-2.21.001 EC-009) |
| EC-009 | BC-2.21.002 EC-005 | Flow sticky-classified `Plus` (first DT frame `Some(0x72)`), later DT frame on the same flow carries `Some(0x32)` | Sticky `classified_protocol` remains `Some(Plus)`; the later `0x32`-leading frame is NOT dissected — no `parse_s7comm_header` call, no finding (F-12 ruling) |
| EC-010 | BC-2.21.002 EC-004 | Flow's first DT frame has `protocol_id: None` (empty payload); a later DT frame has `protocol_id: Some(0x32)` | `classified_protocol` remains `None` after the first frame; the second frame sets it to `Some(Classic)` and is dissected (F-02 ruling) |
| EC-011 | BC-2.21.004 EC-001 (F-14 relabel) | `data.len() == 0`, direct call to `parse_s7comm_header` only | `None`; this input is NOT reachable via `on_data` dispatch in production (a `0x32`-leading DT frame always implies `data.len() >= 1`) — the corresponding unit test is explicitly direct-call-only, no production T0814 implied |

## Token Budget Estimate

| Context Source | Estimated Tokens |
|---------------|-----------------|
| This story spec | ~6,200 |
| BC-2.21.001-009 excl. 003 (8 BCs) | ~9,000 |
| ADR-014 (Decisions 1, 2, 4, 8, 9) | ~10,000 |
| src/analyzer/s7comm.rs (from STORY-186) | ~4,000 |
| Test file delta + new fixture generator | ~4,500 |
| **Total** | **~33,700** |
| Agent context window | 200K for Sonnet |
| **Budget usage** | **~17%** |

## Previous Story Intelligence

| Story | Key Decisions | Patterns Established | Gotchas Discovered |
|-------|--------------|---------------------|-------------------|
| STORY-186 | `S7commAnalyzer`/minimal `S7commFlowState` created; frame-walk loop proven; `on_flow_close` implemented | `S7commFlowState` grows incrementally across stories — do not assume the full field set exists until each field's owning story lands | The frame-walk loop's dispatch point (post `parse_cotp_header`) was a placeholder in STORY-186; this story is the first to give it real branching logic — do not regress the carry-buffer/resync behavior STORY-186 already proved |

The `on_flow_close` behavioral contract was deliberately packaged into STORY-186, not
this story, because flow-map lifecycle cohered better with the flow-map's creation than
with classification dispatch — see STORY-186's own BC table for the justification.

## Architecture Compliance Rules

Extracted from `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md`:
- **ADR-014 Decision 2**: the four-row disambiguation table (classic/plus/session/
  unclassified) lives entirely in `S7commAnalyzer`, never in SS-20. This story wires two
  of the four rows fully (session TPDUs, classic `0x32`); STORY-190 completes the
  remaining two (S7comm-plus, unclassified).
- **ADR-014 Decision 2 reconciliation note (2026-09-24, F-01/F-02/F-12, human-ratified)**:
  (a) `session_established` is set only by a CC observed in the direction OPPOSITE a
  previously-observed CR on the same flow — not by any CC regardless of CR history; (b) a
  DT-TPDU with `protocol_id: None` does NOT participate in sticky first-classification —
  `classified_protocol` remains unset until the first DT-TPDU carrying `Some(byte)`; (c)
  classic S7comm dissection is gated on the flow's *sticky* `classified_protocol ==
  Classic`, never merely on the current frame's raw `protocol_id` byte — a flow
  sticky-classified `Plus`/`Unclassified` must never fall into classic dissection even if
  a later individual frame carries `0x32`.
- **ADR-014 Decision 4**: S7comm classic field layout is derived from free-to-read
  prose/behavioral sources (Wireshark wiki prose, Kleinmann & Wool 2014,
  Orange-Cyberdefense catalog) and permitted open-source design references
  (cisagov/icsnpp-s7comm BSD-3, kprovost/libs7comm BSD-2, gijzelaerr/python-snap7
  MIT) — never from Wireshark's dissector source, Snap7 (the C++ library), or
  libnodave (all GPL/LGPL-tainted).
- **ADR-014 Decision 4 reconciliation note (2026-09-24, STORY-187 per-story
  adversarial pass 5, F-40, human ruling)**: this Decision's allowed-source list is
  amended, not rewritten — prose sources and permitted design references remain the
  only field-semantics sources. The reconciliation note additionally permits
  publicly posted **wire-capture byte examples** (cnblogs, Yiqisoft, and the
  Inductive Automation KB — the three sources cited in AC-187-013) as **test-vector
  sources only**, satisfying DF-CANONICAL-FRAME-HOLDOUT-001; this does not relax or
  extend the design/field-semantics source list, and the banned-code-source list
  (Wireshark dissector source, Snap7 (the C++ library), libnodave, and the AVOID-list
  crates) remains
  excluded in full.
- **ADR-014 Decision 9 item 3**: `parse_s7comm_header` is a pure-core free `fn` — the
  VP-051 Kani P0 target (CONFIRMED per the 2026-09-24 F-14 reconciliation note,
  superseding this item's original cargo-fuzz-only text), and part of the combined
  VP-055 fuzz chain. The BC-2.21.009 caller-side bounds check must be extracted as a
  pure, public (`pub fn`) helper so the Kani harness — which lives in the external
  `tests/s7comm_analyzer_tests.rs` integration-test binary — can call it directly
  (corrected from "crate-visible" pass-11, P11-F-1: `pub(crate)` would not be visible
  from an external test binary).
- **ADR-014 Decision 9 item 3 canonical-frame holdout correction (2026-09-24,
  DF-CANONICAL-FRAME-HOLDOUT-001, human-ratified; provenance corrected 2026-09-24,
  STORY-187 per-story adversarial pass 6, F-46, clean-room provenance; attestation
  scope corrected 2026-09-24, STORY-187 per-story adversarial pass 8, F-51)**:
  `header_len` is `12` for BOTH ROSCTR `0x02` (Ack) AND ROSCTR `0x03` (Ack_Data) — not
  Ack alone. The 12-byte Ack/Ack_Data header (1-byte `error_class` + 1-byte
  `error_code`) is grounded in permitted prose and design-reference sources per
  ADR-014 Decision 4, split by ROSCTR value. Kleinmann & Wool 2014 (§3.2, Fig. 2) is
  an ADR-014 Decision 4 permitted **prose** source that documents the S7comm error
  block as present "only for ROSCTR 3" — the authors' own captured traffic sample
  contained only ROSCTR 1 (Job) and ROSCTR 3 (Ack_Data) frames, so K&W attests the
  Ack_Data (`0x03`) 12-byte header exclusively and says nothing about the Ack (`0x02`)
  case. The Ack (`0x02`) 12-byte layout — and the 1-byte `error_class`/`error_code`
  split for both ROSCTR values — rests instead on the permitted ADR-014 Decision 4
  **design references** cisagov/icsnpp-s7comm (BSD-3; `ROSCTR_ACK`/`ROSCTR_ACK_Data`
  records each embedding an `S7Comm_Error { error_class, error_code }` field pair) and
  gijzelaerr/python-snap7 (MIT; `parse_response`'s ACK/ACK_DATA 12-byte header with
  error bytes at offsets 10/11), with kprovost/libs7comm (BSD-2) consistent in
  aggregate (message types 2 and 3 both carry 2 extra header bytes, modeled there as a
  single 16-bit "result" field rather than the two discrete `error_class`/`error_code`
  bytes) — the three design references are permissively licensed (BSD-3/MIT/BSD-2),
  distinct from the banned GPL/LGPL-tainted Wireshark dissector source, Snap7 (the C++
  library), and libnodave, and establish the field semantics per Decision 4's
  prose/design-source rule. The cnblogs, Yiqisoft, and Inductive Automation KB wire
  captures are the canonical-frame **test vectors** that surfaced and corroborate this
  defect — per ADR-014 Decision 4's reconciliation note (F-40), they are test-vector
  sources only and do NOT establish field semantics; the prior v1.0-v1.6 phrasing
  crediting these captures as one of "four independent real-world sources" for the
  field layout itself contradicted the Decision 4 bullet above (prose/design sources
  are the only field-semantics sources; wire captures are test vectors only) and was
  corrected by the F-46 pass; the v1.6-v1.9 phrasing then overstated K&W's attestation
  as if it grounded the whole Ack/Ack_Data header (and counted K&W among "all four"
  permissively-licensed sources, though an academic paper carries no software
  license) — corrected here (F-51) to scope K&W strictly to the Ack_Data (`0x03`)
  error-block attestation, matching BC-2.21.008 v1.7's F-50 correction. Every "10-byte
  Job/AckData/Userdata" phrasing in this story is superseded by "10-byte Job/Userdata;
  12-byte Ack/Ack_Data" — see BC-2.21.004 v1.2, BC-2.21.006 v1.1, BC-2.21.008 v1.2, and
  BC-2.21.009 v1.1 for the amended contracts.
- Pure/effectful boundary: `parse_s7comm_header` is pure; `on_data`'s dispatch extension
  is the effectful shell.

## Library & Framework Requirements

| Tool | Version | Purpose |
|------|---------|---------|
| Rust stdlib | 1.91+ (2024 edition) | `u16::from_be_bytes`, `Option`, match patterns |
| kani | Latest via `cargo kani` | VP-051 formal verification harness |
| proptest | 1 (pinned in `Cargo.toml`) | VP-053 dispatch-totality skeleton (partial) |
| Python 3.10+ | — | `tests/fixtures/mk_s7comm_pcap.py` generator (mirrors `mk_modbus_large_pcap.py`) |

## File Structure Requirements

| File | Action | Purpose |
|------|--------|---------|
| `src/analyzer/s7comm.rs` | MODIFY | Complete `S7commFlowState`; add `S7Protocol`, `Rosctr`, `S7commHeader`, `parse_s7comm_header`; extend `on_data`'s dispatch |
| `tests/s7comm_analyzer_tests.rs` | MODIFY | Add BC-2.21.001/002/004-009 unit tests + VP-051 Kani skeleton + VP-053 proptest skeleton |
| `tests/fixtures/mk_s7comm_pcap.py` | CREATE | Synthetic fixture generator per ADR-014 Decision 7 — Setup Communication + minimal classic-S7comm frames |
| `tests/fixtures/s7comm-setup-comm.pcap` | CREATE | Committed synthetic capture (output of `mk_s7comm_pcap.py`) carrying the CR/CC handshake plus Setup Communication Job/Ack_Data walk; exercised end-to-end by `test_BC_2_21_002_setup_comm_fixture_pcap_well_formed_no_findings` (AC-187-013) |
| `CHANGELOG.md` | MODIFY | `[Unreleased]` entry for the S7comm header parser and dispatch skeleton, per the CHANGELOG delivery task and the AC-158-001/`changelog-gate` obligation |
| `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md` | MODIFY | Dated reconciliation notes recording the STORY-187 human-ratified rulings (DF-CANONICAL-FRAME-HOLDOUT-001, F-01/F-02/F-12/F-40/F-46/F-51, etc.) referenced throughout this story's Architecture Compliance Rules |

## Forbidden Dependencies

- Wireshark, Snap7 (the C++ library), libnodave source, and any `s7`/`s7-comm`/`s7-client`
  crate — banned/
  avoid per ADR-014 Decision 4
- `parse_s7comm_header` MUST NOT read the `S7commAnalyzer`'s flow state or perform I/O —
  it remains a pure free fn per ADR-014 Decision 9

## Changelog

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.17 | 2026-09-25 | story-writer | STORY-187 per-story adversarial passes 22-24, review closure batch. **P22-F-2 (LOW)**: AC-187-011's fifth Given/When/Then described a generic "second, distinct TPKT frame" scenario that did not match its cited test, `test_BC_2_21_009_dissection_bounded_to_own_tpkt_frame` (confirmed via read of the test body, `tests/s7comm_analyzer_tests.rs` ~2481-2518 in the STORY-187 worktree): the test actually constructs a Job header (`classic_header_bytes(0x01, 0x0001, 0x0002, 0x0000)`, 10 bytes) declaring `param_length == 2` with zero parameter bytes present within its own TPKT-declared frame, immediately followed in the same delivery by a complete CR frame; asserts exactly one T0814 with evidence text containing "exceed available bytes" (bounds check computed against the DT frame's own 10-byte length only, never borrowing the CR frame's bytes), and separately asserts the trailing CR frame is still walked (`state.cr_observed_dir == Some(Direction::ClientToServer)`). Rewrote the AC to match. **P22-F-3 (NIT)**: AC-187-010 (~two "total frame length is exactly N bytes" bullets) and AC-187-011 (~two more) described these bounds/truncation test scenarios by "total frame length" when the referenced tests (`test_BC_2_21_008_truncated_ack_on_data_emits_t0814_once`, `test_BC_2_21_009_ack_header_len_12_bounds_check`) actually construct the S7comm payload — the slice ultimately passed to `parse_s7comm_header` — at the stated byte counts (via `classic_header_bytes`/`ack_header_bytes`) before wrapping it in TPKT/COTP framing via `dt_frame`, which adds further bytes on the wire; reworded all four occurrences to "S7comm payload length (the slice passed to `parse_s7comm_header`)". **P23-F-3 (NIT)**: the Tasks list's test name `test_BC_2_21_001_repeated_same_direction_cr_session_not_established` was wrapped mid-identifier across two lines; rejoined onto one line (full-story grep for other trailing-underscore line-wraps found none remaining). **P23-F-1 (NIT)**: AC-187-003's N-4 note updated to record that `test_BC_2_21_001_session_established_is_monotonic` now also directly asserts `cr_observed_dir` at each step — retained at `Some(ClientToServer)` unchanged after sequence (a)'s establishing CC (the matching `ConnectConfirm` arm only ever reads `cr_observed_dir`, never clears it), and overwritten to `Some(ServerToClient)` by sequence (b)'s further CR even though `session_established` is already `true` (the `ConnectRequest` arm unconditionally overwrites `cr_observed_dir`) — both per BC-2.21.001 postcondition 1 (confirmed via read of the test body, ~5027-5109). **Convergence decision recorded (Tasks section, new closing paragraph)**: per-story adversarial review for STORY-187 is closed per human ruling (2026-09-25) after passes 22-24 — pass 22: LOW/NIT wording only (P22-F-2, P22-F-3, both fixed in this burst); pass 23: one LOW test-adequacy finding already fixed post-convergence with a mutation-kill proof (commit b4fce34a and prior bursts), plus NITs P23-F-1 and P23-F-3 (both fixed in this burst); pass 24: NITPICK_ONLY, no further findings; 24 passes total. Code in `src/analyzer/s7comm.rs` has been frozen and correct since approximately pass 10 — every finding from pass 10 onward was a documentation/wording/test-naming correction, never a behavioral code change; all final-trio items (passes 22-24) were fixed in this single post-convergence documentation/test-only batch without further re-review. No AC semantics beyond the wording corrections above changed; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged; every claim above that a test asserts something was confirmed by read/grep against `tests/s7comm_analyzer_tests.rs` in the STORY-187 worktree before being written. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4-v1.16 and remains stale pending a state-manager hash-refresh burst. |
| 1.16 | 2026-09-25 | story-writer | STORY-187 per-story adversarial passes 19-21 final wording batch. **P20-F-1 (LOW)**: AC-187-006's timestamp note said "exactly two tests in this story" for the shared-helper timestamp assertion, which is true only per-AC (four story-wide across AC-187-006/009/011); reworded to "exactly two tests under this AC (four story-wide; see AC-187-009/011 notes)". **P19-F-2 (LOW)**: AC-187-009's note called `test_BC_2_21_007_unrecognized_rosctr_returns_none` "the c2s variant" — confirmed via grep it is a direct `parse_s7comm_header` call with no `on_data`, no `S7commAnalyzer`, and no `Finding` at all, so "c2s" does not apply; reworded. AC-187-009's "verified independently for BOTH c2s and s2c" bullet was previously unbacked by an explicit c2s test citation; added `test_BC_2_21_007_shares_dedup_flag_with_004_malformed_header` (confirmed via grep: a too-short frame sets the c2s dedup flag, then unrecognized-ROSCTR on c2s is suppressed) to the bullet and the Tests list. AC-187-011's per-direction dedup bullet updated to state that `test_BC_2_21_009_bounds_check_before_parameter_data_slice` (commit b4fce34a) now repeats a c2s delivery of the same malformed frame and asserts `findings.len() == 1` (confirmed via grep). **New test (commit b4fce34a)**: registered `test_BC_2_21_008_canonical_ack_vector_verbatim` against AC-187-010 (parses the BC-2.21.008 canonical Ack row `32 02 00 00 00 01 00 00 00 00 00 00` verbatim, asserting `error_class`/`error_code == Some(0)`), and noted the two 12-byte happy-path tests instead use the canonical-vector SHAPE with distinct, non-zero error bytes 0x81/0x04 substituted (P12-F-2), not the verbatim canonical bytes. Noted (confirmed via grep) that `test_BC_2_21_007_unrecognized_rosctr_emits_t0814_once_s2c` now asserts the exact evidence text "unrecognized ROSCTR byte 0x00" (P20-F-2), and that the shared `assert_reason_specific_evidence` helper (confirmed via grep of its body) now also asserts `finding.summary` is non-empty and contains the same reason text as `finding.evidence` verbatim (P20-F-3). **N-2**: AC-187-010's EC-007 trace bullet marked as covering only the header/error-field extraction half of the canonical shape — the Group-3 function-code classification half of the trailing parameter block at `data[12]` is explicitly out of this story's scope and belongs to STORY-188. **N-4**: AC-187-003's monotonic bullet now states the exact two sequences the test exercises (confirmed via grep): `CR(c2s)`, `CC(s2c)`, `CC(c2s)` and `CR(c2s)`, `CC(s2c)`, `CR(s2c)`. AC-187-008's Reserved-bytes bullet (BC-2.21.006 postcondition 5) now cites `test_BC_2_21_006_nonzero_reserved_bytes_do_not_reject` (confirmed via grep of its body). **Accepted residuals recorded from the pass-19-21 final trio**: **P21-F-1 (deferred, LOW)** — ADR-014's "explicit unwind" placeholder-branch wording does not precisely match this story's actual no-op placeholder implementation; wording reconciliation deferred to STORY-194 rather than corrected here. **D-P21-1 (accepted residual, NIT)** — `on_data`'s carry-buffer re-copy-and-rewalk pattern (pre-existing STORY-186 code, not introduced by this story) is worst-case quadratic in bytes carried across many small deliveries; a CPU-amplification concern, not a correctness/bounds-safety one; accepted as-is, flagged as a phase-5 performance research item requiring magnitude validation under realistic small-segment traffic. **N-3 (accepted residual, NIT)** — BC-2.21.006 EC-002 (`pdu_reference == 0x0000`) and BC-2.21.009 EC-003 (large, plausible `data_length`) have no dedicated example-based unit test in this file (confirmed via grep: the sole `pdu_reference: 0` occurrence is incidental to an unrelated bounds-isolation test; no large-`data_length` test exists); both are covered only symbolically by VP-051's bounded Kani harnesses, accepted as sufficient. No AC semantics beyond the wording/citation corrections above changed; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged; every claim above that a test asserts something was confirmed by grep against `tests/s7comm_analyzer_tests.rs` at commit b4fce34a in the STORY-187 worktree before being written. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4-v1.15 and remains stale pending a state-manager hash-refresh burst. |
| 1.15 | 2026-09-25 | story-writer | STORY-187 per-story adversarial passes 16-18 (P16-F-1/2/3; P17 deferrals). **P16-F-1 (MEDIUM, corrected mid-burst against commit f8ead399)**: the AC-187-006/009/011 notes falsely claimed all of each AC's `on_data`-driven tests "separately assert `finding.timestamp` … against a non-zero `ts` value." Reworded all three notes to state exactly what is asserted, verified by grep against `tests/s7comm_analyzer_tests.rs` in the STORY-187 worktree: the shared `assert_malformed_header_t0814` helper pins `category`/`verdict`/`confidence`/`mitre_techniques == vec!["T0814".to_string()]` (exact)/`direction` and does NOT assert `timestamp`; timestamp is asserted directly by `test_BC_2_21_004_len_shorter_than_10_returns_none_and_emits_t0814_once` and `test_BC_2_21_004_len_shorter_than_10_emits_t0814_once_s2c` (AC-187-006, two tests), `test_BC_2_21_007_unrecognized_rosctr_emits_t0814_once_s2c` (AC-187-009, one test — its c2s sibling `test_BC_2_21_007_unrecognized_rosctr_returns_none` does not), and `test_BC_2_21_009_bounds_check_dedup_s2c` (AC-187-011, one test) — this list reflects commit f8ead399 ("pass-16 touch-ups"), which added the AC-187-009/011 timestamp assertions and the "exceed available bytes" evidence assertion in `test_BC_2_21_009_ack_header_len_12_bounds_check` after this burst's initial grep; re-grepped and corrected per the coordinator's mid-burst update. Sufficiency rationale: all five malformed-header reason classes share the single `report_malformed_header` emission path (`src/analyzer/s7comm.rs:849`), which performs the identical timestamp conversion regardless of reason, so exercising it directly in AC-187-006 (plus once each in AC-187-009/011) is sufficient. **P16-F-2 (LOW)**: AC-187-003 case (e)'s bullet still quoted BC-2.21.001 EC-007's withdrawn "(or first-observed value)" permission wording; BC-2.21.001 v1.4+ EC-007 is deterministic most-recent-CR-wins overwrite, not a discretionary choice — reworded to match (and to match the wording already used by the corresponding test's own doc comment, `test_BC_2_21_001_repeated_same_direction_cr_session_not_established`). **P16-F-3 (LOW)**: the VP-051 Kani Obligation section named only `verify_parse_s7comm_header_bounds_safety` and described the F-18 bounds-check half as part of that same harness; confirmed via grep of `tests/s7comm_analyzer_tests.rs` that the bounds half is a second, separate `#[kani::proof]` fn, `verify_s7comm_bounds_ok_bounds_safety` — registered both harness names in a new "Harnesses" block, renamed the "Bounds-check half of the harness (F-18)" heading to name the second harness explicitly, and noted STORY-194 must execute both (their `kani::cover!` non-vacuity obligations are independent). **P17 deferrals (Tasks section, end)**: **P17-F-1 (deferred)** — `protocol_id_strategy`'s `prop_oneof![Just(None), any::<u8>().prop_map(Some)]` does not explicitly weight `Some(0x32)`/`Some(0x72)` generation (each is 1-of-256 within the `Some` arm); deferred to STORY-190/STORY-194's full non-vacuous VP-053 run. **P17-F-2 (NIT, accepted residual)** — the truncated-Ack `on_data` test asserts only the "truncated Ack header" evidence substring, not the exact 10-vs-11 byte count (unlike the too-short-header and bounds-check-failure evidence tests); accepted as-is, not blocking. **Spot-check (task item 5)**: grepped every "assert(s)"-bearing sentence in the story body and verified ≥5 against `tests/s7comm_analyzer_tests.rs` test bodies: (1) the `assert_malformed_header_t0814`/`assert_reason_specific_evidence` helper bodies (lines ~1506-1553 of the test file); (2) `test_BC_2_21_001_repeated_same_direction_cr_session_not_established` (AC-187-003 case (e)'s `cr_observed_dir == Some(A)` claim); (3) `test_BC_2_21_002_setup_comm_fixture_pcap_well_formed_no_findings` (AC-187-013's "asserts zero findings, session_established, and classified_protocol == Some(Classic)" claim); (4) `verify_parse_s7comm_header_bounds_safety` (the VP-051 positive-assertion bullets); (5) `test_BC_2_21_002_classic_s7comm_dispatch_asserts_classified_protocol_classic` (AC-187-004's dispatch claim); all matched with no overclaim found beyond the P16-F-1 issue already corrected above. No AC semantics beyond the corrections above changed; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4-v1.14 and remains stale pending a state-manager hash-refresh burst. |
| 1.14 | 2026-09-25 | story-writer | STORY-187 per-story adversarial passes 13-15 (P13-F-2, test sync, P15-F-1 accepted). **P13-F-2**: the Edge Cases table's EC-008 row's Source BC column cited only "BC-2.21.001 EC-004..EC-007," omitting the two new "most recent CR direction wins" sub-cases (EC-008: `CR(A)`, `CR(B)`, `CC(A)` -> `session_established` becomes `true`; EC-009: `CR(A)`, `CR(B)`, `CC(B)` -> remains `false`) that AC-187-003's pass-11 `test_BC_2_21_001_most_recent_cr_direction_wins` addition already exercises; extended the Source BC reference to "EC-004..EC-009," added sub-cases (f)/(g) to the Description column, and updated the Expected Behavior column to state the established/not-established outcomes for both new sub-cases. Cross-cited BC-2.21.001 EC-008/EC-009 from AC-187-003's "most recent CR direction wins" bullet. Added a new monotonic-latch property to AC-187-003 (`session_established` never reverts to `false` once set) backed by the new test `test_BC_2_21_001_session_established_is_monotonic`. **Test sync (DF-AC-TEST-NAME-SYNC-001, commit 38ff7ee1)**: registered five new test-writer test names against their ACs, each with a one-line scenario: `test_BC_2_21_002_non_0x32_dt_frame_on_classic_flow_not_dissected` (AC-187-012 — the sticky-Classic dissection gate is a conjunction of the frame's own `protocol_id == Some(0x32)` AND the flow's sticky `classified_protocol == Classic`; a non-0x32 frame on an already-Classic flow is not dissected); `test_BC_2_21_002_unparseable_cotp_does_not_classify` (AC-187-005 — a COTP frame `parse_cotp_header` cannot parse at all never classifies, sets `session_established`, or populates `cr_observed_dir`); `test_BC_2_21_001_session_established_is_monotonic` (AC-187-003, see above); `test_BC_2_21_004_nine_byte_payload_on_data_too_short_evidence` (AC-187-006 — the exact 9-byte boundary via the `on_data` dispatch path, asserting evidence text "header too short: 9"); `test_BC_2_21_008_ack_data_nonzero_error_fields_with_parameter_block` (AC-187-010 — a 12-byte Ack_Data header with non-zero, distinct `error_class`/`error_code` plus a non-empty trailing parameter block, traces to BC-2.21.008 EC-007). Added cross-reference notes to AC-187-006/009/011 documenting that the shared `assert_malformed_header_t0814` test helper asserts `finding.mitre_techniques == vec!["T0814".to_string()]` exactly (not merely containment) and that the individual `on_data`-driven tests separately assert `finding.timestamp` against a non-zero `ts` value. Verified via `grep -c` against `tests/s7comm_analyzer_tests.rs` in the STORY-187 worktree that all five new test names exist exactly once each. **P15-F-1 (accepted residual)**: recorded, as a new note at the end of the Tasks section, the pass-15 finding that `dispatch_classic_s7comm` builds the malformed-header evidence string before the per-direction dedup check runs, so a suppressed duplicate malformed frame still allocates/formats the (discarded) evidence string — bounded, no correctness impact; accepted as-is, flagged as a candidate micro-optimization for a later story rather than blocking this one. No AC semantics beyond the additions above changed; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4-v1.13 and remains stale pending a state-manager hash-refresh burst. |
| 1.13 | 2026-09-25 | story-writer | Registered two new test-writer tests (commit 6705ed8b) that close cargo-mutants survivors at `s7comm.rs:779` (the BC-2.21.009 bounds-check-failure `Finding`'s evidence-text arithmetic): `test_BC_2_21_009_bounds_failure_evidence_reports_declared_and_available` (Job header, declared 18 = `header_len=10 + param_length=3 + data_length=5`, available 12) and `test_BC_2_21_009_bounds_failure_evidence_ack_data_header_len_12` (Ack_Data header, declared 20 = `header_len=12 + param_length=3 + data_length=5`, available 14) — both added to AC-187-011's Tests list with a one-line scenario each. Recorded the scoped mutation-testing result as a new note at the end of the Tasks section: `cargo mutants` 27.1.0, serial (`--jobs 1`), over the STORY-187 diff of `src/analyzer/s7comm.rs` — 43 mutants, 37 caught initially, 3 real survivors (the evidence arithmetic above, now killed), 1 equivalent mutant (the empty `Some(0x72)` placeholder arm in `dispatch_cotp_frame`, killable once STORY-190 fills it in), 2 unviable. Verified both new test names exist exactly once in `tests/s7comm_analyzer_tests.rs`. No AC semantics beyond the two test-list additions changed; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4-v1.12 and remains stale pending a state-manager hash-refresh burst. |
| 1.12 | 2026-09-24 | story-writer | STORY-187 per-story adversarial pass 11/12 (FSR completeness, mutation-discriminating tests, P11-F-1). **FSR completeness (DF-MERGE-AUTH-CLASSIFIER-001 condition 3)**: the File Structure Requirements table listed only `src/analyzer/s7comm.rs`, `tests/s7comm_analyzer_tests.rs`, and `tests/fixtures/mk_s7comm_pcap.py`, omitting three files the branch actually touches per `git diff --stat 47951b7a..HEAD`; added rows for `tests/fixtures/s7comm-setup-comm.pcap` (CREATE — the committed fixture exercised by `test_BC_2_21_002_setup_comm_fixture_pcap_well_formed_no_findings`), `CHANGELOG.md` (MODIFY — the `[Unreleased]` entry required by the changelog-gate), and `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md` (MODIFY — the dated reconciliation notes recording this story's human rulings). **Mutation-discriminating test additions (DF-AC-TEST-NAME-SYNC-001)**, four new test-writer test names registered against their ACs: `test_BC_2_21_009_data_length_overrun_on_data_emits_t0814` and `test_BC_2_21_009_dissection_bounded_to_own_tpkt_frame` added to AC-187-011 (new Given/When/Then scenarios isolating the `data_length` term of the bounds sum from `param_length`, and confirming dissection never reads past a frame's own TPKT-declared length into a following frame in the same delivery); `test_BC_2_21_002_empty_dt_followed_by_frame_same_delivery_stays_unclassified` added to AC-187-005 (a delivery with zero classifying frames leaves `classified_protocol` at `None`); `test_BC_2_21_001_most_recent_cr_direction_wins` added to AC-187-003 with new "most recent CR direction wins" semantics — `CR(A)`, `CR(B)`, `CC(A)` establishes the session (CC opposite the *most recent* CR, B, not the stale first CR, A), while `CR(A)`, `CR(B)`, `CC(B)` does not (CC same-direction as the most recent CR). Also added a mutation-discriminating note to AC-187-010 requiring the `error_class`/`error_code` extraction tests to use distinct, non-zero byte values (not a shared value, not `0x00`/`0x00`) so a field-swap or hard-coded-constant mutant cannot pass undetected. **P11-F-1**: every place this story described `s7comm_bounds_ok` as a "crate-visible" helper (VP-051 Kani Obligation section, the corresponding Tasks bullet, and the Architecture Compliance Rules "ADR-014 Decision 9 item 3" bullet) is corrected to "public (`pub fn`)" — the VP-051 Kani harness and the AC-187-011 bounds tests both live in `tests/s7comm_analyzer_tests.rs`, an external integration-test binary outside the crate, so `pub(crate)` visibility would not actually be callable from there; only `pub` suffices. No AC semantics beyond the additions above changed; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4-v1.11 and remains stale pending a state-manager hash-refresh burst. |
| 1.11 | 2026-09-24 | story-writer | STORY-187 per-story adversarial pass 9 (N-3): NIT — the Architecture Compliance Rules "ADR-014 Decision 4 reconciliation note" bullet's banned-code-source parenthetical (~744, "Wireshark dissector source, Snap7, libnodave, and the AVOID-list crates") and the Forbidden Dependencies section's opening bullet (~811, "Wireshark, Snap7, libnodave source, and any `s7`/`s7-comm`/`s7-client` crate") both used bare "Snap7," which — unlike every other current-state occurrence in this story — did not disambiguate the banned C++ library from the permitted gijzelaerr/python-snap7 (MIT) design reference cited elsewhere in the same Architecture Compliance Rules section (e.g. ~734, ~772-773). Full-story grep for `Snap7` performed: qualified both bare occurrences to "Snap7 (the C++ library)," matching the disambiguation already used at ~734 and ~772-773; no other bare "Snap7" remained in current-state text (the changelog's historical v1.9 row already quotes "Snap7 (the C++ library)" verbatim and was left untouched per changelog-preservation convention). No AC semantics changed; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4-v1.10 and remains stale pending a state-manager hash-refresh burst. |
| 1.10 | 2026-09-24 | story-writer | STORY-187 per-story adversarial pass 8 (F-51): corrected the Architecture Compliance Rules "ADR-014 Decision 9 item 3 canonical-frame holdout correction" / Ack_Data bullet (~751-782) to fix a source-attestation overreach that survived v1.6-v1.9. The bullet previously said the 12-byte Ack/Ack_Data header "derives from Kleinmann & Wool 2014 (§3.2, Fig. 2, Ack_Data) — an ADR-014 Decision 4 permitted prose/design source — corroborated by … icsnpp-s7comm … python-snap7 … kprovost/libs7comm … all four are permissively licensed (BSD-3/MIT/BSD-2)." This misattributed K&W as jointly grounding both ROSCTR values and miscategorized K&W (an academic paper, not licensed software) as one of "four" permissively-licensed sources alongside the three design references. Rewrote the bullet to match ADR-014 Decision 9's canonical-frame holdout reconciliation note and BC-2.21.008 v1.7's F-50 correction exactly: Kleinmann & Wool 2014 is an ADR-014 Decision 4 **prose** source that attests only the Ack_Data (`0x03`) 12-byte error-block layout — the authors' own captured traffic sample contained only ROSCTR 1 (Job) and ROSCTR 3 (Ack_Data) frames, so K&W says nothing about Ack (`0x02`); the Ack (`0x02`) 12-byte layout and the 1-byte `error_class`/`error_code` split for both ROSCTR values rest instead on the permitted ADR-014 Decision 4 **design references** cisagov/icsnpp-s7comm (BSD-3) and gijzelaerr/python-snap7 (MIT), with kprovost/libs7comm (BSD-2) consistent in aggregate (same 2 extra header bytes, modeled as a single 16-bit "result" field rather than the discrete class/code bytes); reworded the licensing clause to "the three design references are permissively licensed (BSD-3/MIT/BSD-2)" (K&W excluded, since it carries no software license); the cnblogs/Yiqisoft/Inductive Automation KB wire captures remain scoped as test-vector-only sources, unchanged. Full-story grep sweep for `Kleinmann`, `design source`, `design reference`, `prose source`, and `permissively licensed` performed; every other hit (AC-187-013's reconciliation-note sentence ~216-219, the Architecture Compliance Rules "ADR-014 Decision 4" bullet ~730-735, and its "reconciliation note" companion bullet ~736-745) already states the general Decision 4 prose-sources-plus-design-references framing correctly and does not make the specific Ack/Ack_Data per-ROSCTR attestation-scope claim this finding concerns — no further changes needed there; remaining matches are historical changelog rows (v1.6, v1.8, v1.9), left untouched per changelog-preservation convention. No AC semantics changed; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged; this is a provenance/citation correction confined to the single Architecture Compliance Rules bullet. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4-v1.9 and remains stale pending a state-manager hash-refresh burst. |
| 1.9 | 2026-09-24 | story-writer | STORY-187 per-story adversarial pass 7 (F-48, F-49). **F-48** — corrected residual field-semantics-provenance drift that survived v1.8's F-46 fix: (1) AC-187-013's reconciliation-note sentence (~215-220) said parser design/field semantics "continue to derive solely from Decision 4's original prose sources (Wireshark wiki, Kleinmann & Wool 2014, Orange-Cyberdefense catalog)" — corrected to "derive from Decision 4's prose sources ... and permitted design references (cisagov/icsnpp-s7comm BSD-3, kprovost/libs7comm BSD-2, gijzelaerr/python-snap7 MIT)," matching the ground truth that field semantics rest on BOTH ADR-014 Decision 4's prose sources AND its permitted design references, not prose alone. (2) The Architecture Compliance Rules "ADR-014 Decision 4 reconciliation note" bullet (~734-736) said "the three prose sources above remain the only sources for parser design and field semantics" — corrected to "prose sources and permitted design references remain the only field-semantics sources." (3) The original "ADR-014 Decision 4" bullet itself (~730-734), which v1.8 did not touch, still said the field layout derives from prose/behavioral sources "only" and omitted the permitted design references entirely — this directly contradicted the reconciliation bullet immediately below it and the already-corrected F-46 bullet further down (~749-762, which already grounds the Ack/Ack_Data layout in prose + design references); corrected to name the three permitted design references alongside the prose sources, and clarified "Snap7 (the C++ library)" as the banned target (distinct from the permitted python-snap7 MIT reference), consistent with the disambiguation already used at ~761. (4) AC-187-013's Ack_Data corroborator list (~228-229) and the matching Tasks bullet (~641-642) wrongly counted Kleinmann & Wool 2014 as one of the canonical-frame **test-vector** corroborators ("corroborated by three further independent sources ... Yiqisoft 2023-03-22, the Inductive Automation KB, and Kleinmann & Wool 2014") — K&W is a prose/field-semantics source per Decision 4, never a test-vector source (test-vector sources are cnblogs (primary), Yiqisoft, and the Inductive Automation KB only, per ADR-014 Decision 4's F-40 reconciliation note and BC-2.21.008's own Description); corrected both to "corroborated by two further independent sources ... Yiqisoft 2023-03-22 and the Inductive Automation KB" / "corroborated by Yiqisoft and the Inductive Automation KB." **Correction to the v1.8 changelog row**: v1.8 stated that a grep sweep found AC-187-013's narrative "already correctly scope[d] cnblogs/Yiqisoft/Inductive Automation KB as test-vector-only sources ... no further changes needed there" — this was incorrect; AC-187-013 still carried both the "solely from ... prose sources" omission of design references (item 1 above) and the Kleinmann-&-Wool-as-test-vector-corroborator miscount (item 4 above), neither of which v1.8's sweep caught. **F-49** — the VP-051 Kani Obligation section's "Covers BC-2.21.004 ..., BC-2.21.008 ..., and BC-2.21.009 ..." sentence and its "VP-051's source BCs are {BC-2.21.004, BC-2.21.008, BC-2.21.009}" citation (~455-459) were stale against the architect's parallel registration of BC-2.21.006 (10-byte Job/Userdata common-header extraction) and BC-2.21.007 (unrecognized-ROSCTR safe-reject) into VP-051's source-BC set; corrected the sentence to cover all five BCs and restated the source-BC set as {BC-2.21.004, BC-2.21.006, BC-2.21.007, BC-2.21.008, BC-2.21.009}, noting the architect's BC-2.21.006/007 registration. Also corrected the VP-051 positive-field-extraction bullet (~497-499, big-endian `pdu_reference`/`param_length`/`data_length` reads) from "(BC-2.21.006 postcondition 1)" to "(BC-2.21.006 postconditions 2-4)" — postcondition 1 is the 10-byte `header_len` selection for Job/Userdata (correctly cited elsewhere, e.g. AC-187-008's trace line), not the big-endian field-extraction postconditions; left AC-187-008's existing "(traces to BC-2.21.006 postcondition 1)" citation unchanged since it correctly cites the `header_len == 10` clause. Full-story grep sweep for `solely\|only from\|only source\|original prose\|Kleinmann\|prose source\|four independent\|four sources` performed; all current-state hits resolved (see final grep output in the delivering agent's report) — remaining matches are either accurate post-fix wording (citing Kleinmann & Wool as a legitimate prose source) or historical changelog rows (v1.3/v1.6/v1.8, left untouched per changelog-preservation convention). No AC semantics changed beyond wording/citation corrections; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4-v1.8 and remains stale pending a state-manager hash-refresh burst. |
| 1.8 | 2026-09-24 | story-writer | STORY-187 per-story adversarial pass 6 (F-46, clean-room provenance): the Architecture Compliance Rules "ADR-014 Decision 9 item 3 canonical-frame holdout correction" / Ack_Data bullet (~745-765) attributed the 12-byte Ack/Ack_Data header layout to "Four independent real-world sources (cnblogs, Yiqisoft, Inductive Automation KB, Kleinmann & Wool 2014)," contradicting the Decision 4 bullet immediately above it (prose/design sources are the only field-semantics sources; wire captures are test vectors only, per ADR-014 Decision 4's F-40 reconciliation note). Rewrote the bullet: the 12-byte Ack (`0x02`)/Ack_Data (`0x03`) header with 1-byte `error_class` + 1-byte `error_code` now derives from Kleinmann & Wool 2014 (§3.2 Fig. 2, Ack_Data — an existing Decision 4 permitted prose source) corroborated by the permitted ADR-014 Decision 4 design references cisagov/icsnpp-s7comm (BSD-3; `ROSCTR_ACK`/`ROSCTR_ACK_Data` records with an `S7Comm_Error { error_class, error_code }` field pair) and python-snap7 (MIT; `parse_response`'s ACK/ACK_DATA 12-byte header with error bytes at offsets 10/11), with kprovost/libs7comm (BSD-2) consistent with the same layout; the cnblogs/Yiqisoft/Inductive Automation KB wire captures are now stated explicitly as the canonical-frame test vectors that surfaced and corroborate the defect (test-vector use only), not field-semantics sources. Grepped the full story for other wire-capture-as-field-semantics attributions: AC-187-013's narrative (~206-227), AC-187-010's narrative (~328-372), and the EC-012/EC-013/EC-014 Edge Cases rows (~678-680) already correctly scope cnblogs/Yiqisoft/Inductive Automation KB as test-vector-only sources (per the F-40 reconciliation note already applied in v1.6) — no further changes needed there. No AC semantics changed; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged; this is a provenance-citation correction confined to the single Architecture Compliance Rules bullet. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4/v1.5/v1.6/v1.7 and remains stale pending a state-manager hash-refresh burst. |
| 1.7 | 2026-09-24 | story-writer | STORY-187 per-story adversarial pass 6 (F-45, F-47, N-2, N-3): **F-45** — the VP-051 Kani Obligation section's opening "Covers BC-2.21.004 ... and BC-2.21.009" sentence omitted BC-2.21.008 from its BC coverage statement despite BC-2.21.008 already being listed in `behavioral_contracts:` frontmatter and the harness's own non-vacuity bullets already covering Ack/Ack_Data error-field extraction; corrected to "Covers BC-2.21.004 ..., BC-2.21.008 ..., and BC-2.21.009 ..." with an explicit note that VP-051's source BCs are {BC-2.21.004, BC-2.21.008, BC-2.21.009} per VP-INDEX.md. **F-47** — the VP-051 non-vacuous-assertion list stated only negative/implication-style obligations (length thresholds, `header_len` membership, `error_class.is_some()`); added four positive field-extraction assertions the harness now also makes: `header_len == 12` iff `rosctr ∈ {Ack, AckData}` else `10`; `error_class == Some(data[10])` and `error_code == Some(data[11])` for Ack/AckData (exact byte values, not just presence); `rosctr` derived from `data[1]`; and `pdu_reference`/`param_length`/`data_length` equal the big-endian `u16` reads of `data[4..6]`/`[6..8]`/`[8..10]` (BC-2.21.006 postcondition 1). **N-2** — the VP-053 Proptest Obligation section split the STORY-190/STORY-194 deferral note across two separate sentences and mischaracterized the skeleton as "compiles here against a `todo!()`/placeholder stub"; consolidated both deferrals (STORY-190 completes the `Some(0x72)`/unclassified branches; STORY-194 runs the full non-vacuous VP-053 totality proof) into one sentence, and corrected the stub language — the not-yet-implemented dispatch branches are `todo!()`-free structural no-ops (compiling cleanly under `tdd_mode: strict`), and this story's proptest skeleton runs under `cargo test`, not `cargo kani`. **N-3** (optional clarity) — added a note under AC-187-011's Tests list: `test_BC_2_21_009_ack_header_len_12_bounds_check` exercises the Ack (`0x02`) case only, and Ack_Data (`0x03`) shares the identical `header_len == 12` bounds-check code path, exercised live by `test_BC_2_21_008_canonical_ack_data_setup_communication_response_on_data` (AC-187-013) rather than by symmetry argument alone. No AC semantics changed beyond the added positive assertions and clarifying note; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged (BC-2.21.008 was already present — this pass fixed only the VP-051 section's prose omission of it). The Architecture Compliance Rules "Decision 9"/Ack_Data provenance bullet (~717-726) was intentionally NOT touched — a separate provenance fix (F-46) is pending research. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4/v1.5/v1.6 and remains stale pending a state-manager hash-refresh burst. |
| 1.6 | 2026-09-24 | story-writer | STORY-187 per-story adversarial pass 5 (F-40): reconciled AC-187-013 (~206-212), the matching Tasks bullet (~598), and the Architecture Compliance Rules section against ADR-014 Decision 4's new 2026-09-24 reconciliation note (STORY-187 per-story adversarial pass 5, F-40, human ruling), read from `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md`. The reconciliation note permits publicly posted wire-capture byte examples (cnblogs, Yiqisoft, Inductive Automation KB) as **test-vector sources only**, satisfying DF-CANONICAL-FRAME-HOLDOUT-001; it does not relax Decision 4's original prose-only source list (Wireshark wiki, Kleinmann & Wool 2014, Orange-Cyberdefense catalog) for parser design/field semantics, and the banned-code-source list is unchanged. Previously, AC-187-013 and the Tasks bullet mischaracterized these three wire-capture sources as "per ADR-014 Decision 4 allowed sources" (implying they were Decision 4 design sources) — both are reworded to cite the reconciliation note and its test-vector-only scoping instead. The Architecture Compliance Rules section's existing "ADR-014 Decision 4" bullet (which restates only the original three-prose-source list) gained a new companion bullet stating the reconciliation note's additional test-vector-only permission by name, so the section no longer contradicts AC-187-013. No AC semantics, BC traceability, or `behavioral_contracts:` frontmatter changed — this is a source-citation wording correction only. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated v1.4/v1.5 and remains stale pending a state-manager hash-refresh burst. |
| 1.5 | 2026-09-24 | story-writer | STORY-187 per-story adversarial pass 4 (F-34): AC-187-006's "reason-specific evidence text" bullet miscounted the malformed-header reason classes sharing the per-direction `malformed_header_reported_c2s`/`_s2c` dedup flag as "three" when there are five — header too short (BC-2.21.004, AC-187-006), unrecognized ROSCTR (BC-2.21.007, AC-187-009), truncated Ack (BC-2.21.008, AC-187-010), truncated Ack_Data (BC-2.21.008, AC-187-010), and declared lengths exceed available/bounds-check failure (BC-2.21.009, AC-187-011); corrected "three" to "five" and expanded the cross-reference list to name all four other reasons (previously omitted AC-187-010's truncated-Ack/Ack_Data pair). AC-187-009 and AC-187-011's matching "distinct from" bullets had the same omission (each cross-referenced only the other two ACs, silently excluding AC-187-010) and are corrected identically, each now stating it is "one of the five malformed-header conditions" sharing the dedup flag. No AC semantics changed — this is a wording/count correction only; no BC H1 title changed; `behavioral_contracts:` frontmatter unchanged. Note: `input-hash` intentionally left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value already predated this edit (per the v1.4 note) and remains stale pending a state-manager hash-refresh burst. |
| 1.4 | 2026-09-24 | story-writer | Folded STORY-187 per-story adversarial pass 3 (F-25, F-28, F-30) findings into this story: **F-28** — AC-187-013's stale "a research agent is sourcing the canonical vector now" placeholder is replaced with the completed primary citation (cnblogs, "西门子S7通讯协议引用整理", https://www.cnblogs.com/crcce-dncs/p/10659087.html — Setup Communication request/response — corroborated by Yiqisoft and the Inductive Automation KB); the matching Tasks bullet for the Job-frame canonical test was checked for the same stale wording (none found needing separate correction beyond AC-187-013's own text). **F-25** — AC-187-013 gained a new fixture-exercise sub-case and test `test_BC_2_21_002_setup_comm_fixture_pcap_well_formed_no_findings` (feeds the committed `tests/fixtures/s7comm-setup-comm.pcap` through `S7commAnalyzer::on_data` packet-by-packet, asserts zero findings, `session_established`, and `classified_protocol == Some(Classic)`); the fixture-generator Task bullet now states the generator's Ack_Data PDUs emit the corrected 12-byte header (pass-3 F-25) and that the resulting committed pcap is exercised end-to-end by that test, satisfying DF-AC-TEST-NAME-SYNC-001. **Test-name sync audit** — cross-checked every test name this story cites against `tests/s7comm_analyzer_tests.rs`'s `mod story_187`: `test_BC_2_21_002_classic_s7comm_dispatch_asserts_classified_protocol_classic` was already correctly named (no change needed); AC-187-010 was missing a citation for the already-implemented `test_BC_2_21_008_error_fields_none_for_job_and_userdata_rosctr` (covering postcondition 3's Job/Userdata `error_class`/`error_code == None` bullet, superseding an earlier `..._for_non_ack_rosctr` name) — added. A residual set of implementer-added supplementary unit tests (e.g. `test_BC_2_21_001_dedup_flags_independent_c2s`, `test_BC_2_21_009_bounds_check_passes_exact_match`, and similar direct-call sanity checks) exist in the file beyond this story's named-test enumeration; each duplicates behavior already covered by a named test under its AC and is not a mismatch. **BC title sync** — the Behavioral Contracts table's BC-2.21.007 and BC-2.21.009 title-column rows were stale against the current on-disk BC H1s; updated verbatim to `parse_s7comm_header` Returns None for an Unrecognized ROSCTR Byte (Safe-Reject, No Force-Fit)" (BC-2.21.007) and "Declared `param_length`/`data_length` Are Bounds-Checked Against Remaining Bytes Before Parameter/Data Block Access (Safe-Reject on Inconsistency)" (BC-2.21.009); BC-2.21.001/002/004/005/006/008 title rows already matched their on-disk H1s verbatim. **F-30 (deferred)** — added a one-line deferred note under the VP-051 Kani Obligation section: the VP-INDEX "deliberate-flip negative check" for VP-051's non-vacuity obligation is deferred to STORY-194's full formal-hardening pass; this story's skeleton scope ends at the `kani::cover!` obligations already specified. No BC H1 was authored by story-writer (BC-2.21.007/009 titles copied verbatim from the already-amended BC files); `behavioral_contracts:` frontmatter unchanged (no BC added or removed). Note: `input-hash` was left unmodified per this burst's edit-only scope (no `compute-input-hash --write`, no commit) — the stored `315fdf1` value predates the BC-2.21.007/009 title amendments already present on disk and will need a hash refresh in a subsequent state-manager burst. |
| 1.3 | 2026-09-24 | story-writer | Synced this story to the STORY-187 canonical-frame holdout human ruling (DF-CANONICAL-FRAME-HOLDOUT-001, 2026-09-24): ROSCTR `0x02` (Ack) AND `0x03` (Ack_Data) both require the 12-byte S7comm header (`error_class = data[10]`, `error_code = data[11]`, `header_len == 12`); Job (`0x01`)/Userdata (`0x07`) remain 10-byte with no error fields; a truncated Ack_Data frame (10 or 11 bytes) returns `None`, exactly like a truncated Ack. Reconciled against the amended BC-2.21.004 v1.2, BC-2.21.006 v1.1, BC-2.21.008 v1.2 (new H1: "`parse_s7comm_header` for ROSCTR=Ack (0x02) and Ack_Data (0x03) Requires 12 Bytes (Error Class + Error Code)"), BC-2.21.009 v1.1, and VP-INDEX.md v2.52's corrected VP-051 row. Every "10-byte Job/AckData/Userdata" assumption in this story is corrected to "10-byte Job/Userdata; 12-byte Ack/Ack_Data": (1) the Behavioral Contracts table's BC-2.21.006/008 title rows updated verbatim to the current BC H1s; (2) AC-187-008 narrowed to `data[1] ∈ {0x01, 0x07}` only, with a note that Ack_Data is handled by AC-187-010; (3) AC-187-010 rewritten to cover both Ack and Ack_Data symmetrically (12-byte minimum, error-field extraction, and the truncated-Ack_Data-returns-None case, both direct-call and `on_data`/T0814 levels); (4) AC-187-013 gained a second canonical-frame sub-case validating the real Ack_Data Setup Communication response (from BC-2.21.008's Canonical Test Vectors, sourced from cnblogs and corroborated by three further independent sources) parses with `header_len == 12` and `error_class`/`error_code == Some(0)`; (5) AC-187-009's totality bullet restated as the length-conditional formula `Some` iff (`0x01`/`0x07` and `len>=10`) or (`0x02`/`0x03` and `len>=12`); (6) the VP-051 Kani Obligation section's non-vacuity assertion corrected from `error_class.is_some() == (rosctr == Ack)` to `(rosctr == Ack \|\| rosctr == AckData)`, matching VP-INDEX.md v2.52's already-corrected row, plus a new explanatory paragraph on `header_len` selection; (7) AC-187-011's Ack-only bounds-check wording generalized to Ack/Ack_Data; (8) the Tasks section's ROSCTR-match bullet, unit-test-naming bullet, and a new canonical-frame task item updated; (9) three new Edge Cases rows (EC-012, EC-013, EC-014) added for Ack_Data truncation and the corrected parameter-block offset; (10) a new Architecture Compliance Rules bullet documents the correction and its four independent real-world sources. New tests named: `test_BC_2_21_008_ack_data_12_byte_header_and_error_fields`, `test_BC_2_21_008_truncated_ack_data_returns_none`, `test_BC_2_21_008_truncated_ack_data_on_data_emits_t0814_once`, `test_BC_2_21_008_canonical_ack_data_setup_communication_response_on_data`, `proptest_bc_2_21_006_008_some_iff_rosctr_and_length_conditional`. No BC H1 was authored by story-writer (titles copied verbatim from the already-amended BC files); `behavioral_contracts:` frontmatter unchanged (BC-2.21.006/008 were already listed). |
| 1.2 | 2026-09-24 | story-writer | Folded STORY-187 per-story adversarial pass 2 (F-19, F-21, F-22, F-23) findings into this story: **F-19** (policy DF-CANONICAL-FRAME-HOLDOUT-001) — new AC-187-013 requires at least one `on_data` test (`test_BC_2_21_006_canonical_setup_communication_job_frame_on_data`, traces BC-2.21.002 postcondition 3 / BC-2.21.006) that feeds a canonical, independently-sourced classic S7comm Setup Communication Job frame over real ISO-on-TCP framing (TPKT + standard class-0 COTP DT `02 F0 80`), asserts `classified_protocol == Some(Classic)` and no findings, with byte values NOT derived from project BCs and a doc-comment citation of the authoritative source (per research-agent sourcing — ADR-014 Decision 4 allowed sources; sourcing in progress); a matching Tasks entry was added. **F-21** — (a) AC-187-006/009/011 (BC-2.21.004/007/009) each gained a bullet requiring the malformed-header `Finding`'s evidence text to be reason-specific and asserted by the test, distinguishing the three conditions sharing the `malformed_header_reported_c2s`/`_s2c` dedup flag; (b) AC-187-010 gained an `on_data`-level truncated-Ack case (10- and 11-byte Ack headers -> exactly one T0814, "truncated Ack" reason), test `test_BC_2_21_008_truncated_ack_on_data_emits_t0814_once`; (c) AC-187-011 gained an Ack-header (`header_len == 12`) bounds case via `on_data` (`param_length == 1`, byte absent at 12 bytes -> T0814; byte present at 13 bytes -> clean), test `test_BC_2_21_009_ack_header_len_12_bounds_check`, asserting `s7comm_bounds_ok` directly. **F-18** (VP-051 section) — the bounds half of the Kani harness now specified to use a second symbolic `data_len: usize` input, assert `s7comm_bounds_ok(&h, data_len) == (data_len as u64 >= header_len + param_length + data_length)` as an exact equality, assert the param/data sub-slice `.get(..)` is `Some` when that condition holds, and use `kani::cover!` to confirm both outcomes are reached (non-vacuity for this half of the harness). **F-22** — AC-187-003 gained a fifth negative case, repeated same-direction CR (BC-2.21.001 EC-007), test `test_BC_2_21_001_repeated_same_direction_cr_session_not_established` (asserts `session_established == false` and `cr_observed_dir == Some(A)`); `test_BC_2_21_001_cr_only_session_not_established` is now explicitly scoped to case (a) only and does not claim EC-007; the Edge Cases table's EC-008 row was corrected to enumerate all five sub-cases with accurate per-case BC-2.21.001 EC-NNN citations. **F-23** — the fixture-generator Task now states the CC-only and two-DT-frames-in-one-delivery fixtures are covered by in-memory unit tests, not the generator, which is scoped to the CR/CC pair and the Setup Communication classic-S7comm frame(s); the `mk_modbus_pcap.py` citations (Tasks section and Library & Framework Requirements table) were corrected to `mk_modbus_large_pcap.py` per ADR-014 Decision 7. No BC H1 title changed; `behavioral_contracts:` frontmatter unchanged (all new ACs trace to already-listed BCs). |
| 1.1 | 2026-09-24 | story-writer | Propagated human-ratified rulings from STORY-187 per-story adversarial pass 1 (2026-09-24) into this story, per BC-2.21.001/002/004/008 v1.1 and ADR-0014 Decision 2/9 reconciliation notes: **F-01** — AC-187-001 gains the pending-CR-direction tracking field (`cr_observed_dir: Option<Direction>` or equivalent); AC-187-003 rewritten to opposite-direction CR->CC semantics with four named negative-case tests (CR-only, CC-only/no-prior-CR, CC-before-CR, same-direction CC). **F-02** — AC-187-005 rewritten: only the first DT frame carrying `protocol_id: Some(byte)` classifies; a `None`-protocol_id DT frame never classifies and does not consume "first DT frame" status; added a named None-then-`0x32` test and a two-frames-per-delivery first-write-wins test (BC-2.21.002 EC-003/EC-004). **F-12** — new AC-187-012: classic dissection is gated on the flow's *sticky* `classified_protocol == Classic`, not the current frame's raw `protocol_id` byte; a `0x32`-leading DT frame on an already Plus-/Unclassified-classified flow is not dissected and emits no finding (BC-2.21.002 EC-005). AC-187-004 rewritten to assert `classified_protocol == Classic` post-dispatch and exactly-one-T0814 on a malformed `0x32` frame on an already-Classic flow. AC-187-006/009/011 made explicit that per-direction malformed-header dedup is tested for BOTH c2s and s2c, with named s2c tests; AC-187-009's totality test now explicitly covers all 256 possible ROSCTR byte values. **F-13** — AC-187-010 notes that BC-2.21.008 Postcondition 4 (Ack error_class/error_code logging/consumption) is out of scope for this story, deferred to STORY-188. **F-14** — VP-051 section rewritten: non-vacuous Kani assertions specified (`len<10`->`None`; `Some`⇒`header_len∈{10,12}`∧`data.len()>=header_len`; `error_class.is_some()==(rosctr==Ack)`), bounded symbolic input (`[u8;16]`+assumed len, or `#[kani::unwind]`) required, and the BC-2.21.009 bounds check extracted as a pure crate-visible helper the harness calls directly. VP-053 section updated for the None-never-classifies and sticky-gating semantics; its proptest strategy must generate the `protocol_id: None` case explicitly. Tasks, Architecture Compliance Rules, and Edge Cases updated to match. No BC H1 title changed. Points: story-writer recommends re-evaluating the 8-point estimate given the added AC, doubled-direction test matrix, and the extracted pure-helper requirement — see handback report. |
| 1.0 | 2026-09-06 | story-writer | Initial authorship — flow-state completion, four-way dispatch skeleton (classic branch fully wired), `parse_s7comm_header`, VP-051 Kani skeleton, VP-053 proptest skeleton (partial), AC-187-001..011. |
