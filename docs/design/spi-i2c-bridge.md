# 68k-to-SPI/I2C Bridge — Design Doc (ttihp-26b proposal)

Status: brainstorm / pre-RTL. Builds directly on
[68k-bus-interface.md](68k-bus-interface.md) — read that first.

## 1. Motivation

A second payload chip on top of the shared 68k bus interface: a hardware
SPI master so the 68k can talk to external SPI memory (SPI flash/EEPROM/
FRAM, SD/microSD cards) without bit-banging chip selects and clock edges
in software. I2C is included here as a secondary, lower-priority feature —
the 68k can already bit-bang I2C directly, so hardware I2C only earns its
keep if it's nearly free on top of the SPI logic. As shown below, it isn't
quite free: I2C's bidirectional open-drain `SDA` runs into the same pin
budget the bus interface already spent on the data bus. Decide in §5
whether that tradeoff is worth it.

## 2. Pin Budget

| Pin | Signal | Direction | Notes |
|---|---|---|---|
| ui_in[6:0] | 68k bus control/address | in | identical to [bus-interface doc §3](68k-bus-interface.md) (CS_n, AS_n, R_W, DS_n, A1-3) |
| ui_in[7] | MISO | in | SPI data from slave — repurposes the bus interface's spare input pin |
| uio[7:0] | D0-D7 | bidir | shared register data bus |
| uo_out[0] | DTACK_n | out | self-generated, see [68k-bus-interface.md §3.1](68k-bus-interface.md#31-dtack-self-generated-plain-push-pull-output) |
| uo_out[1] | IRQ_n | out | transfer-complete, optional |
| uo_out[2] | SPI_SCLK | out | derived from system clock via divider |
| uo_out[3] | SPI_MOSI | out | |
| uo_out[4] | SPI_CS0_n | out | first SPI memory chip |
| uo_out[5] | SPI_CS1_n | out | second SPI memory chip (multi-drop) |
| uo_out[6:7] | spare | out | 3rd/4th CS, or repurpose per §5 |

This spends all `ui_in` and `uio` pins the same way the sound chip does
(bus interface + MISO on the one spare input bit), and uses 6 of 8
`uo_out` for DTACK/IRQ/SPI, leaving 2 spare `uo_out` pins.

## 3. SPI Master Engine

- 8-bit shift register, MSB-first, full-duplex (shifts `MOSI` out and
  `MISO` in on the same edges).
- Clock divider from system clock, register-selectable rate (SPI memory
  chips are typically happy anywhere from a few hundred kHz up to tens of
  MHz — pick a handful of useful divide ratios rather than a fully
  arbitrary one).
- CPOL/CPHA mode bits in a control register — support at least mode 0 and
  mode 3 (the two most common for SPI flash/EEPROM parts); all four modes
  if it's cheap.
- Multiple chip-select outputs (`CS0_n`, `CS1_n`, ...) selected by a field
  in the control register, so one bridge chip can talk to more than one
  SPI device (e.g. boot flash + a separate storage chip).
- `MISO` is asynchronous relative to our clock only in the sense that it's
  an external pin — same 2-FF synchronizer discipline as every other input
  here before it's sampled.

## 4. Register Interface & Bus/SPI Speed Mismatch

Unlike the sound chip (where a register write takes effect immediately),
an SPI byte transfer takes many `SPI_SCLK` cycles — much slower than a 68k
bus write. Two ways to handle that:

- **Recommended: busy/status polling (with optional IRQ).** Writing
  `TX_DATA` starts a transfer and sets a `BUSY` bit in `STATUS`; the byte
  received during that transfer becomes readable in `RX_DATA` once `BUSY`
  clears. Software either polls `STATUS` or waits for `IRQ_n`
  (transfer-complete) before touching the registers again. `DTACK_n`
  still completes normally on every bus cycle — only the *SPI* transfer is
  slow, not the register access itself.
- **Not recommended: stretch DTACK across the whole SPI transfer.** The
  68k is allowed to wait indefinitely for DTACK, so this is technically
  legal, but it ties up the entire 68k bus (blocking every other bus
  master and interrupt response) for the full transfer duration, and a
  logic bug here would hang the whole system rather than just this
  peripheral. Busy-polling degrades much more gracefully.

### 4.1 RX/TX FIFO

A single `RX_DATA`/`TX_DATA` register pair is only safe if firmware
strictly waits for `BUSY` to clear and reads `RX_DATA` before starting the
next transfer — fragile for burst-oriented code (e.g. reading a large SPI
flash block) that wants to queue ahead, and it forces tight polling with
no slack for firmware timing slop.

Add a shallow FIFO on both sides instead — 4 entries, same shape as this
project's own `dual_fifo.v`, reused directly rather than redesigned. This
is the same lesson tt08 already paid for once: the PS/2 decoder needed a
FIFO because keystrokes arrive asynchronously and unpredictably; here
bytes are host-paced rather than async, so a FIFO isn't strictly required
for correctness the way it was there, but it's cheap (internal logic only,
no extra pins) and removes a whole class of "firmware didn't poll fast
enough" bugs. Expose `rx_full`/`rx_ready` in `STATUS`, mirroring the
`fifo_full`/`data_rdy` pattern from `project.v` — including the ttihp
lesson that `fifo_full` needs to actually be wired to a visible bit, not
left dangling.

