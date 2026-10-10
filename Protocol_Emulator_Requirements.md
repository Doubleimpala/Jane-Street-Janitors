# Protocol Emulator: The Basic Parts We Need

This is a starting point for the design, before choosing an architecture or instruction set. The [ISA plan](Protocol_Emulator_ISA_Plan.md) is still a separate proposal.

## 1. The main idea

The emulator should be built from reusable parts for moving data, reading and driving pins, keeping time, and choosing what to do next.

A protocol is a set of settings **plus rules for using those parts**. UART, I²C, and a custom protocol should use the same basic parts with different settings and rules.

For example, a baud-rate setting tells us how long a UART bit lasts. It does not tell us to send a start bit, then data, then a stop bit. We also need a way to describe that sequence.

We have not decided how to store or run those rules. They could be a table of steps, a small program, or another form of control.

## 2. Start with UART

UART normally has separate TX and RX lines. There is no shared clock line: both ends use an agreed baud rate. TX means transmit; RX means receive.

For a basic 8-bit UART link, we need:

| Part or behavior | What it does |
|---|---|
| TX buffer | Holds bytes waiting to be sent. |
| RX buffer | Holds received bytes until the host reads them. |
| TX shift register | Holds the current byte and sends one bit at a time. |
| RX shift register | Builds a byte from bits read one at a time. |
| Timer | Sets the bit length from the baud rate. |
| Input edge detector | Notices when RX changes from high to low, which may be a start bit. |
| Bit counter | Counts how many data bits have been sent or received. |
| Pin control | Drives TX and reads RX. |
| Sequence control | Handles idle, start bit, data bits, optional parity, and stop bits. |
| Status | Reports received data, busy state, and errors. |

A **buffer** holds waiting data. A **shift register** holds the data currently being sent or received.

Sending a byte looks like:

```text
Take a byte from the TX buffer
    → send a low start bit
    → send the data bits, usually lowest bit first
    → send parity if enabled
    → send a high stop bit
    → send the next byte or stay idle
```

Receiving a byte looks like:

```text
Wait for a possible start bit
    → check it is still low near its middle
    → read each data bit near its middle
    → check parity if enabled
    → check the stop bit
    → put the byte and any error result into the RX buffer
```

**UART settings:** TX/RX pins, baud rate, number of data bits, bit order, parity, and number of stop bits.

If TX and RX must run at the same time, both need their own progress and timing state. That does not yet decide whether they use separate hardware or share hardware that is fast enough.

## 3. Now look at I²C

I²C uses two shared lines: SDA for data and SCL for the clock. A device drives a line low or releases it. A pull-up resistor brings a released line high when no device holds it low.

Start with the controller role: our emulator starts transfers and controls the clock.

| Part or behavior | What it does | Can we reuse a UART part? |
|---|---|---|
| TX and RX buffers | Hold outgoing and incoming bytes. | Yes. |
| TX and RX shift registers | Send or collect one bit at a time. | Yes, with a different bit order. |
| Bit counter | Counts the eight data bits and tracks the ACK slot. | Yes. |
| Timer | Sets how long SCL stays low and high. | Yes, with different timing rules. |
| Input reader and edge detector | Observe SDA and SCL. | Yes. |
| Pin control | Pulls SDA/SCL low or releases them. | Extend it to support release and readback. |
| Sequence control | Handles START, address, data, ACK, and STOP. | Same purpose, different rules. |
| ACK check | Reads whether the other device accepted a byte. | Use an input bit to choose the next step. |
| Clock-stretch wait | Waits if another device holds SCL low after we release it. | Needs waiting on a pin, not just a fixed delay. |
| Bus-loss check, if supported | Stops if we release SDA high but read it low during a bit where we must check who owns the bus. | Needs comparing the intended bit with the actual pin. |

Writing a byte looks like:

```text
Generate START and send the address
    → check address ACK
    → take a byte from the TX buffer
    → set SDA while SCL is low
    → release SCL and wait until it reads high
    → complete the high part of the clock pulse
    → pull SCL low and repeat for the next bit
    → release SDA and read ACK on the ninth clock pulse
    → send another byte or generate STOP
```

For ordinary data bits, SDA stays steady while SCL is high. START and STOP are special changes of SDA while SCL is high.

**I²C settings:** SDA/SCL pins, clock timing, bit order, address, read/write direction, byte count, and any wait timeout.

**I²C rules:** when to generate START/STOP, when to drive or release SDA, how to handle ACK, and what to do if the bus does not respond.

An I²C target uses many of the same parts, but follows the incoming clock, checks its address, and responds to the controller. Supporting that role needs its own rules.

