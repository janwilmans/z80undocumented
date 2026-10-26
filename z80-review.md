# Review of "The Undocumented Z80 Documented" (v0.93, 27 June 2012)

Document reviewed: `z80-documented.tex` (5,648 lines). Zilog's `z80cpu_um.pdf` is in the repo for comparison.

---

## 0. How this review was done, and its limits

| Activity | What was actually done |
|---|---|
| Prose chapters | Read in full: Overview, Undocumented Opcodes, Undocumented Effects, Interrupts, Timing/R, Other Information. |
| Group instruction tables | Read in full: flags, encodings, M-cycles and T-states of every table (lines 1400–2723). |
| Opcode-sorted and mnemonic-sorted chapters (1,268 entries) | **Mechanically audited** with a script (`/tmp/claude-1000/-home-jan-project-z80undocumented/3cf922a4-a5ae-4d83-9bad-efeca93b1170/scratchpad/audit.py`, `xcheck.py`). The script regenerates the standard Z80 decode (main, CB, ED, DD/FD substitution, DDCB/FDCB) and diffs it against the document. It also checks operand bytes (`n`, `nn`, `d`, `e`), the `*` (undocumented) markers, and that the two sorted chapters agree with each other. **The reference decode is my own algorithmic table, not hardware.** |
| Zilog manual | The repo's `z80cpu_um.pdf` has an obfuscated text layer. Only the readable interrupt flip-flop table was usable (see Z80-016). Page-specific claims in the document's "errors in Zilog docs" list could **not** be re-checked (see Z80-024). |
| External sources | **No sources were fetched in this session.** References below (Boo-boo/Kladov *MEMPTR*, Patrik Rak's *z80test*, FUSE, MAME `z80.cpp`, Ramsoft's DAA work) are cited **from memory** and marked where detail is uncertain. Verify them before quoting. |
| Hardware | No hardware testing was done. Nothing here is "hardware-verified by me". |

Evidence labels used below: **[Z]** Zilog documentation, **[H]** reliable hardware research or test suite (as recalled), **[M]** multiple independent emulators or references agree (as recalled), **[D]** disputed, variant-dependent or unverified.

---

## 1. Overall assessment

**The document is a good, mostly accurate 2005-era reference for the *common NMOS Zilog Z80* undocumented behaviour. It is not a complete or demonstrably complete reference, and it should not be used as the sole source for a cycle- and flag-exact emulator.**

What holds up:
- Opcode decoding is sound. The audit found **no mnemonic, operand-byte or `*`-marker errors** in the 1,268 sorted entries. The two sorted chapters agree with each other. The fully decoded spaces are CB (incl. SLL), DD/FD (IXh/IXl/IYh/IYl substitution, (IX+d) forms), the ED duplicates (NEG, RETN/RETI, IM, IN F,(C), OUT (C),0) and all 512 DDCB/FDCB entries.
- Timings in the group tables (M-cycles / T-states) matched the standard values in every row I checked.
- The DAA tables, the `LDI/CPI` YF/XF rule, the `OUTI/INI` HF/CF/PF formulas and the IX+d rule for `BIT n,(IX+d)` are correct as far as I can tell [M].

What does not hold up:
- **`BIT n,r` undocumented flags are stated wrongly** (Z80-001).
- Several well-established findings from after 2005 are **absent**: the MEMPTR/WZ register, the Q-dependent `SCF/CCF` flags, flag behaviour of *repeating* block instructions, and the NMOS `LD A,I/R` interrupt bug (Z80-002 to Z80-005).
- **CPU variants are not mentioned at all**. Every undocumented behaviour is presented as universal, which the review brief forbids (Z80-006).
- "Timing" is thin: the chapter named "Timing and R register" contains no timing information, and undocumented opcodes have no T-states (Z80-009).
- The document's own TODO list admits the largest hole (MEMPTR / `BIT n,(HL)`), and the text currently under that heading is incorrect.

---

## 2. Findings

### Critical

