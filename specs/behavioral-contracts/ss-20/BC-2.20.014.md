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
    change: "RECLASSIFIED as a defense-in-depth guard, unreachable by construction under the current BC-2.20.013 walk-first + BC-2.20.015 1-byte-resync design (STORY-186 adversarial gate F-02/F-03, two independent passes; human ruling: Option B — Defense-in-Depth). Because the walk-first loop extracts every complete TPKT frame before stashing only the trailing partial-frame residual to carry, and TPKT `length` is u16-capped at 65,535, the residual can never exceed 65,534 bytes (BC-2.20.013 Reconciliation Note); and because the resync sub-routine (BC-2.20.015) drains un-anchored garbage to fewer than 4 bytes before the end of each `on_data` call, garbage never accumulates across calls either. `residual.len() > 65,535` is therefore provably false on every call — EC-003's 'garbage accumulates past 65,535' premise is unsatisfiable and has been corrected to state the counterfactual it would guard against under a design regression, not a live runtime path. H1 and Description reworded accordingly; the guard's mechanics (strict `>` comparison, clear-not-truncate, one T0814 Anomaly/Possible/Medium per direction, per-direction dedup) are unchanged and remain the guard's specified behavior IF the precondition were ever reached. F-03 (entry-check timing: implementation checks the directional carry at call-entry — the previous call's residual — rather than this BC's original Precondition 1 'after-walk residual, same call') is reconciled in Preconditions/Invariants below: both timings are equivalent under the proven bound (neither can ever fire), so AC-186-005's call-entry framing is retained as the implementation's placement of choice."
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

# BC-2.20.014: Carry-Overflow Bound (`MAX_S7_ISO_ON_TCP_CARRY_BYTES = 65,535`) and T0814 Guard — Defense-in-Depth, Unreachable by Construction Under Walk-First Design

## Description

Per ADR-014 Decision 8, this BC specifies a defense-in-depth guard against unbounded
directional-carry growth: `MAX_S7_ISO_ON_TCP_CARRY_BYTES = 65,535`, derived from the
TPKT `length` field's own maximum representable value (`u16::MAX`, RFC 1006 §6), **not**
from COTP's single-byte Length Indicator (max 254). **This guard is NOT a live runtime
detection under the current design.** BC-2.20.013's walk-first frame-walk loop always
extracts every complete TPKT frame before stashing only the trailing partial-frame
residual to carry, and BC-2.20.015's resync sub-routine drains un-anchored garbage to
fewer than 4 bytes before the end of every `on_data` call — so the directional carry is
bounded at ≤65,534 bytes by construction (BC-2.20.013 Reconciliation Note), and
`residual.len() > 65,535` can never be true on the real `on_data` data path (STORY-186
adversarial gate F-02, confirmed by two independent passes; human ruling: Option
B — Defense-in-Depth, 2026-09-07). The guard and its T0814 (Denial of Service,
`ThreatCategory::Anomaly` / `Verdict::Possible` / `Confidence::Medium`) emission are
retained as **structural safety guards against future design drift** — e.g. if the
walk-first discipline (BC-2.20.013) or the resync semantics (BC-2.20.015) are ever
changed such that the walk no longer runs unconditionally to completion, or garbage is
no longer fully drained each call, this guard is the last line of defense against
unbounded per-flow memory growth. This mirrors the same live→defense-in-depth
reclassification pattern already used for `fix-tls-clienthello-frag` F-EV-001
(`buffer_saturation_drops`, BC-2.07.043/BC-2.07.005 v1.6) and the IEC-104
carry-bound characterization (BC-2.19.025 F-172-201).

## Preconditions

**These preconditions describe the guard's trigger condition AS SPECIFIED — i.e. what
would have to be true for the guard to fire. Under the current BC-2.20.013 (walk-first)
+ BC-2.20.015 (1-byte resync) design, Precondition 2 is provably unreachable (see
Invariant 1); the guard is defense-in-depth, not a condition the implementation expects
to ever satisfy on real traffic.**

1. The frame-walk loop (BC-2.20.013) has completed its extraction pass for the current
   `on_data` call, leaving a residual partial-frame tail.
2. `residual.len() > MAX_S7_ISO_ON_TCP_CARRY_BYTES` (i.e. `> 65,535`).

