# nisprog — Debug Submenu

The `debug` submenu enables verbose tracing of K-line traffic at each
protocol layer. Use it to diagnose adapter problems, timing issues, framing
errors, and unexpected ECU responses.

Enter the submenu by typing `debug` at `nisprog>`. Use `debug help` for
the built-in list of subcommands.

---

## Debug levels and layers

nisprog uses a layered OSI-style protocol stack:

| Layer | What it covers |
|-------|---------------|
| L0 | Hardware driver — serial port open/close, byte-level TX/RX, baud rate changes |
| L1 | Physical framing — 5-baud init, fast init, break generation, echo removal |
| L2 | Protocol framing — KWP2000 packet assembly, header handling, timing (P1–P4) |
| L3 | Service layer — SID parsing, negative response handling (rarely used) |
| CLI | Command parser — useful for debugging ini file parsing issues |

Each layer has an independent debug level that is a bitmask of event flags.

### Debug flag values

| Flag | Value | Events enabled |
|------|-------|---------------|
| `OPEN` | `0x01` | Port open events |
| `CLOSE` | `0x02` | Port close events |
| `READ` | `0x04` | Byte/message receive events |
| `WRITE` | `0x08` | Byte/message transmit events |
| `IOCTL` | `0x10` | Interface control calls (set speed, set break, etc.) |
| `PROTO` | `0x20` | Protocol state machine events |
| `INIT` | `0x40` | Initialisation sequence events |
| `DATA` | `0x80` | Dump raw data bytes when READ or WRITE is also set |
| `TIMER` | `0x100` | Timer and timing events |

Set to `-1` to enable all flags for a layer. Set to `0` to disable.

---

## Debug subcommands

| Command | Description |
|---------|-------------|
| `debug l0 <val>` | Set L0 (hardware driver) debug level |
| `debug l1 <val>` | Set L1 (physical) debug level |
| `debug l2 <val>` | Set L2 (framing) debug level |
| `debug l3 <val>` | Set L3 (service) debug level |
| `debug cli <val>` | Set CLI parser debug level |
| `debug all <val>` | Set all layers to the same level |
| `debug show` | Print current debug levels for all layers |
| `debug l0test [testnum]` | Run dumb adapter signal tests (see below) |

---

## Common debug recipes

### Full K-line byte trace

Enables hex dumps of every byte sent and received on the K-line at the
physical layer. The most useful setting for diagnosing comms problems:

```
debug l1 0x8C    # READ (0x04) + WRITE (0x08) + DATA (0x80)
```

Then run `nc` — you will see every byte exchanged during the init sequence
and each request/response cycle.

Disable:
```
debug l1 0
```

### Watch the init sequence

```
debug l1 0xCC    # INIT (0x40) + READ (0x04) + WRITE (0x08) + DATA (0x80)
up
nc
```

This shows the exact init pulse timing and the first bytes exchanged.

### Protocol framing trace

```
debug l2 0xAC    # PROTO (0x20) + READ (0x04) + WRITE (0x08) + DATA (0x80)
```

Shows L2 packet assembly and disassembly — useful when the ECU returns
unexpected negative responses or framing errors.

### All layers, full verbosity

```
debug all -1
```

Very verbose. Use only when the above targeted traces are not sufficient.

---

## `debug l0test` — adapter signal tests

`l0test` runs direct hardware tests on the dumb adapter without connecting
to a vehicle. **Disconnect the OBD cable from the vehicle first.**

Switch to the test driver first:
```
set interface DUMBT    # use the test driver (not DUMB)
set port \\.\COM19
set dumbopts 0x48
debug l0test ?         # list available test numbers
```

Then run specific tests:
```
debug l0test 1    # send pulses on TXD (K-line) — check polarity and levels with a meter or scope
debug l0test 2    # send pulses on RTS (L-line) — check L-line driver
debug l0test 3    # DTR test
```

### Interpreting results

- **K-line (TXD):** logic 1 = negative voltage (~−12V on RS-232 side). If your
  meter shows positive voltage when idle, the optocoupler polarity may be
  inverted. Try `LLINE_INV` (`0x10`) in dumbopts.
- **L-line (RTS):** should swing between negative and positive voltage when
  `USE_LLINE` (`0x01`) is active. No swing = L-line not wired or driver fault.
- **Echo check (BLOCKDUPLEX):** send a byte on TXD and check whether RXD echoes
  the same byte. Half-duplex cables will echo every transmitted byte on RXD —
  `BLOCKDUPLEX` (`0x40`) removes these echoes automatically.

After testing, switch back to the real driver:
```
set interface DUMB
```

---

## Diagnosing common problems

### `nc` times out — no response

1. `debug l1 0x8C` + retry `nc` — look for any bytes being received.
2. No bytes at all: cable not plugged in, wrong COM port, or adapter not
   powered. Check `set port` and verify the port in Device Manager.
3. Bytes received but framing wrong: check `dumbopts`. Try `0x68` if `0x48`
   fails.
4. Init bytes go out but ECU does not respond: try `set initmode 5baud` — some
   ECUs need slow init.

### Framing errors / bad duplex

Enable `debug l1 0x8C` and inspect the received bytes. Half-duplex echo not
being removed appears as duplicate bytes in the trace. Ensure `BLOCKDUPLEX`
(`0x40`) is in `dumbopts`.

### Negative response 0x22 (conditionsNotCorrect)

ECU responded but refused the service. Common causes:
- Security access not completed — run `gk` or `setkeys` before the failing command.
- Engine running — some services only work with ignition on and engine off.
- Wrong `addrtype` — use `phys` not `func` for flash/dump.

### Speed mismatch after kernel load

After `runkernel`, comms switch to 62500 bps. If you disconnect and reconnect:
```
set speed 62500    # must set this before nc when kernel is running
```

Without this, L1 will try to communicate at 10400 bps and get garbage.