#### Z80-001 — `BIT n,r` YF/XF rule is wrong
- **Severity:** Critical
- **Location:** §"BIT instructions" (lines 720–754), repeated in the BIT table footnote (line 2475).
- **Issue:** The text derives the flags by analogy with `AND r,2^n` and concludes "YF: set if n = 5 and tested bit is set; XF: set if n = 3 and tested bit is set". That is not hardware behaviour.
- **Evidence:** On real Z80s, YF and XF of `BIT n,r` are copied from **bits 5 and 3 of the operand register `r`**, whatever `n` is. Only SF depends on `n` (`n=7` and bit set). FUSE's `BIT` macro, `F = (F & CF) | HF | (value & (F3|F5))`, is the form I remember [M]. The document's other claim, that for `BIT n,(IX+d)` they come from the high byte of IX+d, is correct. It is the same "operand-derived" pattern.
- **Correction:** Replace the YF/XF items with: "YF = bit 5 of the tested operand, XF = bit 3 of the tested operand (for `(HL)` forms see MEMPTR, Z80-004; for `(IX+d)` forms, the high byte of the effective address)". SF = tested bit set **and** n = 7. ZF = PF = tested bit clear.
- **Validation:** `LD A,28h / BIT 0,A`. Bit 0 is clear, so Z=1, P=1, H=1. Correct F (carry as before) is `0x7C` | C: Y=1 and X=1 from A. The document's rule predicts Y=X=0 → `0x54` | C.

---

### Major

#### Z80-002 — Flags of *repeating* block instructions are missing or oversimplified
- **Severity:** Major
- **Location:** §Memory Block Instructions (786–840), §I/O Block Instructions (842–902), tables at 1745–1804 and 2650–2716.
- **Issue:**
  1. When `LDIR/LDDR/CPIR/CPDR` rewind the PC (BC≠0, and no match for CPxR), YF and XF are **not** the values the single-step rule gives. They come from the high byte of the rewound PC. The document only describes the single-step rule.
  2. `INIR/INDR/OTIR/OTDR` have **additional PF/HF modification during non-final iterations**. The document describes only the single-step formulas and presents the repeating forms as always SF=0, ZF=1, YF=0, XF=0 (rows at 2651, 2666, 2696, 2711). That is the state at completion only, and contradicts the text ("DEC B flags", line 853).
- **Evidence:** Documented by the ZX community's later flag research and implemented in Patrik Rak's *z80test* and MAME (a `block_io_interrupted_flags`-style helper, as I recall) [H/M]. **Exact bit positions and formulas are not reproduced here because I could not re-verify them. Take them from z80test's source and expected CRCs.** The existence of the effect is not in doubt.
- **Correction:** Add a "repeat/interrupted iteration" paragraph and table per family. Mark SF/ZF/YF/XF in the repeating rows as "as DEC B (see text)", not as constants.
- **Validation:** z80test's `ldir`, `cpir`, `inir`, `otir` cases, or an interrupt landing between iterations.

#### Z80-003 — `SCF` / `CCF` YF/XF rule is incomplete and variant-dependent
- **Severity:** Major
- **Location:** Table footnote 1 for `CCF`/`SCF` (line 1993): "YF and XF are copied from register A."
- **Issue:** On Zilog NMOS parts the result is not simply A. It depends on whether the **previous instruction modified F**: roughly `YF/XF = (Q ^ F) | A`, where Q is F if the previous instruction wrote F and 0 otherwise [H]. Other manufacturers and CMOS parts differ (z80test distinguishes several), so the rule is not universal [D].
- **Correction:** Document the Q rule, state that it is the Zilog NMOS model, and list the variant differences as open. Variant details I could not verify here.
- **Validation:** `XOR A / OR 28h / LD A,0 / SCF`. `OR 28h` leaves F=`2Ch`. `LD A,0` does not write F, so Q=0. The document's rule predicts Y=X=0 after `SCF`. The Q-model predicts Y=X=1 (from the old F). Not run on hardware; confirm with z80test.

