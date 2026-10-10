# Protocol-Emulator ISA: Detailed Plan

*Design proposal — Jane Street protocol-emulator ASIC challenge (IHP 130 nm CMOS5L via Tiny Tapeout)*

This document is a concrete proposal: the microarchitecture the ISA assumes, the exact instruction encoding, the configuration space, worked programs for UART, I²C, CAN and USB that test whether the ISA holds up, a rough area budget, and the decisions still open. The reasoning behind each choice is included so the team can argue with it rather than just accept it.

## 1. Architectural model

### 1.1 Division of labor

Draw this boundary first, because it constrains everything else. The emulator implements the **link layer and below**: pin behavior, bit timing, line coding, stuffing, CRC, framing, handshakes, arbitration. Everything above that, such as USB enumeration, I²C register semantics, JTAG debug logic or Ethernet/IP, stays on the host MCU (the RP2040 on the demo board, or whatever drives the chip).

That boundary justifies having **no data memory and no load/store**. Protocol state at the link layer fits in a handful of registers and two shift registers. Adding data memory would add area, address-generation logic, and a whole class of verification state for no protocol that actually needs it.

### 1.2 Execution units: state machines

The recommendation is **2 independent state machines (SMs)** sharing a pin space, IRQ flags and the host interface. Each SM has:

- its own instruction memory bank
- PC, a 2-deep call stack, delay counter and clock divider
- 4 × 16-bit general registers (R0–R3) and a small ALU
- a 32-bit output shift register (OSR) and input shift register (ISR), with TX and RX FIFOs to the host
- a **bit-timing unit**: an NCO with edge resynchronization
- a **line pipeline**: CRC/parity tap, bit stuffer/destuffer, NRZ/NRZI/Manchester coder, differential driver/receiver
- a pattern-match unit on the ISR
- a 16-bit timer with prescaler
- sticky status flags

Why two and not one or four:

- **One** can't run a JTAG probe alongside a UART console, which is a realistic debug-probe use case. It also can't run Ethernet TX framing alongside link-pulse or RX handling.
- **Four** (as in PIO) roughly doubles the per-SM area. These SMs are much heavier than PIO's because each carries a CRC unit, an NCO and a line pipeline.

Parameterize the RTL on SM count and decide after the first synthesis run.

### 1.3 Pipeline and determinism

The ISA is only useful if timing is statically analyzable from the program text. The rules:

1. **Every instruction takes exactly 1 SM cycle plus its delay field.** Exceptions are instructions that explicitly stall (`WAIT`, blocking `PUSH`/`PULL`, blocking `STREAM`). Those resume a **fixed, documented number of core cycles** after the stall condition becomes true.
2. **No branch penalty.** Instruction memory is a synchronous-read SRAM, so a naive fetch/execute pipeline costs one bubble per taken branch. That bubble is deterministic, but it complicates every cycle count. Instead, compute `next_pc` combinationally in the execute stage, including condition evaluation, and drive it straight into the SRAM address port. The SRAM's registered output becomes the next cycle's instruction. The cost is one critical path: instruction register → decode → condition mux → next-PC mux → SRAM address setup. Conditions are all single flags or simple 16-bit zero tests, so that should close comfortably at the clock rates under discussion. **Verify it in the first synthesis/P&R pass.**
3. **No interrupts inside an SM.** Asynchronous events become sticky flags that the program polls with `JMP` or blocks on with `WAIT`. This is what makes formal timing proofs possible.
4. **Pin I/O latency is fixed and documented.** Inputs pass through a 2-flop synchronizer and outputs are registered. An instruction that samples a pin sees the pad's value from a fixed number of core cycles earlier, and an output appears a fixed number of cycles after the instruction that drives it. The assembler and the documentation must state both numbers, because they eat into turnaround budgets.

