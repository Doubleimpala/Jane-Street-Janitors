# Protocol Emulator Build Checklist

Build and check the small parts first. Connect them in the top level after their inputs, outputs, and behavior are clear.

This follows the [basic requirements](Protocol_Emulator_Requirements.md). Sizes, instruction format, and the final architecture are still open. The ISA plan is a proposal, not a required implementation.

## 1. Host communication and instruction deserializing

- [ ] Define how the host talks to the chip.
  - [ ] Choose the host link and its pins.
  - [ ] Define how a message starts and ends.
  - [ ] Define bit order, word size, and command/data format.
  - [ ] Define commands for loading settings and rules, writing TX data, reading RX data, and starting/stopping work.
  - [ ] Define replies for accepted commands, invalid commands, and busy/full conditions.
- [ ] Build the input deserializer: turn incoming bits into complete words.
  - [ ] Count received bits and assemble the word in the agreed order.
  - [ ] Mark when a complete word is ready.
  - [ ] Keep that word stable until the next block accepts it.
  - [ ] Define what happens to a partial word on reset or an interrupted message.
- [ ] Build the command reader.
  - [ ] Separate commands, settings, rules/instructions, and payload data.
  - [ ] Check command fields before accepting them.
  - [ ] Prevent new data from overwriting a word that has not been accepted.
- [ ] Build the reply/output serializer.
  - [ ] Return received data, status, and command replies.
  - [ ] Define what to return when no reply or data is ready.
- [ ] Check the host blocks.
  - [ ] Send known words and check the reconstructed values and bit order.
  - [ ] Try back-to-back messages and a receiver that is temporarily busy.
  - [ ] Try invalid messages, interrupted messages, and reset halfway through a word.

## 2. Synchronizers, edge detectors, and clock divider

- [ ] Decide which inputs come from outside the chip's clock.
  - [ ] Choose how each input crosses into the core clock.
  - [ ] Document how many cycles pass before a pin change becomes visible.
  - [ ] Check that the chosen method can capture the fastest required input changes.
  - [ ] Do not synchronize each bit of a changing multi-bit word independently; transfer it as a complete word.
- [ ] Build input synchronizers where needed.
  - [ ] Give them defined reset values.
  - [ ] Use the synchronized values in the blocks that follow them.
- [ ] Build rising-edge and falling-edge detectors.
  - [ ] Produce one event per detected edge.
  - [ ] Avoid false events just after reset.
- [ ] Build the clock divider / timing tick generator.
  - [ ] Define the divider setting and the time between ticks.
  - [ ] Use a one-cycle enable tick on the core clock where possible.
  - [ ] Define behavior for zero, minimum, and maximum settings.
  - [ ] Define when a new setting takes effect and how timing restarts.
  - [ ] Decide whether whole-cycle timing is enough or fractional timing is needed.
- [ ] Check edge detection and tick spacing at different input phases and settings.

## 3. Buffers, registers, timers, and other small blocks

### Data buffers

- [ ] Choose TX and RX buffer widths and depths.
- [ ] Build ordered writes and reads.
- [ ] Provide full, empty, and data-ready signals.
- [ ] Handle a read and write in the same cycle.
- [ ] Define behavior for a write while full and a read while empty.
- [ ] Check ordering, wraparound, full/empty boundaries, and reset.

### Shift registers

- [ ] Build loading and shifting of outgoing data.
- [ ] Build collecting and reading incoming data.
- [ ] Support the required bit orders and bit counts.
- [ ] Define which bit is sent or stored first.
- [ ] Define what happens if load and shift are requested together.
- [ ] Check known patterns, short transfers, and simultaneous send/receive if required.

### Settings and saved values

- [ ] List settings: pin choices, timing, bit order, lengths, and idle values.
- [ ] Choose widths and reset values.
- [ ] Add storage for counters, received fields, and values used in comparisons.
- [ ] Define which values the host may change while a transfer runs.
- [ ] Make new settings take effect at a clear transfer boundary.

### Counters and comparisons

- [ ] Build bit/byte counters with load, step, and done signals.
- [ ] Define zero-count and maximum-count behavior.
- [ ] Compare received values with expected values.
- [ ] Compare a driven/released bit with the actual input when needed.
- [ ] Check the final bit is handled exactly once.