#### Z80-004 — MEMPTR/WZ is undocumented, and the partial description given is wrong
- **Severity:** Major
- **Location:** TODO (line 27); §BIT instructions (762–784); Registers (229–230).
- **Issue:**
  1. `BIT n,(HL)` YF/XF come from the **high byte of the internal MEMPTR/WZ register**. The document says only "some sort of internal register ... related to 16 bit additions".
  2. MEMPTR is changed by a **large set of instructions** (not "most do not change it"). Examples as I recall from Boo-boo/Kladov [H/M], to be verified: `LD A,(nn)`, `LD (nn),A`, `LD rr,(nn)`, `LD (nn),rr`, `LD A,(BC/DE)`, `LD (BC/DE),A`, `JP/CALL/JR/DJNZ/RET/RST` (target), `EX (SP),HL`, `ADD/ADC/SBC HL,rr`, `RLD/RRD`, `IN/OUT` (all forms), `LDIR` and `CPIR` when repeating, `CPI/CPD`, `INI/IND/OUTI/OUTD`, and all (IX+d) forms.
  3. The listed rule "`ADD HL,xx`: use the high byte of HL before the addition" is wrong: MEMPTR = HL_before + 1, so the high byte differs when L = FFh.
  4. MEMPTR also affects the YF/XF of block-instruction flags in some cases, and the repeating flags (Z80-002).
- **Correction:** Add a MEMPTR section with a per-instruction update table, and fix the `BIT n,(HL)` rule to reference it.
- **Validation:** `LD HL,12FFh / LD BC,0 / ADD HL,BC / BIT 0,(HL)`. MEMPTR = 1300h, so Y/X come from 13h, not 12h. See Boo-boo/Kladov, *MEMPTR, esoteric register of the Z80* (2006) [cited from memory].

#### Z80-005 — NMOS `LD A,I` / `LD A,R` interrupt bug not mentioned
- **Severity:** Major
- **Location:** `LD A,I/R` table rows (1530–1538); §NMI (1063–1065); §Things affecting the IFFs (1159).
- **Issue:** The document says these instructions copy IFF2 to PF, full stop. On NMOS Z80s, if a maskable interrupt is accepted at the end of `LD A,I` or `LD A,R`, **PF is read as 0 even when IFF2 = 1**. The standard NMI-handler idiom ("check IFF2 via `LD A,I`") is affected. CMOS parts fix this [H/M, as recalled].
- **Correction:** Add the bug and its CMOS difference to the table footnote and the interrupt chapter.
- **Validation:** Hold INT low, `EI`, `LD A,I`; at the ISR entry do `PUSH AF`. NMOS shows PF=0, CMOS PF=1.

#### Z80-006 — No treatment of CPU variants anywhere
- **Severity:** Major
- **Location:** Whole document. §"History of the Z80" lists only speed grades and the Z280/Z380.
- **Issue:** The brief requires that variant dependence be stated. The document presents all undocumented behaviour as universal. Known or suspected variant differences that should at least be flagged:
  - `ED 71` / `OUT (C),0`: outputs 00h on NMOS but FFh on CMOS [H/M].
  - The `LD A,I/R` bug (Z80-005).
  - `SCF/CCF` YF/XF (Z80-003).
  - Reset state (AF=SP=FFFFh is attributed to unspecified chips, Z80-021).
  - Second-source parts (NEC, Sharp, Toshiba, ST, Mostek) differ in some flag details [D, unverified here].
  - Z180/Z280/Z380/eZ80 and LR35902 (Game Boy, mentioned in the README) are **not** Z80-compatible in the undocumented areas.
- **Correction:** Add a "Variants" section and tag every undocumented claim as universal, NMOS-only, or unverified.

#### Z80-007 — Prefix interactions are only partially specified
- **Severity:** Major
- **Location:** §"Combinations of Prefixes" (700–712), §DD Prefix (500–533).
- **Issue:** The text gives "last DD/FD wins" and one example. Missing (and needed for a correct decoder):
  - A DD/FD **before an instruction that does not use HL/H/L/(HL)** has no effect on the instruction but still costs 4 T-states and increments R. The same holds before `EX DE,HL`, `EXX`, and before ED-prefixed instructions (`DD ED 44` = `NEG` after a wasted DD).
  - Interrupt blocking after DD/FD is documented (1245–1260), but is not tied to this.
  - Effective address `IX+d` wraps modulo 65536.
  - DD/FD followed by `CB`: displacement is read *before* the final opcode, and the final opcode byte is a normal memory read, not an M1 cycle (which is why R grows by 2, not 3 — mentioned only as "weird enough" at line 1280).
