# S7comm (classic, 0x32) Canonical Function-Code Test Vectors

**Research type:** general (technology / protocol wire format)
**Date:** 2026-10-04
**Story / finding:** STORY-188 adversarial finding F-01
**Policy:** DF-CANONICAL-FRAME-HOLDOUT-001
**Provenance regime:** ADR-014 Decision 4 + F-40 reconciliation note (2026-09-24, HUMAN RULING).
Publicly posted **wire-capture byte examples** are **PERMITTED AS TEST-VECTOR SOURCES ONLY**.
No GPL/LGPL source code consulted; field semantics/layout grounded only in permitted design
references (icsnpp-s7comm BSD-3, python-snap7 MIT, libs7comm BSD-2) and prose sources
(Wireshark wiki, Kleinmann & Wool 2014, Orange-Cyberdefense catalog, gmiru.com prose).
The precedent test-vector source is cnblogs "西门子S7通讯协议引用整理"
(<https://www.cnblogs.com/crcce-dncs/p/10659087.html>), already cited in ADR-014 Decision 9.

> **Scope reminder:** All hex below are protocol facts (observed on-the-wire bytes) usable as
> test inputs/expected outputs only. They are NOT used to ground parser field semantics.

---

## Convention

- Hex shown as posted includes the TPKT (`03 00 ...`) + COTP (`02 F0 80`) prefix.
- **The S7comm PDU begins at the `32` byte.** "S7 PDU offset" below is 0-based from `32`.
- **"Parameter offset"** is 0-based from the first byte of the parameter field (the function
  code), i.e. S7 PDU offset 10 for a 10-byte Job/Ack header.
- S7comm Job header (ROSCTR 0x01) = 10 bytes:
  `32 | rosctr(01) | redundancy_id(2) | pdu_ref(2) | param_len(2 BE) | data_len(2 BE)`,
  then the parameter field begins.

---

## 1. Write Var (FC 0x05) Job request — S7ANY, DB area (0x84)

**Source (primary, byte-exact):** cnblogs crcce-dncs, §"4.写操作" / "Write Demo 1 — DB10 Word18".
<https://www.cnblogs.com/crcce-dncs/p/10659087.html> — retrieved 2026-10-04 (direct WebFetch).
**Corroboration (structure only, no byte dump):** gmiru.com "The Siemens S7 Communication – Part 2"
(<http://gmiru.com/article/s7comm-part2/>) describes the identical S7ANY write-item layout.

**Full frame (PC → PLC):**
```
03 00 00 25 02 F0 80 32 01 00 00 00 05 00 0E 00 06 05 01 12 0A 10 02 00 02 00 0A 84 00 00 90 00 04 00 10 FF FE
```
S7 PDU (from `32`): `32 01 00 00 00 05 00 0E 00 06 | 05 01 12 0A 10 02 00 02 00 0A 84 00 00 90 | 00 04 00 10 FF FE`

**Header:** rosctr=01 (Job); pdu_ref=0x0005; param_len=0x000E (14); data_len=0x0006 (6).

**Parameter field (14 bytes) — field-by-field (parameter offsets):**
| p-off | byte(s) | meaning |
|-------|---------|---------|
| 0 | `05` | **function = Write Var** |
| 1 | `01` | item count = 1 |
| 2 | `12` | variable specification marker (0x12) |
| 3 | `0A` | length of following address spec (10) |
| **4** | `10` | **syntax ID = S7ANY (0x10)** ✅ (at parameter offset 4) |
| 5 | `02` | transport size = BYTE (0x02) |
| 6–7 | `00 02` | count = 2 |
| 8–9 | `00 0A` | DB number = 10 |
| **10** | `84` | **area = DB (0x84)** ✅ (at parameter offset 10) |
| 11–13 | `00 00 90` | byte/bit address (0x90 = 18×8 → byte 18) |

**Data field (6 bytes):** `00 04 00 10 FF FE` = reserved `00`, transport size `04` (BYTE-in-data),
length `00 10` (16 bits = 2 bytes), value `FF FE`.

**Additional area coverage from the SAME source page (byte-exact):**
- **Outputs (0x82)** — "Write Demo 3 — Output0":
  `03 00 00 24 02 F0 80 32 01 00 00 00 08 00 0E 00 05 05 01 12 0A 10 02 00 01 00 01 82 00 00 00 00 04 00 08 04`
  (param offset 4 = `10` S7ANY; offset 10 = `82` Outputs).
- **Markers/Flags (0x83)** — "Read Demo 8 — Flags" (read, but same S7ANY+area layout; area byte `83`):
  `03 00 00 1F 02 F0 80 32 01 00 00 05 65 00 0E 00 00 04 01 12 0A 10 02 00 02 00 09 83 00 00 00`
  (no byte-exact *write*-to-0x83 dump was located on this page; flagged single-aspect gap).

**Confidence:** HIGH for DB (0x84) and Outputs (0x82) writes (byte-exact, structure corroborated
by gmiru). MEDIUM for a Markers (0x83) *write* specifically (only a 0x83 *read* dump found).

---

## 2. PLC Control (FC 0x28) Job request — service "P_PROGRAM" (hot/warm start)

**Source A (primary, byte-exact):** cnblogs ZHIZRL, §"PLC Run 请求与响应".
<https://www.cnblogs.com/ZHIZRL/p/18184553> — retrieved 2026-10-04 (direct WebFetch).
**Source B (semantic corroboration):** gmiru.com Part 2 — confirms service `P_PROGRAM` and that a
start is sent "without a (routine) parameter" (verified via perplexity_ask against gmiru +
scadaprotocols). gmiru does NOT publish the byte-level reserved/FD layout.

**Full frame (PC → PLC):**
```
03 00 00 25 02 F0 80 32 01 00 00 00 00 00 14 00 00 28 00 00 00 00 00 00 FD 00 00 09 50 5F 50 52 4F 47 52 41 4D
```
S7 PDU (from `32`): `32 01 00 00 00 00 00 14 00 00 | 28 00 00 00 00 00 00 FD 00 00 09 50 5F 50 52 4F 47 52 41 4D`

**Header:** rosctr=01 (Job); pdu_ref=0x0000; param_len=0x0014 (20); data_len=0x0000.

**Parameter field (20 bytes) — field-by-field:**
| p-off | byte(s) | meaning |
|-------|---------|---------|
| 0 | `28` | **function = PLC Control** |
| 1–7 | `00 00 00 00 00 00 FD` | **7 bytes: six 0x00 then 0xFD** |
| 8–9 | `00 00` | u16 BE block-argument length = 0 (**no block args** for P_PROGRAM start) |
| 10 | `09` | service-name length = 9 |
| 11–19 | `50 5F 50 52 4F 47 52 41 4D` | ASCII **"P_PROGRAM"** |

Byte count check: 1 + 7 + 2 + 1 + 9 = 20 = param_len 0x14. ✅

**Layout-divergence verdict (explicitly requested):**
The assumed layout `[FC][6×0x00 then 0xFD][u16 BE block-len][block args][strlen][service str]`
is **CONFIRMED by the posted bytes** — the 7 reserved bytes ARE `00 00 00 00 00 00 FD`, the 2-byte
BE block-arg length IS present (`00 00`, zero here), and the 1-byte service-name length (`09`)
precedes "P_PROGRAM". **No divergence** from the assumed layout for the 0x28 start case.
Caveat: gmiru corroborates only the *semantic* ("P_PROGRAM start, no argument"), not the exact
0xFD reserved byte; the byte-exact 0xFD is attested by Source A only (single byte-exact source).

**Confidence:** HIGH that the assumed layout matches the posted frame; MEDIUM that the exact
reserved-byte pattern (`...00 FD`) is universal (one byte-exact source; semantic-only corroboration).

---

## 3. PLC Stop (FC 0x29) Job request — service "P_PROGRAM"

**Source (primary, byte-exact):** cnblogs ZHIZRL, §"PLC Stop请求与响应".
<https://www.cnblogs.com/ZHIZRL/p/18184553> — retrieved 2026-10-04 (direct WebFetch).
Corroborated semantically by gmiru Part 2 (0x29 = PLC Stop, service "P_PROGRAM").

**Full frame (PC → PLC):**
```
03 00 00 21 02 F0 80 32 01 00 00 00 00 00 10 00 00 29 00 00 00 00 00 09 50 5F 50 52 4F 47 52 41 4D
```
S7 PDU (from `32`): `32 01 00 00 00 00 00 10 00 00 | 29 00 00 00 00 00 09 50 5F 50 52 4F 47 52 41 4D`

**Header:** rosctr=01 (Job); param_len=0x0010 (16); data_len=0.

**Parameter field (16 bytes) — field-by-field:**
| p-off | byte(s) | meaning |
|-------|---------|---------|
| 0 | `29` | **function = PLC Stop** |
| 1–5 | `00 00 00 00 00` | **5 reserved bytes (NO 0xFD, NO 2-byte block-len field)** |
| 6 | `09` | service-name length = 9 |
| 7–15 | `50 5F 50 52 4F 47 52 41 4D` | ASCII **"P_PROGRAM"** |

Byte count check: 1 + 5 + 1 + 9 = 16 = param_len 0x10. ✅

**IMPORTANT divergence (Stop ≠ Control):** PLC Stop (0x29) uses a **different parameter layout**
from PLC Control (0x28). Stop has only **5 reserved bytes and NO 0xFD, and NO u16 block-length
field** before the service-name length. Do not assume the 0x28 layout for 0x29.

**Confidence:** HIGH (byte-exact source + semantic corroboration).

---

## 4. Read Var (FC 0x04) Job request — S7ANY, DB area (0x84)

**Source (primary, byte-exact):** cnblogs crcce-dncs, §"4.读操作" / "Read Demo 1 — DB10".
<https://www.cnblogs.com/crcce-dncs/p/10659087.html> — retrieved 2026-10-04 (direct WebFetch).
Structure corroborated by gmiru Part 2 (S7ANY read-item layout).

**Full frame (PC → PLC):**
```
03 00 00 1F 02 F0 80 32 01 00 00 00 1C 00 0E 00 00 04 01 12 0A 10 02 00 11 00 0A 84 00 00 98
```
S7 PDU (from `32`): `32 01 00 00 00 1C 00 0E 00 00 | 04 01 12 0A 10 02 00 11 00 0A 84 00 00 98`

**Header:** rosctr=01 (Job); pdu_ref=0x001C; param_len=0x000E (14); data_len=0.

**Parameter field (14 bytes) — field-by-field:**
| p-off | byte(s) | meaning |
|-------|---------|---------|
| 0 | `04` | **function = Read Var** |
| 1 | `01` | item count = 1 |
| 2 | `12` | variable specification (0x12) |
| 3 | `0A` | address-spec length (10) |
| **4** | `10` | **syntax ID = S7ANY** |
| 5 | `02` | transport size = BYTE |
| 6–7 | `00 11` | count = 17 |
| 8–9 | `00 0A` | DB number = 10 |
| **10** | `84` | **area = DB (0x84)** |
| 11–13 | `00 00 98` | byte/bit address (0x98 = 19×8 → byte 19) |

Also on the same page: Read of **Input 0x81** ("Read Demo 6": area byte `81`), **Output 0x82**
("Read Demo 7": area byte `82`), **Flags/Markers 0x83** ("Read Demo 8": area byte `83`) — all
byte-exact, all with syntax ID `10` at p-offset 4 and area byte at p-offset 10.

**Confidence:** HIGH (byte-exact; multiple area variants on one page; structure corroborated).

---

## Cross-cutting findings & contradictions

- **Offset invariants confirmed across all Read/Write vectors:** syntax ID 0x10 at parameter
  offset 4; area byte at parameter offset 10. No contradicting vector was found.
- **Area byte coverage located (byte-exact):** DB 0x84, Inputs 0x81, Outputs 0x82, Markers 0x83
  (all as reads; DB + Outputs also as writes).
- **PLC Control vs PLC Stop layouts DIFFER** (see §2 vs §3) — this is the single most important
  layout caveat; the 7-byte+FD+block-length structure is specific to 0x28, not 0x29.
- **Single-byte-exact-source items:** the 0x28 `...00 FD` reserved pattern and the 0x29 5-reserved
  layout each come from one byte-exact source (cnblogs ZHIZRL), with gmiru giving semantic-only
  corroboration. No source was found that *contradicts* these byte patterns.
- **No GPL dumps used:** none of the above bytes were taken from Wireshark `packet-s7comm.c`,
  Snap7, or libnodave. cnblogs/gmiru are public blog/KB prose with posted captures.

---

## Research Methods

| Tool | Queries | Purpose |
|------|---------|---------|
| **Perplexity perplexity_research (PRIMARY)** | 1 | Deep multi-source sweep for publicly posted S7comm 0x32 hex dumps (Write/Read/PLC Control/Stop); surfaced cnblogs crcce-dncs + ZHIZRL + gmiru |
| Perplexity perplexity_ask | 1 | Semantic corroboration of the 0x28 P_PROGRAM layout against gmiru/scadaprotocols |
| WebFetch | 3 | Direct byte-exact extraction from cnblogs crcce-dncs (success), cnblogs ZHIZRL (success), gmiru (TLS cert failure — fell back to perplexity) |
| Read / Grep / Glob | several | ADR-014 Decision 4 + F-40 ruling, convergence-state, research dir |
| Training data | 1 area | S7comm header/field layout conventions used only to *label* observed bytes (not as the source of the bytes) — flagged |

**Total MCP tool calls:** 2 (1 perplexity_research + 1 perplexity_ask)
**Training data reliance:** low — all byte vectors are from retrieved public web sources with
URLs + retrieval date; training data used only to annotate field names.

**Inconclusive / gaps flagged:**
- Byte-exact *write to Markers 0x83*: not found (only a 0x83 read dump).
- gmiru TLS cert mismatch blocked direct fetch; Wayback blocked; gmiru corroboration is therefore
  via perplexity synthesis of gmiru text, not a direct page capture.
- 0x28 `0xFD` reserved byte and 0x29 5-byte reserved layout: one byte-exact source each.
