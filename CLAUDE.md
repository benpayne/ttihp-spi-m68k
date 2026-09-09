# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working in this repository.

## Project Overview

This is a Tiny Tapeout ihp-26b project: a hardware SPI master bridge for a
68k-based retrocomputer. It exposes itself as a memory-mapped peripheral
on the 68k's asynchronous bus and lets the CPU talk to external SPI
memory (flash/EEPROM/FRAM, SD cards) without bit-banging clock edges and
chip selects in software.

**Status: nothing is implemented yet.** `src/project.v` is a placeholder
that drives all outputs to zero. `src/dual_fifo.v` has been copied in
from a sibling project as a proven, reusable 4-entry FIFO — reuse it
directly for RX/TX buffering rather than rewriting it. Your job is to
build the rest from the spec below.

**Top module**: `tt_um_benpayne_spi_bridge`
**Target clock**: 25 MHz (set via `clock_hz` in `info.yaml` — plain round
number, no audio-style precision constraint here, feel free to revisit if
a different value makes the SPI clock divider chain cleaner)
**Design size**: 1x1 tile
**Process**: IHP SG13G2 (130nm BiCMOS), ihp-26b shuttle

## Read the spec first

- [`docs/design/spi-i2c-bridge.md`](docs/design/spi-i2c-bridge.md) — the
  actual spec for this chip: pin budget, SPI master engine, the
  busy/status-polling protocol for the bus/SPI speed mismatch, RX/TX FIFO
  design (§4.1), register map, test plan, and open risks. Also documents
  *why* I2C was descoped (§5) — don't re-add it without re-reading that
  reasoning.
- [`docs/design/68k-bus-interface.md`](docs/design/68k-bus-interface.md)
  — the shared 68k bus interface this chip is built on
  (`bus68k_if`-shaped register access: `reg_addr`/`reg_wdata`/`reg_rdata`/
  `reg_write`/`reg_read`). This is not implemented anywhere yet either —
  check whether the sibling project
  ([ttihp-sound-m68k](https://github.com/benpayne/ttihp-sound-m68k)) has
  already built it before writing your own from scratch; if both projects
  land on genuinely different needs, that's fine, but check first.

Both docs are living design docs from a brainstorming session, not
locked specs — if you find something that doesn't work once you're
actually writing RTL, that's expected; use your judgment and note the
deviation, don't treat every line as gospel.

## Key resolved decisions (don't relitigate these)

- **I2C is out of scope.** It needs a bidirectional open-drain SDA, and
  all 8 `uio` pins are already spent on the 68k data bus. The 68k can
  already bit-bang I2C in software — that's fine, keep doing that. See
  spi-i2c-bridge.md §5 for the full reasoning if this comes up again.
- **DTACK is self-generated** on a plain push-pull `uo_out` pin, not
  tri-stated. External board-level glue (a single AND gate) combines
  multiple peripherals' `DTACK_n` lines — this chip does not need to
  worry about bus contention. See 68k-bus-interface.md §3.1.
- **Don't stretch DTACK across an SPI transfer.** Use busy/status polling
  (with optional IRQ) instead — see spi-i2c-bridge.md §4. Stretching
  DTACK for an unbounded serial transfer ties up the whole 68k bus.
- **RX/TX FIFO, not a single register.** 4 entries each, reusing
  `dual_fifo.v` — see §4.1. This is cheap insurance against firmware
  timing bugs during burst transfers, even though SPI here is host-paced
  rather than truly asynchronous.

## Why this matters: the tt08 lesson

A prior sibling project (the PS/2 decoder that predates this
retrocomputer's ttihp port) was fabricated on tt08 *without*
metastability synchronizers on its async inputs, without CS glitch
filtering, and with a `fifo_full` status signal left unwired — and it
failed on the real board as a result. Don't repeat that: every signal
crossing from the 68k bus (and from `miso`) into this chip's clock domain
needs a synchronizer, and every FIFO status flag (`rx_ready`, `rx_full`,
`tx_full`) needs to actually be wired to a visible bit in `STATUS`, not
left dangling.

## Development Commands

Standard Tiny Tapeout cocotb flow (see `test/README.md` for details):

```bash
cd test
make -B                    # RTL simulation
```

Gate-level simulation, once synthesized: copy
`../runs/wokwi/results/final/verilog/gl/{top_module}.v` to
`test/gate_level_netlist.v`, then:

```bash
make -B GATES=yes
```

Keep `info.yaml`'s `source_files` and `test/Makefile`'s
`PROJECT_SOURCES` in sync as you add files to `src/`.

## File Structure

- `src/` — Verilog source (`project.v` is a placeholder; `dual_fifo.v` is
  ready to use)
- `test/` — cocotb testbench (currently the template default, needs
  rewriting per the test plans in the design docs)
- `docs/design/` — the specs described above
- `docs/info.md` — user-facing datasheet (fill in as the design solidifies)
- `info.yaml` — Tiny Tapeout project configuration (pinout, clock, source
  file list)