- **Correction:** State these rules explicitly and add a decoding flowchart or state-machine description.
- **Validation:** `DD 00` takes 8 T-states total and R+2; `DD EB` behaves as `EX DE,HL` with 8 T-states.

#### Z80-008 — The "Timing" chapter contains no timing, and undocumented opcodes have no timing
- **Severity:** Major
- **Location:** Chapter "Timing and R register" (1263–1294); all undocumented rows.
- **Issue:** Group tables give T-states only for documented forms. There are no figures for: `*` forms such as `LD IXh,n`, `ADD A,IXl`, `INC IXh` (8 / 11 T-states, as per decode); undocumented DDCB variants (23 T); the 2-byte ED NOPs (8 T); a stray DD/FD (4 T); contended-memory / WAIT behaviour; M1 versus memory-read cycle layout; refresh cycle position; and interrupt-response timing for IM 0 with multi-byte instructions on the bus.
- **Correction:** Add timing rows for `*` entries and a bus-cycle section, or state that they are out of scope.

#### Z80-009 — Instruction table "Comments" column wrongly lists IXh/IXl/IYh/IYl for real H/L in `(IX+d)` forms
- **Severity:** Major (internal contradiction in primary instruction tables; sorted tables are correct)
- **Location:** `LD r,(IX+d)` (1446–1448), `LD (IX+d),r` (1468–1470), and similar `r` legends for `(IX+d)` rows.
- **Issue:** The legend for r=100/101 says `IXh` / `IXl` (`IYh` / `IYl`). In `LD H,(IX+d)` (DD 66) and `LD (IX+d),H` (DD 74), H and L are the **real** H/L, which is exactly what the DD-prefix text says (lines 513–516) and what the sorted chapters show (`DD66 LD H,(IX+d)`, `DD74 LD (IX+d),H`).
- **Correction:** Show H and L in the r-legend of the `(IX+d)` rows; reserve IXh/IXl for the `p`/`q` rows.

---

### Minor

#### Z80-010 — Generic YF/XF definition is misleading
- **Location:** §Flags (257–265, 282–283).
- **Issue:** "YF: a copy of bit 5 of the result" is false for `CP` (operand), `BIT`, `SCF/CCF`, `CPI/CPD` (n = A−(HL)−HF), `LDI`, and I/O block instructions. The text says XF/YF/NF "can only be read using PUSH AF" but HF cannot be read directly either.
- **Correction:** Say "of the result for most ALU instructions; see each instruction" and cross-link the exceptions.

#### Z80-011 — DAA description inconsistent with its own table
- **Location:** §DAA (1031–1032, 1008).
- **Issue:** The closing paragraph lists SF/YF/XF/ZF/NF but **omits PF**, which is the parity of the result (the instruction table at line 1940 correctly says PF). The heading "The NF flags is affected as follows" introduces what is actually the **HF** table.
- **Correction:** Add "PF = parity of the result" and rename the heading.

#### Z80-012 — 16-bit addition paragraph conflates ADD with ADC/SBC
- **Location:** §16 Bit Additions (923–929).
- **Issue:** "ZF is set if the whole 16 bit result is 0" and "SF ... affected" apply to `ADC/SBC HL,rr` only. `ADD HL,rr` / `ADD IX,rr` do not change SF, ZF, PF (the table at 2010 is right).
- **Correction:** Split into ADD vs ADC/SBC.

#### Z80-013 — Group tables: dangling footnotes and a mislabelled row
- **Location:** I/O table (2619–2723).
- **Issue:** Footnotes 4 and 5 are referenced (IND, OTDR) but only 1–3 are defined. OTIR's PF has no footnote (2696). The row `IN F,(n)` should read `IN F,(C)`. The LDI note "PF is 0 the result of BC-1=0" (line 1806) is garbled. The block I/O formulas use `(HL)` to mean "the byte transferred", which is ambiguous next to the L-after-update rule.
- **Correction:** Fix footnotes and label the byte `val`.