The single-port SRAM is the consequence for SM count: **each SM needs its own instruction bank.** Time-multiplexing one port halves every SM's instruction rate, which kills the USB budget. Find out which IHP SRAM macros Tiny Tapeout actually exposes on this node, and their sizes, before committing. If the smallest macro is much larger than 256 × 32, consider one macro shared through a wide word, or latch-based instruction memory.

### 1.4 Two timing regimes, one programming model

This is the central design decision.

- **Program-timed (bit-banged):** every bit is produced or consumed by an instruction, and the timing comes from the delay fields and the clock divider. This suits I²C, SPI, JTAG, SWD and PS/2, plus CAN when combined with the bit-timing unit. It is exactly the PIO model.
- **Engine-timed (streamed):** the `STREAM` instruction tells the line pipeline to clock N bits, or bits until an event, autonomously at the bit-timing unit's rate. It pulls from the OSR/TX FIFO or pushes into the ISR/RX FIFO, inserting and removing stuff bits and updating the CRC, while the SM stalls or does other work. This is how LS USB stays within its cycle budget and how 10BASE-T becomes possible at all.

Both regimes share the same OSR/ISR, CRC, stuffer, coder and pins, so a single program can bit-bang a SYNC pattern, `STREAM` a payload, and bit-bang an EOP. A separate fixed "USB serializer" peripheral would be less general and less novel.

## 2. Instruction format

The format is a fixed 32 bits. PIO uses 16 bits, but PIO has no CRC, stuffing, NCO or streaming controls, and cramming those into 16 bits would force mode registers, which is the "configure, don't program" trap again. With 256–512 words per bank, 32-bit width is 1–2 KB of SRAM per SM, which a macro handles fine.

```
 31      27 26  25   23 22     19 18                                    0
+----------+---+-------+---------+---------------------------------------+
|  OPCODE  |SE | SIDE  |  DELAY  |           OPERANDS (19 bits)          |
+----------+---+-------+---------+---------------------------------------+
    5b      1b   3b       4b
```

- **OPCODE (5 bits):** 13 opcodes are defined below, which leaves 19 slots for later additions. If the encoding needs the space, this could shrink to 4 bits and give operands 20 bits.
- **SE + SIDE (side-set):** when SE = 1, the 3 SIDE bits are applied to the configured side-set pins (up to 3 consecutive logical pins) **in the same cycle the instruction issues**, even if the instruction then stalls. This is how a clock edge and a data change stay atomically coordinated (SCK/MOSI, TCK/TMS, SCL/SDA).
- **DELAY (4 bits):** 0–15 extra SM cycles of idle after the instruction completes. Longer delays use the timer.

### Pin semantics: the open-drain abstraction

Each logical pin has a **pad mode** configuration bit:

| Pad mode | Logical 1 | Logical 0 |
|---|---|---|
| Push-pull | `out = 1`, `oe = 1` | `out = 0`, `oe = 1` |
| Open-drain | `oe = 0` (released) | `out = 0`, `oe = 1` |

The ISA always writes logical levels. On an open-drain pin, "1" automatically means release. I²C, PS/2 and 1-Wire firmware therefore never touch output enables. Push-pull bus turnaround (SWD, USB, SPI 3-wire) uses `PINDIRS` explicitly.

## 3. Instruction set

### 3.1 Summary

| Op | Mnemonic | Purpose |
|---|---|---|
| 00 | `JMP` | Conditional branch |
| 01 | `CALL` | Push PC+1 onto the 2-deep stack and branch |
| 02 | `RET` | Pop the stack |
| 03 | `WAIT` | Stall until a pin, tick, timer, IRQ or engine event (optional timeout) |
| 04 | `OUT` | Shift bits from the OSR to a destination |
| 05 | `IN` | Shift bits from a source into the ISR |
| 06 | `XFER` | PUSH (ISR → RX FIFO) / PULL (TX FIFO → OSR) |
| 07 | `MOV` | Register-to-register move with optional invert or bit reversal |
| 08 | `SET` | Load a 16-bit immediate into a register, pins, pindirs, timer or IRQ |
| 09 | `ALU` | 16-bit add, subtract, logic and shift ops |
| 0A | `PIPE` | Line-pipeline control: enables, CRC init, sticky clears, line forcing |
| 0B | `CFG` | Read or write a configuration register from or to a GPR |
| 0C | `STREAM` | Autonomous engine-timed TX/RX of N bits or until an event |