## 4. SPI gives us another useful check

SPI has a shared clock, separate data lines in the common case, and a select line. It can send and receive a bit during the same clock period.

It reuses buffers, shift registers, counters, timing, and pin control. Its settings choose the clock's idle level, which edge reads data, which edge changes data, and the bit order. Its rules control selection and transfer length.

This checks that our parts can read and write together, and act on either edge of a clock.

## 5. The reusable parts

These are jobs the design needs to support. They do not have to become one hardware block each.

| Part | Basic operations it should support | Things we can configure |
|---|---|---|
| Host connection | Load settings and rules, write TX data, read RX data and status, start or stop work. | How the host connects; exact commands are undecided. |
| Data buffers | Store and return data in order; report full or empty. | How data is grouped; size is undecided. |
| Shift registers | Load data, send one or more bits, collect bits, return collected data. | Bit order and number of bits. |
| Pin control | Read, drive low, drive high, release, and coordinate changes on multiple pins. | Pin choice, idle value, and allowed drive mode. |
| Input checks | Detect high/low, rising/falling edges, and compare a read value with an expected value. | Pin, edge, or value to check. |
| Timing | Wait a set time, create repeated bit timing, or follow an incoming clock. | Bit period, pulse lengths, sample time, and timeout. |
| Counters and saved values | Count bits or bytes and remember progress or received fields. | Starting count and values to compare. |
| Sequence control | Run steps, repeat them, wait for an event, and choose the next step based on a result. | The rules that define the protocol. |
| Status and errors | Report done, received data, failed checks, timeouts, and buffer problems. | Which events to report and how to recover. |

Some protocols also need parity, CRC, or changes to how bits appear on the wire. A CRC is a check value calculated from the data to help detect errors. We can later decide which extra operations deserve their own hardware and which can use the basic parts.

Reusing a type of part does not always mean sharing one copy. Two transfers running at once may need separate buffers, counters, or saved state.

## 6. How these parts cover the main jobs

| Main job | What the reusable parts need to do |
|---|---|
| Accept work and configuration | Receive settings, sequence rules, and data from the host. Keep those settings steady during a transfer. |
| Observe the pins | Read levels, notice edges, and read bits at the right time. |
| Understand what was read | Collect bits into values, count progress, and check expected bits or fields. The rules give those values meaning. |
| Choose what happens next | Continue, repeat, wait, respond, finish, or stop after an error. |
| Act on the pins | Drive or release the selected pins at the right time. |
| Return data and results | Put received data in the RX buffer and report completion or errors. |

These jobs repeat during a transfer. They are not necessarily separate pipeline stages like a CPU's fetch, decode, and execute.

## 7. What a protocol configuration contains

A complete configuration needs three things:

1. **Settings:** pins, timing, bit order, data length, and idle/drive behavior.
2. **Rules:** the order of actions, what to wait for, what to check, and what to do for each result.
3. **Data:** bytes to send, expected values, or prepared replies.

For a simple custom protocol, we might choose:

```text
Pins: DATA and READY
Timing: one bit every 10 microseconds
Data: 12 bits, highest bit first

Rules:
    Wait until READY is high
    Send a low start pulse on DATA for one bit period
    Send the 12 data bits
    Return DATA high
    Report done
```

That uses the same pin control, timer, shift register, counter, and sequence control as the standard protocols. We should be able to load new settings and rules without changing RTL, as long as the protocol fits the chip's pins, timing, storage, and supported operations.

This makes flexible rules a core requirement. A menu containing only “UART”, “I²C”, and “SPI” would not let us describe this custom protocol.

## 8. Basic rules the design must get right

- Read and change pins at the times specified by the configuration and rules.
- Allow decisions based on input pins and received data without waiting for the host each time.
- Keep data in order and report when a TX buffer is empty or an RX buffer is full. Define what happens to the transfer in either case.
- Define how to stop, reset, or recover, including what happens to driven pins.
- Make settings changes take effect at a clear point, such as before the next transfer.
- Support simultaneous TX/RX if we decide that is required.

## 9. What we should decide next

First, try writing UART, I²C, SPI, and one custom protocol using the same basic operations:

```text
Read a pin                 Drive or release a pin
Wait for time or an event  Send or collect bits
Count                      Compare
Repeat                     Choose the next step
Move data to/from buffers  Report a result
```

If an example cannot be expressed, identify the missing operation. Then choose target speeds, buffer sizes, and how many things must run at once.

After that, we can decide how to build the control and data paths, whether to use instructions, and whether operations take one cycle or several. The first goal is to agree on the reusable jobs and the rules needed to combine them.
