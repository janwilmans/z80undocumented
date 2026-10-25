# The Undocumented Z80 Documented 🕹️

The community reference for everything the Zilog Z80 does that the official manual never told you. Originally written by **Sean Young** (1997–2005) and maintained by **Jan Wilmans** since 2006. The current source is version 0.93 (27 June 2012).

If you're writing an emulator, a disassembler, or a ZX Spectrum / MSX / Game Boy-adjacent toolchain, or you just like poking at silicon, this is for you.

## What's inside

The document is a single LaTeX `book` (`z80-documented.tex`, about 5,600 lines). Its chapters are:

| Chapter | What you get |
|---|---|
| **Overview** | Z80 history, registers, flags (including the undocumented bits 3 and 5), power-on defaults and the pin descriptions |
| **Undocumented Opcodes** | The `CB`, `DD`, `FD`, `ED`, `DDCB` and `FDCB` prefix pages, plus what happens when prefixes are combined |
| **Undocumented Effects** | `BIT` flag quirks, memory and I/O block instruction flags (`LDI`, `CPI`, `INI`, `OUTI` and friends), 16-bit I/O ports, 16-bit additions and the full `DAA` truth table |
| **Interrupts** | NMI and maskable INT, IM 0/1/2, the `IFF1`/`IFF2` flip-flops, `HALT`, and where interrupts are and aren't accepted |
| **Timing and R register** | How `R` increments and how it relates to memory refresh |
| **Other Information** | A list of errors in the official Zilog documentation |
| **Instruction Tables** | Every instruction grouped by function, then sorted by opcode and by mnemonic |
| **Appendices** | Proofing and test routines, plus the GNU FDL |

## Building

You need a LaTeX distribution (TeX Live or MiKTeX). The document uses `fancyhdr`, `a4wide` and `multicol`.

```sh
pdflatex z80-documented.tex
pdflatex z80-documented.tex   # run twice to resolve the TOC and references
```

The result is `z80-documented.pdf`, an A4 document.

## Repository contents

- `z80-documented.tex`: the document source
- `z80-documented.pdf`: the built PDF
- `z80cpu_um.pdf`: Zilog's official Z80 CPU User Manual, for comparison
- `hyperref_manual.pdf`: the `hyperref` package manual, a LaTeX reference

## Contributing

Found an error, a missing flag effect or a cycle-count typo? Corrections are welcome. The `TODO` at the top of the `.tex` file lists open items:

- Describe the undocumented `BIT n,(HL)` flags, which come from the internal `MEMPTR`/`WZ` register
- Add test code to the appendices

Please open an issue or a pull request.

## License

Copyright © 1997–2005 Sean Young and © 2006–2012 Jan Wilmans.

Released under the **GNU Free Documentation License** (v1.1 or later), with no Invariant Sections and no Front-Cover or Back-Cover Texts. You can copy, distribute and modify it. The full license text is in the document.