### 3.2 Encodings (operand field [18:0])

**JMP**

```
[18] INV  [17:14] COND  [13:10] PSEL  [9] rsvd  [8:0] TARGET
```

PSEL selects a pin, register or IRQ index, depending on the condition. TARGET gives a 512-word reach.

**Conditions** (with INV, 32 total):

| Code | Name | True when |
|---|---|---|
| 0 | `ALWAYS` | Always |
| 1 | `PIN` | Logical pin [PSEL] reads 1 |
| 2 | `RZ` | R[PSEL[1:0]] == 0 |
| 3 | `RDEC` | R[PSEL[1:0]] ≠ 0, then post-decrement (loop primitive) |
| 4 | `OSRE` | OSR shift count ≥ pull threshold |
| 5 | `ISRF` | ISR shift count ≥ push threshold |
| 6 | `TXFE` | TX FIFO empty |
| 7 | `RXFF` | RX FIFO full |
| 8 | `MATCH` | (ISR & MATCH_MASK) == (MATCH_VAL & MATCH_MASK) |
| 9 | `CRCOK` | CRC state == CRC_RESIDUE |
| 10 | `STUFF` | The stuffer says the next TX bit must be a stuff bit |
| 11 | `SERR` | Sticky: stuff-rule violation seen on RX |
| 12 | `ARB` | Sticky: an `IN … CMP` saw bus ≠ driven bit |
| 13 | `TOUT` | Sticky: the last timeout-enabled `WAIT` expired |
| 14 | `IRQ` | IRQ flag [PSEL[2:0]] set |
| 15 | `PAR` | Parity accumulator is odd |

The residue check in `CRCOK` is the standard trick: running the CRC over data *plus* the received CRC leaves a known constant (for example 0x800D for USB CRC-16, 0xC704DD7B for Ethernet). One comparator then handles every CRC flavor, and no 32-bit immediate has to fit into an instruction.

**CALL** `[8:0] TARGET` and **RET** (no operands). The stack has 2 levels, and overflow sets a sticky fault flag. Two levels covers subroutines like `i2c_byte` called from `i2c_transaction`. Deeper nesting in 256 words of protocol code is a smell.

**WAIT**

```
[18] POL  [17:14] SRC  [13:10] IDX  [9] TO  [8:0] rsvd
```

| SRC | Meaning |
|---|---|
| 0 | Pin [IDX] level == POL |
| 1 / 2 / 3 | Pin [IDX] rising / falling / any edge |
| 4 / 5 / 6 | Bit-timing unit BIT tick / SAMPLE tick / HALF tick |
| 7 | Timer expired |
| 8 | IRQ flag [IDX] == POL (clear-on-consume option) |
| 9 | Engine event [IDX]: MATCH, LINE_SE0, LINE_IDLE, SERR, STREAM_DONE, … |
| 10 | Differential line state [IDX] (J / K / SE0 / SE1) |
| 11 | **SYNCEDGE**: wait for an RX edge (polarity POL), then **hard-sync the NCO** to it |

When TO = 1, the timer runs concurrently. If it expires first, the WAIT falls through and sets `TOUT`. This is how I²C clock-stretch timeouts, USB response timeouts and PS/2 inhibit timing work without polling loops.

**OUT**

```
[18:15] DST  [14:10] COUNT (0 = 32)  [9] CRC  [8] PAR  [7] STF  [6:0] rsvd
```

DST is one of `PINS` (parallel onto COUNT consecutive out-pins), `LINE` (one bit through the TX pipeline: stuffer, coder, differential driver), `PINDIRS`, `NULL`, `R0`–`R3`, `TIMER`, or `CRC` (feed the CRC without driving a pin).