#### Z80-014 — `JR` displacement range note is off by one
- **Location:** Jump table note (2540).
- **Issue:** "e is ... in the range -127, 129" mixes conventions. As an offset from the instruction address the range is **−126 … +129** (opcode byte −128…+127 relative to PC+2).
- **Correction:** State both: "operand byte −128…127, relative to the address after the instruction; i.e. −126…+129 from the instruction start".

#### Z80-015 — Pin description errors
- **Location:** Data bus entry (391–393), HALT entry (395–400), RESET entry (439–446).
- **Issue:** `D7–D0` are described as "active low"; the data bus is active-high. "The HALT line is set" (1192) is ambiguous for an active-low pin. The pin diagram's pin assignments match the 40-pin DIP, so the diagram is fine.
- **Correction:** Fix polarity wording.

#### Z80-016 — NMI/IFF2 text contradicts itself, and the "Zilog errata" attribution is doubtful
- **Location:** NMI section (1051–1053) versus 1166–1172 and the errata list (1326–1328).
- **Issue:** Line 1052 says IFF2 is "to keep a copy of IFF1", which reads as the NMI doing the copy. Lines 1166–1172 correctly conclude via a nested-NMI experiment that IFF1 is **not** copied to IFF2 on NMI. The errata list then lists this as an error in "Zilog documentation". **Zilog's own interrupt table in the repo's `z80cpu_um.pdf` already says "Accept NMI: IFF1 = 0, IFF2 = \*" (unchanged)**, so the table is consistent with the author's finding. Only the prose around it is loose [Z, checked in the extracted table].
- **Correction:** Reword the NMI paragraph; reword the errata bullet to say "the prose of some documents implies a copy; the table does not".

#### Z80-017 — Cross-reference error
- **Location:** §Registers (229–230).
- **Issue:** "Also there is an internal register which is described in section `\ref{block_io}`". The I/O block section does not describe any register; the intended section is §BIT instructions (`bit_flags`).

#### Z80-018 — Opcode-sorted tables do not list the "no-op" ED space, ED77 and ED7F
- **Location:** Chapters at 2725 and 4007; the TODO comment at lines 2728 and 4010.
- **Issue:** `ED 77` and `ED 7F` are described in the ED section (lines 578, 586) but missing from both sorted chapters (the audit shows all other opcode slots present). The sentence "any instruction not listed has no effect" is in a comment, not in the text. The ED NOP space is 178 slots: 00–3F, 80–9F, A4–A7, AC–AF, B4–B7, BC–BF, C0–FF, plus 77 and 7F. RST entries use mixed formats (`RST 0H`, `RST 38H`) and `ED 70` is shown as `IN F,(C) / IN (C)` in one line while the others use one mnemonic.
- **Correction:** Add the missing rows and an explicit NOP-space table.

#### Z80-019 — Interrupt semantics: imprecise or missing details
- **Location:** §Interrupts (1034–1261).
- **Issue:**
  1. "Directly after an EI or DI instruction, interrupts aren't accepted" (1199, footnote at 1995): only `EI` has a delay; after `DI` interrupts are disabled anyway.
  2. NMI during the EI delay and NMI during DD/FD sequences are only *assumed* ("I haven't tested this", 1261).
  3. IM 2: "vector address = I×256 + bus value, LSB not forced to 0". This agrees with later experience, but when the vector LSB is FFh the second byte read from the next page is not discussed (unverified). The claim rests on one test by the author.
  4. IM 0 with a multi-byte instruction (e.g. `CALL`) on the bus is not covered; the 13 T-states are valid only for `RST`.
  5. Return address pushed when an interrupt hits a block instruction (PC rewound to the start of the instruction) is implied but never stated.

