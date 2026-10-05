## PR Review: #478 (fix PR: clippy 1.99 `clone_on_copy`)

**Verdict: APPROVE** (posted as a comment because the PR is self-authored and GitHub does not allow approving your own PR)

`covered_sha: 93b10804de05a412b8ca3a471e9ce5069bedd596`

### What I checked

- **The diff is what the description says.** One file (`tests/reporter_terminal_tests.rs`), +2/-2, one commit. The only changes are `Clone::clone(&a)` -> `a` at L4002 (`mod story_120`) and `Clone::clone(&grouped_collapsed)` -> `grouped_collapsed` at L4332 (`mod story_122`). Nothing else changed.
- **`FindingsRender` is `Copy`.** At the head SHA, `src/reporter/terminal.rs:143` has `#[derive(Debug, Clone, Copy, PartialEq, Eq)]` on `pub struct FindingsRender`. That is why clippy 1.99 now flags the UFCS `Clone::clone(&x)` form as `clone_on_copy`.
- **The tests behave the same.** For a `#[derive(Clone, Copy)]` type, the derived `clone()` returns `*self`, which is a bitwise copy, the same as `let c = a`. Every assertion compares the same values as before (`a == c`, `grouped_collapsed == gc2`). The source of `a` and `grouped_collapsed` is still usable after the move because they are `Copy`, so the later `assert_eq!(a, ...)` and `assert_eq!(grouped_collapsed, ...)` lines still compile and pass.
- **The `Clone` derive is still enforced.** `Clone` is a supertrait of `Copy`, so dropping `Clone` from the derive would fail to compile even with the explicit call removed.
- **Production code is untouched.** Only `tests/` changed, and `tests/` is excluded from the changelog gate, so no CHANGELOG entry is needed. The CHANGELOG gate is green.
- **Commit and title** follow the conventional format (`test: ...`). The Semantic PR check passes.
- **CI at review time:** Format, Semantic PR, CHANGELOG, Deny, Action pin and the other gates pass. Clippy, Test, Audit and Fuzz build were still running. This approval assumes Clippy and Test finish green before merge.

### Findings

| # | Severity | Category | Finding | Suggestion |
|---|----------|----------|---------|------------|
| 1 | NIT | coverage | `test_findings_render_derives_debug_clone_copy_partialeq_eq` (L4002) and the story_122 "Clone semantics" block (L4332) no longer call `Clone` at runtime. The `Clone` bound is still checked at compile time through the `Copy` supertrait, so nothing is lost. The assertion messages ("cloned Grouped", "Clone + PartialEq ... == clone") now describe a copy. | Optional: add `fn assert_clone<T: Clone>() {} assert_clone::<FindingsRender>();` to make the `Clone` check explicit, or reword the messages. Not required for merge. |

No BLOCKING or WARNING findings. Demo evidence, BC traceability and story-spec checks don't apply because this is a toolchain-drift fix PR, not a story.
