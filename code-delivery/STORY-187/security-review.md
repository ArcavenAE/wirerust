---
document_type: security-review
level: ops
version: "1.0"
status: complete
producer: security-reviewer
timestamp: 2026-09-25T09:43:00
phase: 5
inputs: []
input-hash: "[live-state]"
traces_to: .factory/stories/STORY-187.md
total_findings: 4
critical: 0
high: 0
medium: 0
low: 2
files_reviewed: 2
---

# Security Review: wirerust — STORY-187 (PR #475)

## Executive Summary

No CRITICAL or HIGH findings against the new classic S7comm header parser
and four-way `protocol_id` dispatch. No panic, out-of-bounds slice, or
integer-overflow path is reachable from adversarial network input; the
Kani VP-051 formal-verification claim was independently checked against
the actual parser logic and found credible (not a misleadingly narrow
harness). Overall security posture: **APPROVE**, with 2 LOW observations
recorded as non-blocking architectural notes and 2 INFO items for context.

**Scope:** `parse_s7comm_header`, `s7comm_bounds_ok`, the new four-way COTP-
frame dispatch (`dispatch_cotp_frame`), `dispatch_classic_s7comm`,
`classify_first_dt_frame`, `classify_malformed_header_reason`,
`report_malformed_header` (`src/analyzer/s7comm.rs`), and both VP-051 Kani
harnesses (`verify_parse_s7comm_header_bounds_safety` at
`tests/s7comm_analyzer_tests.rs:4303`, `verify_s7comm_bounds_ok_bounds_safety`
at `tests/s7comm_analyzer_tests.rs:4473`), diffed against `origin/develop`.
This code parses untrusted network input (raw S7comm/ISO-on-TCP bytes from
live traffic or pcap capture), so a manual security pass was requested per
this repo's standing practice for parsers of untrusted input.

## Findings

### SEC-001: Slice-index reliance on an upstream (SS-20) invariant, not locally defended
- **Severity:** LOW
- **CWE:** CWE-125 (Out-of-bounds Read), context only — not currently exploitable
- **OWASP:** N/A (memory-safety class, not an OWASP Top 10 web category)
- **Attack Vector:** None reachable today. `dispatch_cotp_frame`'s
  `Some(0x32)` arm does `&tpkt_payload[header.payload_offset..]` without a
  local bounds check in `s7comm.rs`.