#### Z80-020 — R register: gaps
- **Location:** §R register (1265–1294).
- **Issue:**
  - `HALT` increments R every NOP-cycle while halted (not stated) [M, as recalled].
  - "Accepting a maskable or non-maskable interrupt increases R by one" is stated without evidence for the NMI part (unverified here).
  - The `LDIR` R-count sentence is correct but confusing for BC=0 (R += 2×65536 → 0 mod 128).
  - Whether R is incremented before or after being put on the bus is stated as unknown (1270).

---

### Informational

#### Z80-021 — Reset / power-on state is single-sourced and hedged
`AF = SP = FFFFh` is attributed to "Matt" with no data or chip list (288–297). Line 294 says IM is "probably 0" while the pin description (441–443) says reset sets IM 0. These should be reconciled and flagged as NMOS-observation (variant-dependent) [D]. Also state that reset does not clear the other registers and what happens to a pending EI delay.

#### Z80-022 — Missing "exact access order" information
The brief asks for read/write order. The tables give the symbolic operation but not bus order for `EX (SP),HL`, `PUSH`/`POP`, `LD (nn),rr`, `INC (HL)`, `CALL`, `RLD/RRD`, block instructions or wrap-around at 0000h / FFFFh. This matters for memory-mapped I/O, ROM, and flag-on-bus-error cases.

#### Z80-023 — Non-technical / factual blemishes in the History chapter
- The Z280 paragraph ends mid-sentence with an unclosed parenthesis (line 190–191).
- The Z280/Z380 paragraphs look copied from vendor material.
- Zilog's founding date and Shima's role (lines 165–167) look incorrect to me (Zilog was founded in 1974); verify before keeping.
- ChangeLog v0.92 reads "-change information not added yet-".
- "OTIR/INIT" typo in the v0.93 entry.

#### Z80-024 — Unverified items in the errata list
These Zilog-manual errata could not be checked because the text layer of `z80cpu_um.pdf` is obfuscated: the `LD r,(IX+d)` bit-pattern statement (1331–1332) is ambiguous as worded; the `ADD IX,pp` bit-6 statement (1335–1336); "page 21 figure 2" and "page 26 figure 16" (1350, 1354); "Z=0 if B<>0 otherwise Z=0" (1318). Mark as **unverified**.

---

## 3. Coverage matrix

| Area | Status | Notes |
|---|---|---|
| Unprefixed opcodes (252 + 4 prefix bytes) | **Complete** | Mechanically audited against standard decode. Mnemonics, operands and markers correct. |
| CB (incl. SLL 30–37) | **Complete** | Correct. |
| ED documented + duplicates | **Complete, minor gaps** | ED77/ED7F missing from sorted tables; NOP space not tabulated (Z80-018). |
| ED block instructions: encodings and timings | **Complete** | Correct. |
| ED block instructions: single-step flags | **Complete** | LDI/CPI/INI/OUTI rules match what I know [M]. |
| ED block instructions: repeating flags | **Incomplete** | Z80-002. |
| DD/FD substitution and IXh/IXl forms | **Complete** (encodings) / **Incomplete** (timing, unused-prefix rules) | Z80-007, Z80-008. |
| DDCB/FDCB (512 slots) | **Complete** | Result copy to register is correct; BIT forms correct. |
| Flags: standard ALU, rotates, DAA, NEG | **Complete** | Z80-011 (DAA PF text). |
| Flags: `BIT n,r` | **Incorrect** | Z80-001. |
| Flags: `BIT n,(HL)` / MEMPTR | **Incorrect + incomplete** | Z80-004. |
| Flags: SCF / CCF | **Incomplete / variant-dependent** | Z80-003. |
| Flags: LD A,I / LD A,R | **Incomplete** | Z80-005. |
| Flags: I/O block (single-step) | **Complete** | Formulas match the standard model. |
| Interrupts (NMI, INT, IM 0/1/2, IFF, EI delay, HALT) | **Mostly complete, imprecise** | Z80-016, Z80-019. |
| RETI/RETN equivalence | **Complete** | Consistent with later knowledge [M]. |
| R register | **Mostly complete** | Z80-020. |
| Instruction timing (documented) | **Complete** | Matched standard values in every row I checked. |
| Instruction timing (undocumented) / bus cycles / WAIT | **Missing** | Z80-008. |
| Power-on / reset | **Unverified** | Z80-021. |
| CPU variants (NMOS vs CMOS vs clones) | **Missing** | Z80-006. |
| Prefix edge cases | **Incomplete** | Z80-007. |
| Pin descriptions | **Mostly correct** | Z80-015. |