Registers (indices, not final — same indirect-addressing pattern as the
sound chip if we need more than 8 direct slots):

| Reg | Purpose |
|---|---|
| CTRL | CPOL/CPHA, clock divider, CS select |
| TX_DATA (write) | pushes a byte into the TX FIFO; hardware drains it one SPI transfer at a time |
| RX_DATA (read) | pops the oldest byte from the RX FIFO |
| STATUS | `BUSY`, transfer-complete (drives `IRQ_n`), `rx_ready` (RX FIFO non-empty), `rx_full` (RX FIFO full — byte(s) would be/were dropped), `tx_full` (TX FIFO full, write ignored until space frees up) |

## 5. I2C: the pin conflict

I2C fundamentally needs a bidirectional, open-drain `SDA` (the addressed
device pulls it low to ACK, and it must be driveable by either side), and
ideally an open-drain `SCL` too (to support clock stretching). Both `uio`
pins we'd want for that are already spent on the 68k data bus — all 8
`uio` are D0-D7, and freeing one would mean an awkward 7-bit-wide register
data path.

Options:

- **Drop I2C from this chip (recommended).** Keep bit-banging I2C on the
  68k in software, as today — it already works, and the pin cost here
  isn't justified by "nice to have." Ship SPI only.
- **Push-pull, no-clock-stretch, transmit-only I2C.** Put `SCL` on one of
  the two spare `uo_out` pins and... there's no good place for a real
  bidirectional `SDA`. A push-pull `SDA` output could drive a start/
  address/data sequence but could never read an ACK back or receive data
  from a slave — not real I2C, just a fixed-direction bit-banger, which
  the 68k can already do itself in software with no chip needed. Doesn't
  clear the bar.
- **Steal a `uio` bit from the data bus.** Would need a 7-bit (or
  multiplexed) register data path to free one `uio` pin for `SDA`, adding
  real complexity to the shared bus interface for the benefit of one
  payload chip. Not recommended given the interface is meant to stay
  simple and reusable.

Recommendation: **ship this chip as SPI-only.** Revisit I2C only if a
concrete use case shows up that the 68k's existing software bit-bang can't
handle (e.g. needing I2C activity to continue while the CPU is busy
elsewhere) — until then it doesn't earn a pin.

## 6. Test Plan

- cocotb: program `CTRL`/`TX_DATA` through the register interface, verify
  `SPI_SCLK`/`MOSI` waveform matches the programmed mode and divider,
  drive a known `MISO` pattern and verify it shows up correctly in
  `RX_DATA`, verify `BUSY`/`IRQ_n` timing around a transfer.
- FIFO-specific: back-to-back transfers with `RX_DATA` left unread, verify
  `rx_ready`/`rx_full` track FIFO occupancy correctly and that a full FIFO
  drops bytes cleanly (visible via `rx_full`) rather than corrupting
  unrelated entries — directly reusing the existing `test_fifo_overflow`
  pattern from `test/test.py`.
- Consider a cocotb SPI-flash behavioral model (respond to a JEDEC ID or
  read command) as an end-to-end sanity check, not just loopback.

## 7. Open Questions / Risks

- Confirm how many SPI chip selects are actually needed for the
  retrocomputer's storage plans — that decides how many of the 2 spare
  `uo_out` pins go to CS vs. staying spare.
- Revisit the I2C decision (§5) if a real use case emerges.
- Same clock/pin-template caveat as the other docs: confirm ttihp-26b
  matches tt08's 8/8/8 assumption before finalizing.

## 8. Dependency

Built on `bus68k_if.v` from [68k-bus-interface.md](68k-bus-interface.md),
same as the sound chip — no 68k bus signal handling is duplicated here.
