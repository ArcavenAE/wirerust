# PR Review — STORY-185 (COTP TPDU-Type Parser)

**PR:** #467 (feat(iso-on-tcp): add COTP TPDU parser (STORY-185))
**Reviewer:** vsdd-factory:pr-review-triage (dispatched agent `pr185-review-c1`)
**Cycle:** 1 of max 10
**Verdict: APPROVE**

Posted as PR comment:
https://github.com/Zious11/wirerust/pull/467#issuecomment-5564673215

## Review Method

Diff reviewed (`src/analyzer/iso_on_tcp.rs` +173/-5, `tests/iso_on_tcp_tests.rs`
+708) against BC-2.20.005–012. Branch built in an isolated worktree: all 22
`mod story_185` tests pass; `cargo clippy --all-targets -- -D warnings` clean.

## Acceptance-Criteria Trace (all satisfied)

| BC | Requirement | Implementation | Verified |
|----|-------------|-----------------|----------|
| BC-2.20.005 | `len < 2` -> `None` | `if tpkt_payload.len() < 2 { return None }` | Yes |
| BC-2.20.006 | LI-truncation -> `None` | `if tpkt_payload.len() < 1 + li { return None }` — no overflow (`li` <= 255), no OOB | Yes |
| BC-2.20.007 | CR (`& 0xF0 == 0xE0`), `protocol_id: None` | `0xE0 =>` arm | Yes |
| BC-2.20.008 | CC (`0xD0`), `protocol_id: None` | `0xD0 =>` arm | Yes |
| BC-2.20.009 | DT (`0xF0`) non-empty -> verbatim byte | `Some(tpkt_payload[payload_offset])` when `len > payload_offset` | Yes |
| BC-2.20.010 | DT empty payload -> `None` | `else None`, no OOB at `payload_offset == len` | Yes |
| BC-2.20.011 | any other high nibble -> `None` | `_ => None`; exhaustive & mutually-exclusive over all 16 nibble values | Yes |
| BC-2.20.012 | `protocol_id` never interpreted | Extracted verbatim; no comparison against `0x32`/`0x72`/"S7comm" anywhere in parsing logic (grep-verified against the diff) | Yes |

## Strengths

- Bounds safety is airtight: every index dominated by a prior length guard;
  `li == 0` and `li == 0xFF` both handled without panic.
- Rigorous test suite: exhaustive 16-value high-nibble partition, exhaustive
  256-value `protocol_id` totality sweep, `u8`-boundary byte checks, and four
  independent RFC 905 holdout vectors.
- VP-049 `#[cfg(kani)]` skeleton correctly scoped to no-panic/bounds-safety
  only; full classification/totality proof explicitly deferred to STORY-194.

## Non-Blocking Notes (no change requested — 0 blocking findings)

1. **Pre-dispositioned (per-story adversarial pass, already accepted):** the
   `protocol_id` regression-guard test comment claims the file contains zero
   occurrences of the guarded literals "anywhere"; the assertion in fact
   checks only the `0x32`/`0x72` byte literals (not the `"S7comm"` string,
   which is present in module doc comments). Accepted residual, not a merge
   blocker.
2. **New, also non-blocking:** degenerate `LI == 0` DT case — `payload_offset
   == 1`, so `protocol_id` becomes the TPDU-code byte itself. Contract-
   consistent (`payload_offset = 1 + LI` is the frozen definition) and
   explicitly pinned by
   `test_BC_2_20_006_li_zero_not_truncated_proceeds_to_classification` —
   documented behavior, not a defect.

## Convergence

Converged in 1 cycle. 0 blocking findings. No triage/fix dispatch required.

| Cycle | Findings | Blocking | Fixed | Remaining |
|-------|----------|----------|-------|-----------|
| 1 | 2 (non-blocking) | 0 | N/A | 0 |