**F-03 reconciliation (entry-check timing):** the implementation checks the directional
carry at call-entry — i.e. it evaluates `carry[direction].len() > 65,535` on the
*previous* call's stashed residual, before the current call's `working = carry ++
incoming_data` concatenation and frame-walk begin — rather than literally re-deriving
"after this call's walk, same call" as Precondition 1 originally implied. AC-186-005
documents this call-entry placement explicitly. The two timings are equivalent here:
because the walk-first design bounds every stashed residual (from any prior call) at
≤65,534 bytes, `carry[direction].len() > 65,535` is false at call-entry for exactly the
same reason it would be false after the current call's walk — there is no call at which
either check can observe a value exceeding the bound. Precondition 1/2 above and
AC-186-005's call-entry framing are therefore reconciled as two descriptions of the same
(unreachable) condition, not a contradiction; the call-entry placement is ACCEPTABLE and
is the one the implementation should use going forward.

## Postconditions

**The following describe the guard's mechanics IF its precondition is ever reached
(defense-in-depth specification). Under the current design these postconditions are
never exercised on real traffic — see Description and Invariant 1 — but they remain the
binding specification for the guard's behavior should it ever fire, e.g. under a future
design regression.**

1. `carry[direction]` is cleared (set to empty) — the oversized residual is discarded,
   not truncated or partially retained.
2. The walk resyncs: it scans the discarded bytes (or continues scanning subsequent
   incoming bytes) for the next `0x03` version-byte candidate, advancing 1 byte at a
   time (BC-2.20.015) — this is a fresh-start resync, **not** a permanent desync latch;
   the flow remains tracked and subsequent valid frames are parsed normally.
3. Exactly one T0814 finding is emitted for this direction, with
   `verdict: Possible`, `confidence: Medium`, `threat_category: Anomaly`, guarded by a
   per-direction dedup flag (e.g. `carry_overflow_reported_c2s` /
   `carry_overflow_reported_s2c` on `S7commFlowState`) so that repeated overflow events
   in the same direction on the same flow do not each produce a new finding.
4. This dedup flag is **separate** from any malformed-TPKT/COTP-length dedup flag —
   each anomaly class has its own suppression flag, mirroring the IEC-104 precedent
   (BC-2.19.025/026's distinct `carry_overflow_reported_*` vs. `malformed_len_reported_*`
   flags).

## Invariants

1. **Bound derivation is exact, not heuristic; overflow is unreachable-by-construction
   for ALL traffic under the current design, not merely conformant traffic**:
   `65,535 == u16::MAX`, the maximum value the TPKT `length` field can ever represent.
   Under BC-2.20.013's walk-first discipline, the frame-walk loop always extracts every
   complete TPKT frame before stashing only the trailing partial-frame residual to
   carry; the largest such residual is a declared-but-incomplete frame with
   `length == 65,535` minus at least 1 already-available byte, i.e. `≤ 65,534` bytes.
   Under BC-2.20.015's resync sub-routine, any run of un-anchored garbage bytes is
   drained 1 byte at a time down to fewer than 4 remaining bytes before the current
   `on_data` call's walk terminates — garbage never accumulates carry-to-carry across
   calls. Both the legitimate-residual path (bounded ≤65,534) and the adversarial-garbage
   path (bounded <4 per call) therefore make `residual.len() > 65,535` provably false on
   every call, for both conformant and adversarial input. The bound is retained as
   defense-in-depth against a future design regression (see Description) — it is not,
   under the current design, reachable at all. This mirrors IEC-104's "conformant
   residual ≤ 254 bytes so the bound is fail-closed defense-in-depth" characterization
   (BC-2.19.025 F-172-201), extended here to cover the adversarial-garbage case as well
   because of the resync sub-routine's per-call drain guarantee.
2. **Clear, not truncate (IF reached)**: on overflow, the entire residual is discarded —
   there is no attempt to salvage a prefix of it, since a >65,535-byte undelimited
   residual has no reliable frame boundary to preserve. This mechanic applies only in
   the counterfactual case the guard is reached under a future design regression.
3. **Resync, not desync (IF reached)**: the flow is never permanently abandoned; a
   single T0814 is emitted and normal parsing resumes as soon as a valid `0x03`
   candidate is found and yields a parseable frame.
4. **Per-direction, per-flow scope (IF reached)**: the dedup flag lives on
   `S7commFlowState`, so a fresh flow (new TCP connection) gets a fresh
   overflow-reporting opportunity.
5. **Entry-check vs. post-walk-check timing is immaterial (F-03 reconciliation)**: since
   Invariant 1 proves the bound is never exceeded at any observation point — call-entry
   (previous call's residual) or post-walk (current call's residual) — the choice of
   where the implementation places the check does not change any observable behavior.
   AC-186-005's call-entry placement is consistent with this BC and is the placement of
   record.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | `residual.len() == 65,535` exactly (at the bound, not over it) | No overflow — this is the legitimate maximum-length single-frame case (BC-2.20.004 EC-002); bound comparison is strict `>`, not `>=` |