- CRC = 1 / PAR = 1: the shifted bits also feed the CRC LFSR or the parity accumulator.
- STF = 1 (LINE only): emit the stuffer's required stuff bit instead of OSR data. Programs check `JMP STUFF` before each bit in program-timed mode, so stuffing stays visible in the program and the cycle count stays honest.
- `OUT LINE` with COUNT > 1 is illegal: multi-bit line output is `STREAM`'s job.

**IN**

```
[18:15] SRC  [14:10] COUNT  [9] CRC  [8] PAR  [7] CMP  [6] DSTF  [5:0] rsvd
```

SRC is one of `PINS`, `LINE` (through the RX pipeline: differential receiver, decoder), `NULL` (shift in zeros), `ONES`, `R0`–`R3`, `STATUS`, or `OSR`.

- CMP = 1: compare the sampled bit against the last bit this SM drove on that line, and set sticky `ARB` on mismatch. This single bit implements I²C multi-controller arbitration, CAN arbitration and CAN ACK-slot checking.
- DSTF = 1: apply destuffing. If the stuffer marks this bit as a stuff bit, it is checked against the stuff rule (setting `SERR` if violated) and *not* shifted into the ISR.

**XFER** (PUSH/PULL)

```
[18] DIR (0 = PUSH, 1 = PULL)  [17] BLOCK  [16] IFTHR  [15:0] rsvd
```

Same semantics as PIO, including autopush/autopull, which is configured in SHIFTCTRL.

**MOV**

```
[18:15] DST  [14:11] SRC  [10:9] OP (none / invert / bit-reverse / byte-swap)
```

Registers: `R0`–`R3`, `OSR`, `ISR`, `PINS`, `PINDIRS`, `CRC` (as a source, the finalized CRC with reflection and xorout applied; as a destination, raw LFSR load), `CRCRAW`, `TIMER`, `CAPT` (timer capture of the last edge), `STATUS`. Bit-reverse handles MSB/LSB mismatches between the wire and the host's byte order.

**SET**

```
[18:16] DST (R0–R3, PINS, PINDIRS, TIMER, IRQ)  [15:0] IMM16
```

For IRQ, IMM[15] = set/clear and IMM[2:0] = flag index. Flag 7 is wired to the host interrupt pin.

**ALU**

```
[18:16] OP (ADD, SUB, AND, OR, XOR, SHL, SHR, MOVI)  [15:14] RD  [13] IMMF
[12:0]  IMM13 (signed)   or   [12:11] RS
```

Keep it 16 bits with no multiply or divide. SHR covers autobaud (measure 8 bit times, shift right by 3), and XOR followed by `JMP RZ` covers equality tests (PID decode, address match).

**PIPE**

```
[18:16] SET flags   [15:13] CLR flags        (flags: CRC_EN, STUFF_EN, PAR_EN; apply to STREAM)
[12:8]  ACTIONS     (CRC_INIT, PAR_CLR, STUFF_RST, STICKY_CLR, CODER_RST)
[7:6]   LINE_FORCE  (00 no change, 01 release force, 10 force SE0, 11 force idle/J)
```

LINE_FORCE produces line states the data path can't encode: USB SE0 EOP, Ethernet TP_IDL, CAN error-frame dominance.

**CFG**

```
[18:12] CFG_IDX (128 half-word registers)  [11:10] RS/RD  [9] DIR (write / read)
```

This lets a program reconfigure itself mid-protocol. The obvious case is switching between USB's CRC-5 (tokens) and CRC-16 (data packets) depending on the PID just received.

**STREAM**

```
[18]    DIR (TX / RX)
[17:16] COUNT_SRC (IMM, R0, R1, events-only)
[15:8]  IMM_COUNT (bits, 1–256)
[7:4]   STOP_MASK (MATCH, LINE_SE0, SERR, LINE_IDLE)
[3]     APPEND_CRC (TX: shift the finalized CRC out after the payload, through the same stuffer/coder)
[2]     BLOCK (stall the SM until done; otherwise continue and later WAIT on STREAM_DONE)
```

