# PR Review — #475 (STORY-187): classic S7comm header parser + four-way `protocol_id` dispatch

**Reviewer:** pr-reviewer (fresh-eyes, diff + description + test evidence only)
**Head reviewed:** `31da9aff4a562ef269637d2808c0079f99c33919`
**Verdict:** **APPROVE** — 0 blocking, 3 suggestion, 3 nit

## What I checked

1. **Every changed file (26).** `src/analyzer/s7comm.rs` line by line; `tests/s7comm_analyzer_tests.rs` (the new `story_187` module: helpers, the Kani harnesses, the VP-053 proptest, and the fixture-pcap test); `CHANGELOG.md`; `docs/adr/0014-*.md`; `tests/fixtures/mk_s7comm_pcap.py` and the `.pcap` it generates; `docs/demo-evidence/STORY-187/*`.
2. **Correctness of `parse_s7comm_header`.** The length check (`< 10`) runs before any index. The `data[0]` check runs before any other field read. Indices 4..9 are only read after `len >= 10`, and 10/11 only after `len >= 12` in the Ack/Ack_Data arm. There is no panic site. Unknown ROSCTR values hit `_ => None`, so nothing is forced to fit.
3. **`s7comm_bounds_ok`.** It uses `checked_add` twice. `matches!` returns false if either addition overflows. No slice is ever built from the declared lengths.
4. **Dispatch.** The slice `&tpkt_payload[header.payload_offset..]` is sound. `parse_cotp_header` only returns `protocol_id: Some(_)` when `tpkt_payload.len() > payload_offset` (iso_on_tcp.rs:226-232). The CR/CC opposite-direction latch, the F-02 rule that `None` never classifies, and the F-12 conjunction gate all match the PR description and AC-187-003/005/012.
5. **Coherence.** Every change traces to STORY-187: the analyzer, its tests, the fixture and its generator, the ADR-0014 reconciliation notes for this story's rulings, one CHANGELOG entry, and the demo evidence. No other analyzer, the dispatcher, or `iso_on_tcp.rs` is touched.
6. **AC coverage.** I extracted all 52 test names the story spec cites for AC-187-001..013 and confirmed each exists as a `fn` in `tests/s7comm_analyzer_tests.rs` (`comm -23` gave an empty diff). Each AC has at least one named test.
7. **CI.** All 13 checks pass at this head: Test, Clippy, Format, CHANGELOG gate, Fuzz build, Deny, Audit, Action-pin, Semantic PR, Green-doc-tense, Help-provenance, Trust-boundary, and Bin selftest. The branch is mergeable (CLEAN).
8. **Dependencies.** The branch is based on `develop` `47951b7a`, which includes STORY-186 (#470) and its follow-up (#473).
9. **Demo evidence.** `evidence-report.md` is present, with one `.gif` + `.webm` + `.tape` for each AC group (001-003, 004/005/012, 006-009, 010/011, 013) and an all-green run. No absolute paths or usernames leak into `.md`, `.tape`, or the fixture generator.

## PG-W74-PRDESC-ROW-VERIFY: row and count verification

**Rows checked against `tests/s7comm_analyzer_tests.rs` (7 checked, at least 3 required). All match:**

| PR-body entry | Claimed line | Actual (`grep -n "fn <name>"`) | Match |
|---|---|---|---|
| `test_BC_2_21_001_flow_state_field_set` | 1632 | 1632 | YES |
| `test_BC_2_21_002_classic_s7comm_dispatch_asserts_classified_protocol_classic` | 2092 | 2092 | YES |
| `test_BC_2_21_008_canonical_ack_data_setup_communication_response_on_data` | 2731 | 2731 | YES |
| `test_BC_2_21_009_bounds_check_before_parameter_data_slice` | 3663 | 3663 | YES |
| `verify_parse_s7comm_header_bounds_safety` (Kani) | 4303 | 4303 | YES |
| `verify_s7comm_bounds_ok_bounds_safety` (Kani) | 4473 | 4473 | YES |
| `proptest_vp053_protocol_id_dispatch_totality` | exists | 4608 | YES |

**Aggregate counts, re-run locally at `31da9aff`:**

| Claim | Command | Result | Match |
|---|---|---|---|
| `s7comm_analyzer_tests` 83/83 | `cargo test --test s7comm_analyzer_tests` | `83 passed; 0 failed` | YES |
| `story_187` module has 63 tests | `cargo test --test s7comm_analyzer_tests story_187` | `63 passed; 20 filtered out` | YES |
| Rollback leaves the 20-test `story_186` module | `... story_186` | `20 passed; 63 filtered out` | YES |
| Full suite 2,803/2,803 | `cargo test --all-targets`, summed over all `test result:` lines (96 binaries) | `passed=2803 failed=0` | YES |
| Mutation 43 = 40 killed + 1 equivalent + 2 unviable | arithmetic only; not re-run | 40+1+2 = 43 | consistent |
| Kani VP-051 both harnesses successful | not re-run (the harnesses are `#[cfg(kani)]` and not part of CI) | — | not independently reproduced |

## Findings

| # | Severity | Category | Location | Finding | Suggestion |
|---|---|---|---|---|---|
| 1 | suggestion | coherence | `src/analyzer/s7comm.rs:802-831, 876` | Internal spec and policy IDs leak into the user-facing `Finding.evidence`, and the summary cites them twice. Each malformed-header `reason` ends with `(BC-2.21.004)`, `(BC-2.21.007)`, `(BC-2.21.008, DF-CANONICAL-FRAME-HOLDOUT-001)`, or `(BC-2.21.009)`. That string goes into `evidence`, and `summary` then wraps it and adds `(T0814; BC-2.21.004/007/008/009)`. A rendered summary therefore reads `...10 required (BC-2.21.004) (T0814; BC-2.21.004/007/008/009)`. The other analyzers (the STORY-186 carry-overflow finding here, IEC-104, DNP3) put BC IDs only in `summary` and keep `evidence` to wire facts. A process-policy ID like `DF-CANONICAL-FRAME-HOLDOUT-001` means nothing to an analyst reading a report. | Drop the `(BC-…)` / `DF-…` suffixes from the `reason` strings, so `evidence` holds only facts (byte counts and the ROSCTR value), and keep the single BC citation in `summary`. The existing tests match substrings like `"header too short: 9"` and `"unrecognized ROSCTR byte 0x00"`, so this should need little or no test churn. Fine as a follow-up (for example STORY-188, which touches this path next). |
| 2 | suggestion | coherence | `src/analyzer/s7comm.rs:671` | `&tpkt_payload[header.payload_offset..]` depends on an invariant of another module (SS-20 `parse_cotp_header`, VP-049), and nothing in this function re-checks it locally. It is correct today; the security review's LOW-01 flags the same point. The cost of a local guard is near zero. | Optional: `if let Some(payload) = tpkt_payload.get(header.payload_offset..) { … }`. This makes the analyzer panic-free by construction if SS-20's contract changes. Accept as-is if you would rather avoid an unreachable or equivalent-mutant branch; the security review already documents the coupling. |
| 3 | suggestion | size | whole PR | +5,752 / -34 lines, well above the 500-line guideline. Production code is modest (`s7comm.rs` +576/-24). Most of the size is tests (+3,934), the fixture generator (+445), the ADR (+174), and the demo evidence report (+218). | Non-blocking. The size comes from test and evidence depth, not scope creep, and splitting it now would not help review. Keep future SS-21 stories near this production-code footprint. |
| 4 | nit | description | PR body, Traceability table, row "BC-2.21.004/005/007" | The row lists `test_BC_2_21_004_*` and `test_BC_2_21_007_*` but leaves out BC-2.21.005's own test, `test_BC_2_21_005_defensive_reject_wrong_protocol_id_byte`. It also lists "Kani VP-051" as verification for the whole row, but the story's VP-051 source-BC set is {004, 006, 007, 008, 009}, which excludes 005. | Add `test_BC_2_21_005_*` to the row, and scope the VP-051 cell to 004/007 (or note that 005 is covered by unit test only). |
| 5 | nit | coverage | `tests/s7comm_analyzer_tests.rs:4503` | In `verify_s7comm_bounds_ok_bounds_safety`, the second assertion (`vec![0u8; data_len].get(header_len..end).is_some()`) mostly re-proves `Vec::get` semantics once the exact-equality assertion above it holds. No production code slices the parameter/data blocks yet. It is fine as a skeleton, but it does not tie the proof to a real call site. | When STORY-188 adds the actual parameter/data slicing, point this half of the harness at that call site (or a pure helper it uses), so VP-051 covers the slice that production code builds. |
| 6 | nit | coherence | commit history | Several commits use a `wip(STORY-187): …` prefix, and `wip` is not one of the repo's allowed conventional types. The Semantic-PR gate checks only the PR title, which is compliant. | Squash-merge this PR, which the history suggests is the normal practice, so `develop` gets a single conventional commit. No action needed if squashing. |

## Checklist summary

| # | Item | Result |
|---|---|---|
| 1 | Diff coherence | PASS. All changes trace to STORY-187; no unrelated files. |
| 2 | Description accuracy | PASS with nit #4. Counts, line numbers, the file list, the dependency status, and the scope of the `0x72`/unclassified placeholders all match the diff. |
| 3 | Test coverage | PASS. All 52 spec-named tests exist; 63 new tests; the local mutation and Kani evidence are consistent. |
| 4 | Demo evidence | PASS. `evidence-report.md` is present, with a `.gif` and `.webm` per AC group, recorded against the test harness (library surface, no CLI yet). |
| 5 | Commit quality | PASS with nit #6. |
| 6 | Diff size | FLAGGED (suggestion #3). Justified by test and evidence volume. |
| 7 | Missing changes | None found. The `S7commFlowState` fields, `S7Protocol`, `Rosctr`, `S7commHeader`, `parse_s7comm_header`, the `pub fn s7comm_bounds_ok`, the dispatch, the T0814 dedup, the VP-051 and VP-053 skeletons, and the fixture with its generator are all present. |
| 8 | Dependency status | PASS. STORY-186 (#470, #473) is merged to `develop`. |

**Verdict: APPROVE.** No blocking findings. The suggestions and nits can be deferred to follow-up work, which is where they belong given the post-convergence state. Per standing arrangement PG-MERGE-CLASSIFIER-F4, the human carries out the merge.
