# PR Review — #477 STORY-188 (S7comm Job/Ack_Data function-code classification)

**Verdict:** APPROVE (no blocking findings in the diff)
**covered_sha:** `6a6a7d39c23dd34bb85b61acf392f882279fcd25`
**Merge note:** do not merge until the `Clippy` check is green (see W-1). That failure is not caused by this diff.

## What I checked

- Every changed file: `src/analyzer/s7comm.rs` (+357/-20), `tests/s7comm_analyzer_tests.rs` (+1388/-17), `tests/fixtures/mk_s7comm_pcap.py`, `tests/fixtures/s7comm-fc-classification.pcap`, `CHANGELOG.md`, and `docs/demo-evidence/STORY-188/` (11 tape/gif/webm triples plus `evidence-report.md`).
- `classify_job_ack_function`: it is total, it handles `param_length == 0` first, and `header_len + param_length` uses `checked_add`. The parameter block is taken with `data.get(..)`. Every decoder reads only the `param` sub-slice and never `data.len()`.
- Write Var S7ANY offsets: syntax id at `param[4]`, area at `param[10]`, and a 14-byte minimum. These match the standard item layout `[FC, count, 0x12, len, syntax, transport, count(2), db(2), area, addr(3)]`.
- PLC Control decode: the arg length is the big-endian u16 at bytes 8-9. The name-length offset and name slice use checked arithmetic and `get`, and the name match is byte-exact. PLC Stop (0x29) is classified by the FC byte only, as the story requires.
- Ack record: the list is capped at 1024 with a saturating dropped counter. The `BTreeMap` count map has at most 2x256x256 keys and saturating counters. Job and Userdata frames record nothing, and frames that fail the bounds check record nothing.
- `cargo +1.99.0 clippy --all-targets --keep-going -- -D warnings` on the PR head: the only errors are the 2 pre-existing ones in `tests/reporter_terminal_tests.rs`. Files changed by this PR are clippy-clean.

## Test-evidence row-verify (PG-W74-PRDESC-ROW-VERIFY)

All 6 rows of the PR description's per-test table, checked against `tests/s7comm_analyzer_tests.rs` at the PR head:

- `test_BC_2_21_012_write_var_descriptor_length_boundary_11_12_13_14`: line 6189. Matches.
- `vp054::proptest_vp054_download_upload_structural_disjointness`: line 5865, `mod vp054` at 5844. Matches.
- `test_BC_2_21_008_ack_error_histogram_counts_beyond_list_cap`: line 6291. Matches.
- `test_BC_2_21_008_job_frames_record_no_ack_error_observation`: line 5976. Matches.
- `canonical::test_BC_2_21_016_canonical_plc_stop_classified`: line 6515, `mod canonical` at 6377. Matches.
- `vp051_kani::verify_classify_job_ack_function_param_slicing_safe`: line 6539, `#[cfg(kani)] mod vp051_kani` at 6529. Matches.

**Aggregate cross-check.** Source: CI run 37271461059, `Test` job, head `6a6a7d39`.
- `s7comm_analyzer_tests`: **117 passed; 0 failed**. This matches the claim.
- The sum over all `test result:` lines is 2838. The extra 1 comes from a separate CI step after `cargo test --all-targets`: the `iec104_e2e_real_pcaps_tests` manifest re-run (1 passed, 4 filtered).
- Excluding that step, the CI `--all-targets` total is **2837 passed; 0 failed**. That equals the claimed 2837.
- A local re-run at the same head also gave 2837 passed.
- Per-binary counts are identical between CI and the local run. Only the interleaving order differs.
- 117 − 83 pre-existing = 34 new `story_188` tests. This is consistent with the PR description.

## Output-format / breaking-change check

- **No output-format change.** `S7commAnalyzer` is still not registered with the dispatcher (STORY-193). The new code emits no `Finding` and no stderr. No reporter, JSON, CSV or CLI files are touched.
- **The public API is additive**, with one technicality (see N-1).

## Checklist

| # | Item | Result |
|---|------|--------|
| 1 | Diff coherence | PASS: every change is STORY-188 scope |
| 2 | Description accuracy | PASS: the description matches the diff; counts verified above |
| 3 | Test coverage | PASS: every new branch is exercised (exhaustive u8, boundary 11-14, truncation/case variants, cap/dropped, defensive unsliceable path), plus proptests VP-052/054 and a committed pcap end-to-end test |
| 4 | Demo evidence | PASS: 11/11 ACs mapped, with gif and webm per group, success and error paths, and a clean path scrub (no `/Users/` or `/home/`) |
| 5 | Commit quality | PASS: 21 commits, all in conventional format |
| 6 | Diff size | WARNING (informational): +2285 lines in total, but only 357 in `src/`. The rest is tests (1388), the fixture generator, and demo assets. Acceptable. |
| 7 | Missing changes | PASS: every test the story names under AC-188-001..011, the VP-051 Kani harness and the VP-054 proptest are present |
| 8 | Dependency status | PASS: STORY-187 is merged (`17b00031` = develop tip) |

## Findings

| ID | Severity | Category | Finding | Suggestion |
|----|----------|----------|---------|------------|
| W-1 | WARNING (merge-gating, not a diff defect) | dependency/CI | The CI `Clippy` check fails on rustc/clippy 1.99.0, which `dtolnay/rust-toolchain@stable` picked up on 2026-09-28. The errors are 2 `clippy::clone_on_copy` hits in `tests/reporter_terminal_tests.rs:4002` and `:4332` (`Clone::clone(&a)` on `FindingsRender: Copy`). That file is not in this PR. Develop's last green run was on 1.98.1, so develop will fail the same way on its next run. | Land a separate `chore`/`test` fix on develop: add `#[allow(clippy::clone_on_copy)]` on those two deliberate explicit-Clone assertions. Then rebase or re-run CI here. Do not merge #477 while `Clippy` is red. |
| N-1 | NIT | API | Before this PR, every field of `S7commAnalyzer` was `pub`, so an external crate could build it with a struct literal. The new private `ack_error_*` fields remove that ability. No in-tree caller uses a struct literal (all use `new()`/`Default`), and the analyzer is not yet public-facing. In practice this is not breaking. | Optional: leave as is, or mention it in the CHANGELOG. |
| N-2 | NIT | API | `S7ClassicFunction` is documented as "STORY-189 extends this enum with the `Userdata(..)` arm". Adding that variant will break exhaustive matches downstream. | Consider `#[non_exhaustive]` on `S7ClassicFunction`, `S7AreaCode` and `PlcControlService` while the S7 surface is still evolving. Can be deferred to STORY-189. |
| N-3 | NIT | description | `evidence-report.md` records branch HEAD `c6bd91e3`. The only commit after it is the evidence commit itself, so the recordings match the code at `6a6a7d39`. | None required. |
| N-4 | NIT | coverage | `decode_plc_control_service` does not check the `0xFD` marker at `param[7]`, so it is lenient on that byte. This is consistent with the STORY-191-192-DECODER-LENIENCY carry-forward that the description mentions. | Track it under that carry-forward. |