TX takes one bit per BIT tick. RX samples on each SAMPLE tick. Stuff bits never count toward COUNT. APPEND_CRC exists because USB, CAN and Ethernet all end their frames with the CRC sent the same way.

## 4. Configuration space (per SM)

These registers are written by the host at load time or by the `CFG` instruction. The assembler should emit them alongside the program, so a protocol "driver" is just a program image plus a configuration image.

| Register group | Contents |
|---|---|
| CLKDIV | 16.8 fractional divider giving the SM execution-enable rate (jitter ±1 core cycle) |
| PINCTRL | Out-base/count, in-base, side-base/count, side-targets-pindirs bit, LINE_TX / LINE_TX_N / LINE_RX / LINE_RX_N pin indices |
| PADMODE | Per-logical-pin push-pull / open-drain, plus a pin-ownership bit (which SM drives it) |
| SHIFTCTRL | OSR/ISR direction, autopull/autopush enable and thresholds |
| BITTIME | 24-bit NCO increment, 8-bit SAMPLE phase, resync mode (none / hard-on-SYNCEDGE / soft), SJW in phase units, resync edge polarity |
| LINE | TX/RX coding (NRZ, NRZI, Manchester, Manchester-inverted), differential enable, idle level, stuff mode (off / after-N-ones / after-N-identical), N (3 bits), stuff-bit rule (insert 0 / insert complement) |
| CRC | Width (1–32), POLY (32), INIT (32), reflect-in, reflect-out, xorout (all-zeros or all-ones), RESIDUE (32) |
| MATCH | MATCH_VAL (32), MATCH_MASK (32) |
| TIMER | Prescaler (8 bits) |
| EVENT | Event enables, SE0/idle minimum-duration filters |

**The bit-timing unit in detail.** A 24-bit phase accumulator advances by `INC = round(2^24 · f_bit / f_core)` every core cycle; it runs at the core clock, independent of CLKDIV. Its outputs:

- **BIT tick** on wrap
- **SAMPLE tick** when the top 8 phase bits cross SAMPLE_PHASE
- **HALF tick** at phase 0.5, for Manchester

Resynchronization works as follows. On a qualifying RX edge, the phase error is the current phase relative to 0, corrected by the known synchronizer latency. Hard sync sets the phase to 0. Soft resync subtracts the error clamped to ±SJW. That one mechanism covers several protocols:

- UART start-bit sync: hard sync via `WAIT SYNCEDGE`
- CAN: hard sync at SOF, soft resync with SJW on recessive→dominant edges
- USB: soft resync on every transition with a large SJW, which amounts to a first-order DPLL
- 1-Wire and PS/2 device-side timing

At 50 MHz, a 24-bit accumulator gives about 3 Hz frequency resolution, which is far below any protocol's tolerance.

## 5. Worked programs

Syntax: `OP operands [side n] [delay]`; `!` inverts a condition.

### 5.1 UART RX, 8N1

This shows the bit-timing unit removing the usual 16× oversampling loop. Configuration: BITTIME.INC = baud rate, SAMPLE = 0.5, resync = hard; LINE = NRZ, no stuffing; ISR shifts right (LSB first) with autopush disabled.

```
rx:     WAIT   0 SYNCEDGE            ; falling edge on RX; NCO phase := 0
        WAIT   1 SAMPTICK            ; mid start bit
        JMP    PIN[RX], rx           ; still high: glitch, re-arm
        SET    R0, 7
rbit:   WAIT   1 SAMPTICK
        IN     LINE, 1
        JMP    RDEC R0, rbit
        WAIT   1 SAMPTICK            ; mid stop bit
        JMP    !PIN[RX], ferr        ; stop bit low: framing error
        XFER   PUSH noblock
        JMP    rx
ferr:   SET    IRQ, set|7            ; tell the host, then resync on the next edge
        JMP    rx
```