### Timer and timeouts

- [ ] Build a timer with load, start, stop, and expired signals.
- [ ] Define the time unit and whether it uses core cycles or timing ticks.
- [ ] Support a fixed delay and a timeout while waiting for an input.
- [ ] Define what happens if the input event and timeout arrive together.
- [ ] Check zero, one-tick, maximum, restart, and cancel cases.

### Pin control

- [ ] Support reading, driving low, driving high, and releasing a pin where the pad allows it.
- [ ] Support open-drain behavior: low means drive; high means release.
- [ ] Keep input readback available while controlling a shared line.
- [ ] Coordinate changes on more than one pin when required.
- [ ] Define safe pin values and output enables for reset, idle, and stop.
- [ ] Reject pin selections or drive modes the physical pins cannot support.
- [ ] Check drive values and output enables together.

### Extra data checks, if needed

- [ ] Decide which checks the first protocol examples require.
- [ ] Add parity generation/checking if needed.
- [ ] Add CRC generation/checking if needed.
- [ ] Add bit encoding or stuffing only when a selected example needs it.
- [ ] Check each added operation against known input/output examples.

## 4. Rules and individual control operations

Define these operations before deciding how the final scheduler runs them.

- [ ] Choose how the host describes a sequence: instructions, a table of steps, or another format.
- [ ] Define the basic operations.
  - [ ] Read a pin; drive or release a pin.
  - [ ] Wait for a time, level, or edge.
  - [ ] Send or collect bits.
  - [ ] Move data to/from buffers and registers.
  - [ ] Count, compare, repeat, and choose the next step.
  - [ ] Finish or report an error.
- [ ] Define each operation's inputs, outputs, and completion signal.
- [ ] Define when a waiting operation finishes or is cancelled.
- [ ] Decide how rules are stored and loaded.
- [ ] Define behavior for invalid rules or out-of-range accesses.
- [ ] Check that the rules can describe a custom protocol without changing RTL.

## 5. Example sequences and expected behavior

Prepare these before full integration so we know what the finished design must do.

- [ ] Write UART TX and RX sequences.
  - [ ] Include start/data/stop bits, sample timing, and framing errors.
- [ ] Write an I²C controller write/read sequence.
  - [ ] Include START/STOP, ACK/NACK, drive/release, and clock-stretch waiting.
- [ ] Write an SPI transfer sequence.
  - [ ] Include clock edge choices and sending/receiving together.
- [ ] Write one small custom protocol using the same operations.
- [ ] Choose initial rates, transfer lengths, and host data rates for these examples.
- [ ] Record expected pin waveforms, received data, and completion/error results.

## 6. Top level, scheduling, and instrumentation — last

### Top-level wiring

- [ ] Finalize the host and protocol pin map.
- [ ] Connect the checked blocks and replace placeholder outputs.
- [ ] Make clock, reset, and enable behavior consistent across blocks.
- [ ] Check that all outputs and output enables have defined values.
- [ ] Define who may control each pin and prevent conflicting requests.

### Scheduling and running sequences

- [ ] Connect operation selection, saved progress, and next-step control.
- [ ] Start the next operation only when its required data and parts are ready.
- [ ] Handle waits, buffer full/empty conditions, completion, stop, and reset.
- [ ] Decide whether TX/RX or separate sequences must run at the same time.
- [ ] If parts are shared, check that sharing does not miss pin timing or lose state.
- [ ] Measure input-to-response delay and check it against the example requirements.

### Instrumentation and status

- [ ] Expose busy, done, buffer state, timeout, and error results to the host.
- [ ] Define how status is read and cleared without losing new events.
- [ ] Add useful simulation signals such as current step, counters, and wait reason.
- [ ] Choose any debug pins or event counters needed for hardware bring-up.

### Full-design checks

- [ ] Run the example sequences from host command through to pin activity and returned results.
- [ ] Check timing as well as data values.
- [ ] Try delayed host data, full RX buffers, bad input, and reset during a transfer.
- [ ] Check simultaneous transfers if supported.
- [ ] Run lint and synthesis; review warnings, area, and achievable clock speed.
- [ ] Run the required implementation checks before claiming the chosen rates work on hardware.
- [ ] Document supported settings, limits, host commands, and example usage.