| EC-002 | A conformant TPKT frame declaring `length = 65,535` arrives split across many small TCP segments, with the carry temporarily holding up to `65,534` bytes of a still-incomplete frame | Never triggers overflow — the bound exactly accommodates the maximum legitimate single-frame residual; this is the load-bearing off-by-one correctness property named in the ADR (the carry cap must be `>=` the max declarable frame size, not merely "large") |
| EC-003 | **(CORRECTED 2026-09-07, STORY-186 adversarial F-02.)** Original premise — "an adversarial stream that never presents a valid TPKT header, causing the carry to grow past 65,535 bytes of accumulated garbage" — is **unsatisfiable under the current design** and has been struck. BC-2.20.015's resync sub-routine drains un-anchored garbage 1 byte at a time to fewer than 4 remaining bytes before every `on_data` call's walk terminates, so garbage never accumulates carry-to-carry across calls; it cannot reach anywhere near 65,535 bytes. | **Restated as the counterfactual the guard would catch under a design regression:** garbage cannot accumulate today, so no T0814 is ever emitted via this path on real traffic. IF a future change caused the resync sub-routine (or its invocation from the frame-walk loop) to stop draining garbage per-call — e.g. an early `break` that leaves undrained garbage in carry, or a change that stashes un-anchored bytes to carry without resyncing — THEN garbage could again accumulate across calls, and this guard would be the mechanism that catches the resulting carry growth once it crosses 65,535 bytes: one T0814 emitted, carry cleared, resync re-attempted. Not exercisable via the real `on_data` data path today. |
| EC-004 | A second overflow event occurs in the same direction on the same flow shortly after the first — **only reachable in the same design-regression counterfactual as EC-003, or via direct (non-`on_data`) construction of an oversized `S7commFlowState.carry_c2s`/`carry_s2c` for unit-level guard-mechanics testing** | No second T0814 emitted (dedup flag already set); carry is still cleared and resync still occurs each time |
| EC-005 | An overflow in `c2s` direction does not suppress or affect overflow detection in `s2c` on the same flow — **same reachability caveat as EC-004** | Independent dedup flags per direction |

## Canonical Test Vectors

| Scenario | Input | Expected Behavior | Category |
|----------|-------|--------------------|---------|
| At-bound, legitimate | Residual of exactly 65,535 bytes from a valid `length=65,535` frame still incomplete | No overflow; carry retains all 65,535 bytes; no finding | legit: exact ceiling — reachable via real `on_data` traffic |
| Over-bound, guard-mechanics (SYNTHETIC — not reachable via real `on_data` traffic) | `S7commFlowState.carry_c2s` (or `carry_s2c`) directly constructed/injected at 65,536 bytes with no valid TPKT frame boundary, bypassing the normal `on_data` walk-first/resync path, to unit-test the guard's own mechanics in isolation | Carry cleared; resync to next `0x03`; exactly one T0814 (Anomaly/Possible/Medium) for this direction | defense-in-depth: guard mechanics, IF reached |
| Repeated over-bound, same direction (SYNTHETIC — same reachability caveat) | Two consecutive directly-injected overflow conditions, same flow, same direction | First event: T0814 emitted, dedup flag set. Second event: no finding emitted, dedup flag already set; carry still cleared and resync still occurs | defense-in-depth: dedup guard mechanics, IF reached |

## Verification Properties