The core does about 3 instructions per bit, independent of baud rate. The timing accuracy comes from the NCO, not from instruction counting, so the same program runs at 9600 baud and at 3 Mbaud. 7 data bits, parity or 2 stop bits are small edits.

### 5.2 I²C controller: write one byte, check ACK, with stretching and arbitration

Configuration: SDA = logical pin 0 (open-drain), SCL = side-set pin (open-drain), OSR shifts left (MSB first). SM tick = 1 / (10 · f_SCL), which is 4 MHz for 400 kHz.

```
i2c_wb:  SET    R0, 7               side 0        ; SCL held low
wbit:    OUT    PINS, 1             side 0 [4]    ; drive/release SDA; low phase = 6 ticks incl. JMP below
         WAIT   1 PIN[SCL], TO      side 1        ; release SCL; stall while a target stretches
         JMP    TOUT, i2c_timeout
         IN     PINS, 1, CMP                      ; sample SDA; ARB if bus != what we drove
         JMP    ARB, i2c_lost
         JMP    RDEC R0, wbit       side 0        ; SCL low again
         SET    PINS, 1             side 0 [4]    ; release SDA for the ACK slot
         WAIT   1 PIN[SCL], TO      side 1
         JMP    TOUT, i2c_timeout
         IN     PINS, 1                           ; ACK bit (0 = ACK)
         XFER   PUSH noblock        side 0
         RET
```

Timing check at 400 kHz with no stretching:

- Low phase = `JMP RDEC` (1) + `OUT [4]` (5) = 6 ticks = 1.5 µs, which meets tLOW ≥ 1.3 µs.
- High phase = `WAIT` (1) + `JMP` (1) + `IN` (1) + `JMP` (1) = 4 ticks = 1.0 µs, which meets tHIGH ≥ 0.6 µs.

SDA is sampled 2 ticks into the high phase. Stretching extends the high phase automatically, because `WAIT` doesn't start counting until SCL actually reads high. Timeout and arbitration-loss handling cost 2 instructions each. START, STOP and repeated START are 3–4 instruction subroutines.

### 5.3 CAN: transmit bits with stuffing and arbitration (program-timed with the NCO)