- **Impact:** Safe today because `iso_on_tcp::parse_cotp_header` guarantees
  `protocol_id == Some(byte)` is only returned when
  `tpkt_payload.len() > payload_offset` (verified at
  `src/analyzer/iso_on_tcp.rs:265-270`, itself VP-049's Kani obligation).
  The safety of this slice is a cross-module invariant, not something
  `s7comm.rs` re-verifies locally — a future change to
  `parse_cotp_header`'s `payload_offset`/`protocol_id` contract would
  silently reintroduce a panic here with no local guard to catch it.
- **Evidence:** `src/analyzer/s7comm.rs:671` (`dispatch_cotp_frame`,
  `Some(0x32)` arm slicing `tpkt_payload[header.payload_offset..]`);
  invariant source `src/analyzer/iso_on_tcp.rs:265-270`.
- **Proposed Mitigation:** Architecturally acceptable per ADR-014
  Decision 9's frozen SS-20/SS-21 pure-core boundary — no change requested
  for this PR. Flagged as a coupling point for future SS-20 changes to
  watch; a `.get()`-based guard could be added defensively in a follow-up
  if the boundary is ever relaxed.

### SEC-002: `debug_assert_eq!` as the only local self-check on a call-site invariant
- **Severity:** LOW
- **CWE:** CWE-617-adjacent (Reachable Assertion) — not exploitable
- **OWASP:** N/A
- **Attack Vector:** None. `dispatch_classic_s7comm`'s
  `debug_assert_eq!(payload.first(), Some(&0x32u8), ...)` compiles out in
  release builds (the release profile has `overflow-checks = true` but not
  `debug-assertions = true`).
- **Impact:** None — `parse_s7comm_header`'s own always-on
  `data[0] != 0x32` check (BC-2.21.005) is the actual defensive re-check
  that runs in all build profiles, so no panic-on-attacker-input risk
  exists whether or not the `debug_assert_eq!` is compiled in.
- **Evidence:** `src/analyzer/s7comm.rs` (`dispatch_classic_s7comm`
  function body).
- **Proposed Mitigation:** None required; noted for completeness only.

### SEC-003 (INFO): Kani harness scope is credible, not misleadingly narrow
- **Severity:** INFO (not counted in the LOW total above)
- **CWE:** N/A
- **OWASP:** N/A
- **Attack Vector:** N/A — this is a verification-of-verification check,
  not a vulnerability.
- **Impact:** None. Both harnesses use bona fide fully-symbolic input:
  `[u8;16]` with `len` ranging 0..=16 for the header-extraction harness (a
  complete equivalence-class cover of the real input space, since
  `parse_s7comm_header` only ever reads indices 0..11), and an
  independently-symbolic `data_len: usize` up to `u16::MAX*3` for the
  bounds-check harness, proven for exact two-directional equality.
  `kani::cover!` non-vacuity checks are present for both `None`/`Some`
  paths and both bounds outcomes. The VERIFICATION SUCCESSFUL claim is
  credible for the code as written.
- **Evidence:** `tests/s7comm_analyzer_tests.rs:4303`, `:4473`.
- **Proposed Mitigation:** None required. Non-blocking note: full
  non-vacuity currently relies on manual inspection since
  `--fail-uncoverable` is deferred to STORY-194 per the harness comment —
  a documented, accepted gap, not a new finding.

### SEC-004 (INFO): Finding-vector growth pattern (pre-existing, not introduced by this diff)
- **Severity:** INFO (not counted in the LOW total above)
- **CWE:** CWE-400-adjacent (Uncontrolled Resource Consumption), context only
- **OWASP:** N/A
- **Attack Vector:** An attacker opening many short-lived flows, each
  carrying one malformed classic-S7comm header, could grow
  `S7commAnalyzer::findings` proportionally to flow count.
- **Impact:** `S7commAnalyzer::findings` accumulates one T0814
  malformed-header finding per flow-direction (deduplicated via
  `malformed_header_reported_c2s`/`_s2c`), extending the existing
  unbounded-`findings`-Vec pattern already present from STORY-186's
  carry-overflow finding path. Architecturally identical to that
  pre-existing pattern — not a regression introduced by STORY-187.
- **Evidence:** `src/analyzer/s7comm.rs` (`report_malformed_header`,
  `S7commAnalyzer::findings`).
- **Proposed Mitigation:** Any fix belongs at the flow-table/dispatcher
  level (bounding total findings or flow count globally), out of this
  story's scope.

## Summary Table

| ID | Severity | CWE | Location | Status |
|----|----------|-----|----------|--------|
| SEC-001 | LOW | CWE-125 (context) | `src/analyzer/s7comm.rs:671` | accepted-risk |
| SEC-002 | LOW | CWE-617-adjacent | `src/analyzer/s7comm.rs` (`dispatch_classic_s7comm`) | accepted-risk |
| SEC-003 | INFO | N/A | `tests/s7comm_analyzer_tests.rs:4303,4473` | accepted (verification note) |
| SEC-004 | INFO | CWE-400-adjacent (context) | `src/analyzer/s7comm.rs` (`report_malformed_header`) | accepted (pre-existing pattern) |

## Positive Findings (Defensive Measures Present)

- `parse_s7comm_header` is length-gated at 10 bytes (Job/Userdata) and 12
  bytes (Ack/Ack_Data) before any indexing; all `u16::from_be_bytes` reads
  are inside the length-checked region; unrecognized ROSCTR bytes
  safe-reject (no force-fit) — CWE-125/CWE-20 concerns fully mitigated.
- `s7comm_bounds_ok` uses `checked_add` twice
  (`header_len + param_length + data_length` as `usize`), so no CWE-190
  (Integer Overflow) is reachable — a `None` from either `checked_add`
  correctly maps to a bounds-check failure rather than a wrapped/truncated
  comparison.
- `dispatch_classic_s7comm`'s malformed/oversized-declared-length paths
  both route to `report_malformed_header`, never to a slice construction —
  no OOB read is possible from this PR's code (consuming
  `param_length`/`data_length` to slice the payload is explicitly out of
  this story's scope, deferred).
- Two independent Kani harnesses (VP-051) formally verify bounds-safety of
  both `parse_s7comm_header` and `s7comm_bounds_ok` against fully-symbolic
  input, both reporting VERIFICATION SUCCESSFUL locally (0/264 and 0/306
  checks failed respectively), with non-vacuity `kani::cover!` checks
  present.
- Defense-in-depth: `parse_s7comm_header` re-checks `data[0] == 0x32` even
  though callers are expected to have already gated on
  `protocol_id == Some(0x32)` (BC-2.21.005).

## Recommendations Priority

### Immediate (before merge)
- None. No CRITICAL/HIGH findings block merge.

### Before Release
- None required for this story's scope.

### Post-Release
- Consider tracking SEC-001 (cross-module slice-safety coupling) if the
  SS-20/SS-21 boundary is ever revisited (e.g. alongside STORY-190's
  S7comm-plus implementation).
- SEC-004's unbounded-findings-Vec pattern should be addressed at the
  flow-table/dispatcher level in a future hardening pass, alongside the
  pre-existing STORY-186 instance of the same pattern.