| Property | Proof Method (planned) |
|----------|-------------------------|
| `MAX_S7_ISO_ON_TCP_CARRY_BYTES = 65,535` exactly accommodates the maximum single-frame residual (`u16::MAX`); the overflow comparison is strict (`>`, not `>=`) so no conformant frame ever triggers a false overflow | Kani P0 (bound arithmetic per ADR-014 Decision 9) — VP-NNN allocation deferred to the F2 INTEGRATE sub-burst |
| Per-direction dedup guarantees at most one T0814 emission per direction per flow for repeated overflow events (guard mechanics, exercised via direct flow-state construction, not via `on_data`) | proptest P1 — VP-NNN allocation deferred |
| **(NEW 2026-09-07, STORY-186 F-02 closure)** Reachability property: for the composed system (BC-2.20.013 walk-first + BC-2.20.015 1-byte resync), `carry[direction].len() > 65,535` is false for every finite sequence of `on_data` calls — i.e. this BC's overflow precondition is unreachable via the public `on_data` entry point. This is the formal counterpart of Invariant 1 and should be proved (or at minimum property-tested) alongside VP-050 so the "unreachable by construction" claim is not merely asserted in prose. | proptest P1 (extends VP-050 scope) or Kani P1 if the bound arithmetic across BC-2.20.013/014/015 can be composed into one harness — VP-NNN allocation deferred to the F2 INTEGRATE sub-burst; architect to confirm VP-050 scope covers this or allocate a new VP |

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-20 ("ISO-on-TCP Framing (TPKT/COTP)") per domain/capabilities/cap-20-iso-on-tcp-framing.md §CAP-20 |
| Capability Anchor Justification | CAP-20 ("ISO-on-TCP Framing (TPKT/COTP)") per domain/capabilities/cap-20-iso-on-tcp-framing.md §CAP-20 — the carry-buffer bound is the DoS-resistance property of the ISO-on-TCP framing layer, protecting against unbounded per-flow memory growth |
| L2 Domain Invariants | INV-2 (Content-First Dispatch Precedence); bounded-resource design invariant (mirrors SS-19's `MAX_IEC104_CARRY_BYTES` treatment in ARCH-INDEX Bounded-Resource Design) |
| Architecture Module | SS-20/SS-21 boundary; `S7commFlowState.carry_c2s`/`carry_s2c`, `MAX_S7_ISO_ON_TCP_CARRY_BYTES` constant (planned) |
| ADR | ADR-014 Decision 8. **OPEN ITEM (2026-09-07):** if ADR-014 Decisions 5/8 assert the T0814 carry-overflow emission as a live runtime detection, that ADR text is now internally inconsistent with this BC's v1.1 defense-in-depth reclassification and needs an architect reconciliation note; not amended by product-owner (out of scope — flagged for orchestrator to dispatch architect). |
| Stories | (TBD — story-writer assigns in F3) |
| Feature | feature-s7comm |
| MITRE Techniques | T0814 (Denial of Service) — specified for the defense-in-depth guard path; `Verdict::Possible`, `Confidence::Medium`, `ThreatCategory::Anomaly`. **Not observable on real traffic under the current design** (see Description/Invariant 1); retained in the emitted-technique catalog as a guard against future design drift, not as evidence this technique is currently detectable end-to-end. |

## Related BCs

- BC-2.20.004 — depends on (the `length == 65,535` maximum this bound is derived from)
- BC-2.20.013 — composes with (this BC's overflow bound is proven unreachable by BC-2.20.013's walk-first residual bound — see BC-2.20.013 Reconciliation Note, added 2026-09-07)
- BC-2.20.015 — composes with (the resync mechanism invoked after clearing the carry; BC-2.20.015's per-call garbage-drain guarantee is the other half of the unreachability proof, alongside BC-2.20.013's walk-first residual bound)

## Architecture Anchors

- `src/analyzer/iso_on_tcp.rs` or `S7commFlowState` (planned) — `const MAX_S7_ISO_ON_TCP_CARRY_BYTES: usize = 65_535;`
- `S7commFlowState.carry_overflow_reported_c2s: bool` / `carry_overflow_reported_s2c: bool` (planned) — per-direction dedup flags, distinct from any malformed-length dedup flag
- `docs/adr/0014-s7comm-iso-on-tcp-stream-dispatch-and-parser-design.md §Decision 8` — "Carry-buffer sizing" and "Carry-overflow reaction" subsections, full derivation and reaction spec

## Story Anchor

STORY-186

## VP Anchors

- VP-050 (proptest P1) — TPKT/COTP Carry-Buffer Residual-Bound Reassembly, Overflow
  Isolation, and 1-Byte Resync; registered F2 INTEGRATE sub-burst per VP-INDEX.md
  v2.48; traces BC-2.20.013..015 (supersedes this BC's own speculative separate
  "Kani P0" note — the registered VP-050 is a single proptest VP covering both the
  bound-arithmetic and dedup-flag sub-properties)
- VP-055 (cargo-fuzz P1) — S7comm/ISO-on-TCP combined parse-chain no-panic fuzz
  (`fuzz_s7comm_parser`); registered representative-subset source_bc includes this BC

## Purity Classification

| Property | Assessment |
|----------|-----------|
| **I/O operations** | none |
| **Global state access** | per-flow mutable state (`S7commFlowState` carry buffer and dedup flags) |
| **Deterministic** | yes — given the same sequence of `on_data` calls |
| **Thread safety** | flow state is per-flow |
| **Overall classification** | stateful orchestration; the bound-comparison arithmetic itself is a pure, Kani-provable sub-property |