Configuration: BITTIME.INC = bit rate, SAMPLE ≈ 0.75–0.875 (the protocol's sample point), soft resync with SJW, recessive→dominant edges only. LINE = NRZ, stuff = after 5 identical, insert complement. CRC = 15-bit, poly 0x4599, init 0. R0 holds the bit count minus 1.

```
cbit:    WAIT   1 BITTICK
         JMP    STUFF, cstuff
         OUT    LINE, 1, CRC                 ; data bit: feeds CRC and stuff counter
         JMP    csamp
cstuff:  OUT    LINE, 1, STF                 ; stuff bit: no CRC, not counted
         WAIT   1 SAMPTICK
         IN     LINE, 1, CMP
         JMP    ARB, can_lost
         JMP    cbit                         ; stuff bit doesn't consume R0
csamp:   WAIT   1 SAMPTICK
         IN     LINE, 1, CMP
         JMP    ARB, can_lost                ; only meaningful during ID/RTR; mask otherwise
         JMP    RDEC R0, cbit
```

The worst case is about 7 instructions per bit. At 1 Mb/s with a 50 MHz core there are 50 cycles available, so there is plenty of slack. The CRC sequence itself can be sent with `MOV OSR, CRC` and the same loop. The fixed-form tail (CRC delimiter, ACK, EOF) runs after `PIPE` clears STUFF_EN.

Once arbitration is won, the data field could switch to `STREAM TX`. Program-timed is fine at CAN rates, though, and keeping the readback per bit is simpler.

### 5.4 USB low-speed: receive a packet (sketch, engine-timed)

Configuration: BITTIME = 1.5 Mb/s, soft resync with a large SJW. LINE: differential on, RX = NRZI, stuff = after 6 ones, insert 0. MATCH = SYNC (decoded `0x80`, aligned to the ISR's top byte).

```
urx:     WAIT   1 LINESTATE[K]               ; J→K transition = start of SYNC
         PIPE   clr CRC_EN, set STUFF_EN, STUFF_RST|STICKY_CLR
         STREAM RX, events-only, stop=MATCH|SE0, block    ; hunt for SYNC
         JMP    !MATCH, urx
         STREAM RX, imm 8, block             ; PID byte
         ; PID check (low nibble == ~high nibble) + dispatch, about 6 ALU/JMP instrs
         ; select CRC-5 or CRC-16 profile via CFG writes based on PID type
         PIPE   set CRC_EN, CRC_INIT
         STREAM RX, events-only, stop=SE0|SERR, block     ; payload + CRC until EOP
         JMP    SERR, ubad
         JMP    !CRCOK, ubad
         SET    IRQ, set|7                   ; packet ready
         ; device role: handshake TX (ACK) must start within the bus turnaround window
```

The whole packet costs about 15–25 instructions regardless of payload length. The response deadline (a handful of bit times, roughly 33 core cycles each at 50 MHz) leaves a few hundred cycles. That's enough to send an ACK or NAK, but **a DATA response to an IN token must already be sitting in the TX FIFO**, so the host-side driver has to stay ahead. EOP transmission is `PIPE LINE_FORCE=SE0`, two BIT ticks, then `LINE_FORCE=idle`, then release.

## 6. Pin allocation on Tiny Tapeout

The standard user interface is 8 dedicated inputs (`ui_in`), 8 dedicated outputs (`uo_out`) and 8 bidirectional pins with per-pin OE (`uio`). Only the `uio` pins can be open-drain or turnaround pins, so they are the protocol pins.

| Function | Pins |
|---|---|
| Host SPI slave (SCK, CS_n, MOSI) | `ui_in[0:2]` |
| Host SPI MISO, host IRQ | `uo_out[0:1]` |
| Logical pins 0–7 (bidirectional, open-drain capable) | `uio[0:7]` |
| Logical pins 8–11 (input-only) | `ui_in[4:7]` |
| Logical pins 12–15 (output-only) | `uo_out[2:5]` |
| Debug: SM0/SM1 trigger outputs (driven by `SET IRQ` to a logic analyzer) | `uo_out[6:7]` |

That leaves `ui_in[3]` spare, perhaps as an external clock or a global trigger. A 16-entry logical pin space matches the 4-bit PSEL/IDX fields.

The debug trigger outputs are worth the pins. During silicon bring-up, being able to scope "the SM reached this instruction" is invaluable.

Host SPI throughput is roughly f_core/4 on SCK, about 12 Mb/s raw, which is fine for everything except sustained 10BASE-T. Ethernet frames would have to be buffered or throttled. That is another reason it stays a stretch goal.

## 7. Rough area budget

These are **flop counts only**, as a first-order sanity check. Convert them to area with the team's own synthesis run on the IHP flow rather than an assumed cells-per-flop figure.

| Block (per SM) | Flops (approx.) |
|---|---|
| Sequencer (PC, stack, delay counter, CLKDIV accumulator and config) | ~80 |
| R0–R3 (16-bit) | 64 |
| OSR/ISR plus shift counters | ~76 |
| TX/RX FIFOs (4 × 32 each) | 256 |
| Bit-timing unit (phase, increment, sample, SJW, mode) | ~70 |
| CRC (state, poly, init, residue, flags) | ~136 |
| Match unit | 64 |
| Timer and prescaler | ~32 |
| Stuffer, coder, sticky flags | ~32 |
| Pin config and synchronizers | ~70 |
| **Per-SM total** | **~880** |

Two SMs come to about 1,760 flops, plus the host interface and global registers (about 150), plus the instruction SRAM macros. The FIFOs are the largest single item. If area gets tight, the cuts in order of least pain are:

1. Shrink the FIFOs to 2 entries.
2. Replace the CRC INIT register with an "init = 0 / all-ones" flag (saves 32 flops per SM, and every target protocol uses one of those two).
3. Narrow the MATCH registers to 16 bits.

Do not cut the NCO or the CRC unit. They are the novelty.

## 8. Assembler and pseudoinstructions

The assembler should be a Python tool that emits the instruction image plus the configuration image, in the same spirit as pioasm.

**Pseudoinstructions** (1:1 or 1:2 expansions):

| Pseudo | Expansion |
|---|---|
| `NOP [d]` | `JMP ALWAYS, .+1 [d]` |
| `LOOP Rn, lbl` | `JMP RDEC Rn, lbl` |
| `DRIVE p, 0/1/Z` | `SET PINS` / `SET PINDIRS` combinations |
| `CLR Rn` | `ALU MOVI Rn, 0` |
| `BEQ Rn, imm, lbl` | `ALU XOR` (on a scratch copy), then `JMP RZ` |
| `DELAY n` | `SET TIMER, n` then `WAIT TIMER` (for n > 15) |

**Macro library** (multi-instruction protocol primitives):

- `uart_tx` / `uart_rx`
- `spi_xfer mode`
- `i2c_start`, `i2c_stop`, `i2c_wbyte`, `i2c_rbyte ack`
- `jtag_tms_seq`, `jtag_shift`
- `swd_req`, `swd_ack`, `swd_data`
- `usb_rx_packet`, `usb_tx_handshake`
- `can_tx_frame`

This is where the "ease for common protocols" concern is resolved. Users call macros and never see the raw ISA unless they're writing a new protocol.

## 9. What's deliberately excluded

- **Data memory, load/store:** the link layer doesn't need it. Upper layers live on the host.
- **Multiply and divide:** autobaud and timing math need only shifts.
- **In-SM interrupts and exceptions:** they would destroy static timing analysis.
- **Variable-latency anything:** the same reason.
- **Analog PHY functions:** 10BASE-T drive and receive thresholds, USB pull-ups and line levels all stay external.

## 10. Open decisions for the team

1. **Core clock target.** Every cycle budget above assumes about 50 MHz. Run synthesis plus P&R on a skeleton SM early and replace that number with a real one.
2. **SRAM macro availability.** Bank size, count and read latency on IHP via Tiny Tapeout decide whether "one bank per SM" is affordable.
3. **SM count: 2 vs. 4.** Decide after measuring per-SM area.
4. **Ethernet RX hardware.** Supporting it needs DDR input sampling (both clock edges) plus an edge-interval Manchester decoder mode. The alternative is to declare 10BASE-T TX-only. Decide after everything else fits.
5. **GPR width.** 16 bits is assumed. 32 bits costs 64 more flops per SM and simplifies some CRC and match manipulation.
6. **Host link.** SPI is simplest. A parallel interface costs pins that probably aren't available.

## 11. Suggested build order

The risk here is building all of this at once and debugging everything at the same time. Stage it:

1. **Phase 1, PIO-equivalent core:** sequencer, GPRs, OSR/ISR, FIFOs, side-set, delays, `WAIT` on pins, host SPI. Prove UART TX, SPI, I²C, JTAG and SWD end to end.
2. **Phase 2:** bit-timing unit, `SYNCEDGE`, timer and timeouts. Prove UART RX, PS/2 and CAN (without stuffing).
3. **Phase 3:** CRC, stuffer/destuffer, NRZI coder, differential line states. Prove full CAN and LS USB.
4. **Phase 4:** `STREAM`, Manchester, and the Ethernet stretch goal.

**Verification note.** Because the ISA is fully deterministic, a Python or C instruction-set simulator can be **cycle-exact**. That makes it a golden model for constrained-random testing: random legal instruction streams plus random pin stimulus, with the RTL and ISS compared pin-by-pin, every cycle. Layer protocol-level tests on top, with firmware macros running against bus-functional models for each protocol. This is exactly the verification-methodology story Jane Street said they want to see.
