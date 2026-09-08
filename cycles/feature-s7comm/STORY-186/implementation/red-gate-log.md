---
document_type: red-gate-log
level: ops
version: "1.0"
status: final
producer: state-manager
timestamp: 2026-09-07T00:00:00
phase: f4
inputs: []
input-hash: "d41d8cd"
traces_to: "STORY-186"
stub_architect_agent: "[orchestrator-verified 2026-09-07]"
stub_compile_verified: true
test_writer_agent: "[orchestrator-verified 2026-09-07]"
red_gate_verified: true
---

# Red Gate Log: STORY-186 — S7comm ISO-on-TCP Carry-Buffer Reassembly, Walk-First Frame Extraction, Resync, and the Frozen SS-20/SS-21 Module Boundary

## Summary

| Story | Tests Written | All Fail (Red)? | Gate |
|-------|--------------|-----------------|------|
| STORY-186 | 14 behavioral + 2 static module-boundary guards | YES (14 behavioral RED; 2 static guards GREEN from stub baseline) | PASSED |

**Verdict: PASSED.** STORY-186 introduces `S7commAnalyzer` (`src/analyzer/s7comm.rs`, SS-21) as
a new file with `todo!()`-body stubs for all new methods (`on_data`, `on_flow_close`, and the
carry-buffer reassembly/resync internals). 14 behavioral tests exercising carry-buffer
reassembly, walk-first frame extraction, 1-byte resync, the carry-overflow guard, and
`on_flow_close` teardown were written first and confirmed to fail (panic on `todo!()`) against
the stub baseline. 2 static regression-guard tests — asserting `iso_on_tcp.rs` contains zero
`StreamAnalyzer` impls, and that no `IsoOnTcpFlowState` type exists anywhere in the tree — were
green from the stub baseline itself, since the stub scaffold does not violate the frozen
SS-20/SS-21 module boundary merely by existing.

## Stubs Created

### STORY-186: `src/analyzer/s7comm.rs` (new file, SS-21)

- `S7commAnalyzer` struct + `S7commFlowState` (carry_c2s/carry_s2c per-direction carry buffers
  and overflow latches) — stub fields present, no behavior.
- `on_data` — `todo!()` body; will implement directional carry-buffer TPKT reassembly, the
  walk-first residual-bound frame-extraction loop, the shared 1-byte resync sub-routine, and the
  65,535-byte carry-bound overflow guard.
- `on_flow_close` — `todo!()` body; will implement carry-byte teardown with no finding emitted.
- No modifications to `iso_on_tcp.rs` (SS-20) — STORY-186 only *consumes* STORY-184/185's
  stateless `parse_tpkt_header`/`parse_cotp_header` functions; the module boundary freeze means
  `iso_on_tcp.rs` must remain a pure-core parsing library with no `StreamAnalyzer` impl and no
  flow-state type of its own.

## Red Gate Verification

### STORY-186 — 14 behavioral tests, all RED against `todo!()` stubs

| Test Area | Count | Pre-implementation Status |
|-----------|-------|---------------------------|
| Carry-buffer reassembly across TCP segment boundaries | ~4 | RED (`todo!()` panic) |
| Walk-first frame extraction (residual-bound, no aggregate pre-check) | ~3 | RED |
| 1-byte resync (bad-version-byte + post-overflow, shared sub-routine) | ~3 | RED |
| Carry-overflow guard (65,535-byte bound, defense-in-depth per v1.1 reconciliation) | ~2 | RED |
| `on_flow_close` teardown (carry discard, no finding) | ~2 | RED |
| **Total behavioral** | **14** | **all RED** |

### Static module-boundary regression guards (GREEN from stub baseline)

| Guard | Assertion | Pre-implementation Status |
|-------|-----------|---------------------------|
| `iso_on_tcp.rs` zero `StreamAnalyzer` impls | `grep`-equivalent structural assertion | GREEN (stub scaffold does not touch `iso_on_tcp.rs`) |
| No `IsoOnTcpFlowState` type anywhere in tree | `grep`-equivalent structural assertion | GREEN (STORY-186 introduces `S7commFlowState`, not `IsoOnTcpFlowState`) |

These two guards freeze the SS-20/SS-21 module boundary: SS-20 (`iso_on_tcp.rs`) stays a
stateless pure-core parsing library forever; all `StreamAnalyzer`/flow-state behavior for
S7comm lives in SS-21 (`s7comm.rs`). Being green at the Red Gate (rather than red) is correct
and expected for this pair — they are architectural invariants, not new-behavior assertions,
and the stub scaffold itself does not violate them.

## Regression Check

| Test Set | Status |
|----------|--------|
| Full `cargo test --all-targets` at stub baseline | pre-existing suite unaffected; 14 new behavioral tests RED, 2 new static guards GREEN |
| STORY-184/185 TPKT/COTP parser tests (`iso_on_tcp.rs`) | all pass — untouched by STORY-186's stub scaffold |

## Hand-Off to Implementer

- Story ready for implementation: STORY-186.
- Implementation guidance:
  - Target: `src/analyzer/s7comm.rs` (new file, SS-21), consuming `parse_tpkt_header`/
    `parse_cotp_header` from `src/analyzer/iso_on_tcp.rs` (SS-20, unmodified).
  - Implement directional carry-buffer TPKT reassembly using a walk-first, residual-bound
    frame-extraction loop — explicitly no aggregate `carry.len() + data.len()` pre-check.
  - Implement a single shared 1-byte resync sub-routine reused verbatim for both bad-version-
    byte and post-overflow conditions (BC-2.20.013).
  - Implement the 65,535-byte carry-bound overflow guard as a defense-in-depth measure (clear-
    not-truncate + one finding per direction), per the v1.1 reconciliation of BC-2.20.014 —
    note this guard is proven unreachable under real `on_data` traffic given the walk-first
    resync design, and is retained deliberately (human ruling Option B), not as a live-reachable
    code path.
  - Implement `on_flow_close` to discard carry bytes on flow teardown with no finding emitted.
  - Exit gate: all 14 behavioral tests green; both static module-boundary guards remain green;
    `cargo clippy --all-targets -- -D warnings` zero warnings; full `cargo test --all-targets`
    green with no regressions.
- Post-implementation actual result: 18/18 green (14 behavioral + 2 static guards + 2 additional
  tests added during the Pass 1 remediation burst covering the BC-2.20.013/014 v1.1
  reconciliation and the AC-186-007 length-typo fix).