---

## 4. Opcode spaces and behaviours still requiring investigation

1. **MEMPTR/WZ update table.** Complete list per instruction, including interrupted block instructions and `(IX+d)` forms.
2. **Repeating block-instruction flags.** Exact YF/XF source and the PF/HF adjustments for `INIR/INDR/OTIR/OTDR` (use z80test).
3. **Q-register effects.** `SCF/CCF` and any other instruction that consumes the previous instruction's flag-write status.
4. **ED NOP space (178 opcodes).** Timing (8 T-states), R increment (2), and that the following byte is *not* consumed as an operand.
5. **DD/FD + ED / DD/FD + EX DE,HL / DD/FD + EXX.** Confirm "prefix ignored" on each variant.
6. **IM 2 with odd vector bytes.** Page-crossing behaviour of the second vector byte.
7. **NMI during EI delay, during DD/FD sequences, and during HALT.** The document says it didn't test these.
8. **R increment on NMI acceptance and during HALT.**
9. **`OUT (C),0` value (00h vs FFh) per chip family.**
10. **Interrupt acceptance in IM 0 with multi-byte opcodes on the bus** (timing and R).
11. **Behaviour on non-Zilog parts** (NEC, Sharp, Toshiba, ST, Mostek) for the above.

## 5. Unresolved or variant-dependent claims in the document

| Claim | Where | Status |
|---|---|---|
| `AF = SP = FFFFh` after reset; other registers undefined | 288–297 | Single source, unspecified chip [D] |
| IM after reset "probably 0" | 294 | Hedged; contradicts line 441 certainty |
| Vector LSB is not forced to 0 in IM 2 | 1100–1105 | Single-author test; plausible [D] |
| NMI does not copy IFF1→IFF2 | 1166–1172, 1326 | Supported by author test and Zilog's own table [Z] |
| `OUT (C),0` outputs 0 | 593 | NMOS only [D] |
| "SCF/CCF: YF/XF from A" | 1993 | Incomplete / variant-dependent [D] |
| All undocumented ED opcodes are RETN-like (including `RETI`'s IFF behaviour) | 610–613 | Consistent with later references [M] |
| "Z80 has no V flag distinct from P" etc. | 267–269 | Fine |
| "R register incremented on interrupt accept" | 1288 | NMI half unverified |

## 6. Prioritised recommended corrections

1. **Fix `BIT n,r` flags** (Z80-001) — affects every emulator that follows the document literally.
2. **Add MEMPTR/WZ section** and correct the `BIT n,(HL)` text (Z80-004).
3. **Document SCF/CCF Q-behaviour** and the `LD A,I/R` NMOS bug (Z80-003, Z80-005).
4. **Document repeating block-instruction flag behaviour** (Z80-002).
5. **Add a variants section** and tag every undocumented claim by applicability (Z80-006).
6. **Specify prefix handling fully**, with timing and R effects for unused prefixes (Z80-007, Z80-008).
7. **Correct the `(IX+d)` legends** in the group tables (Z80-009).
8. **Fix local errors**: DAA text (Z80-011), 16-bit-addition text (Z80-012), I/O footnotes (Z80-013), JR range (Z80-014), pin polarity (Z80-015), NMI wording and errata (Z80-016), cross-reference (Z80-017), missing ED77/ED7F rows (Z80-018).
9. **Add undocumented-opcode timing** and a bus-cycle/access-order section (Z80-008, Z80-022).
10. **Add verification pointers**: reference *z80test* and FUSE/ZEXALL for each undocumented flag claim, and say which tests exist for each.

---

*Written for: maintainers of the document and emulator/disassembler authors who rely on it. External references are from memory and unverified in this session; the opcode-table audit is reproducible with the scripts in the scratchpad.*
