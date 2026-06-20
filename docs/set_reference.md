# nisprog — Set Subcommand Reference

Full reference for all `set` subcommands. At the interactive prompt use
`set <param> <value>`; in `.ini` files write bare `<param> <value>` after a
lone `set` line. Use `set show` to print all current values. Use `set <param>
?` for inline help.

---

## `interface` — hardware driver

| Name | Hardware | Notes |
|------|----------|-------|
| `DUMB` | Any dumb serial K-line cable (FT232, CP2102, CH340, optocoupler) | **Use this for all Nissan K-line work.** Anything without an ELM327 chip is DUMB. |
| `ELM` | ELM327-based adapters | Smart interpreted adapter — very slow, flash/dump not supported. |
| `BR1` | B. Roadman BR-1 | Dedicated KWP2000 hardware. Supports J1850/ISO9141/ISO14230. |
| `MET16` | Multiplex Engineering T16 | Multi-protocol lab interface. |
| `DUMBT` | Dumb interface test driver | Not for ECU comms — use `debug l0test` for signal checks. |

**Selection rule:** if the cable has nothing other than a USB-serial bridge
(FT232, CP2102, CH340, PL2303) it is `DUMB`. If it has an ELM327 chip
(marked on PCB or in product listing) use `ELM`. When uncertain, `DUMB` is
almost always correct.

---

## `dumbopts` — dumb adapter option flags

`dumbopts` is a bitmask — set it to the sum of the flags you need.

| Flag | Value | Description |
|------|-------|-------------|
| `USE_LLINE` | `0x01` | Drive the L-line (RTS pin) during 5-baud init. Only needed for cables with a separate L-line wire (some VAGtool clones). |
| `CLEAR_DTR` | `0x02` | Hold DTR low (negative voltage) continuously. Unusual; default is DTR high. |
| `SET_RTS` | `0x04` | Hold RTS high continuously. Unusual. Do not combine with `USE_LLINE`. |
| `MAN_BREAK` | `0x08` | Force software-bitbanged 5bps break pulses. **Essential for USB-serial bridges** (CP2102, CH340, FT232). Enabled in default `0x48`. |
| `LLINE_INV` | `0x10` | Invert L-line polarity. Use only with `USE_LLINE`; very rare. |
| `FAST_BREAK` | `0x20` | Alternate fast-init: send `0x00` at 360 bps (≈25 ms) instead of a hardware break. Try this if `initmode fast` fails. |
| `BLOCKDUPLEX` | `0x40` | Remove echoed bytes at the message level (half-duplex cleanup when P4=0). Enabled in default `0x48`. |

**Common presets:**

| Value | Flags | When to use |
|-------|-------|-------------|
| `0x48` | `MAN_BREAK \| BLOCKDUPLEX` | **Default. Works for virtually all USB K-line cables.** Start here. |
| `0x49` | `MAN_BREAK \| BLOCKDUPLEX \| USE_LLINE` | Cable exposes an L-line that must be driven during 5-baud init. |
| `0x68` | `MAN_BREAK \| BLOCKDUPLEX \| FAST_BREAK` | Fast init fails with `0x48`; try alternate break method. |
| `0x08` | `MAN_BREAK` only | Full-duplex adapter (no echo removal needed). |

**Troubleshooting sequence:**
1. Start with `0x48` — correct for the vast majority of USB K-line cables.
2. If `nc` returns framing errors or no response with `initmode fast`, try `0x68`.
3. If the cable has a separate L-line wire, add `0x01` → `0x49`.
4. Use `set interface DUMBT` + `debug l0test` to send test pulses on K and L
   lines and verify adapter signal levels before connecting to the vehicle.

---

## `l1protocol` — physical layer

Selects hardware framing. For Nissan MEC07 (SH7055) ECUs, set this explicitly
alongside `l2protocol` — relying on auto-selection has been observed to cause
connection failures on some adapters.

| Value | Description |
|-------|-------------|
| `ISO9141` | ISO 9141 physical framing (UART at 10400 bps, 5-baud wake-up). Older ECUs. |
| `ISO14230` | KWP2000 physical framing. **Set this explicitly for Nissan MEC07 K-line.** |
| `J1850-VPW` | SAE J1850 Variable Pulse Width. Pre-2003 GM/Chrysler. Not Nissan. |
| `J1850-PWM` | SAE J1850 Pulse Width Modulation. Pre-2003 Ford. Not Nissan. |
| `CAN` | ISO 15765 CAN. Not used by nisprog (K-line only tool). |
| `RAW` | Raw byte passthrough — no framing. Required for Subaru SSM (`l2protocol raw`). |

Use `set l1protocol ?` to list compiled-in choices.

---

## `l2protocol` — software / framing layer

Must match the ECU's protocol dialect. Use `set l2protocol ?` to list what
is compiled into your binary.

| Value | Description |
|-------|-------------|
| `iso14230` | KWP2000 (ISO 14230-4). **Required for all Nissan ECU operations.** Provides security access (SID 27/36), ReadMemoryByAddress (SID 0xAC), and the service layer that `gk`, `nc`, `flrom`, and `dm` depend on. |
| `raw` | Raw L1 passthrough — no L2 framing. Required for Subaru SSM (`ssmprog.ini`). |
| `iso9141` | ISO 9141-2. Older OBD-II protocol; no security access SID. Diagnostic reads only. |

---

## `initmode` — bus wake-up sequence

Controls how nisprog wakes up the ECU on the K-line.

| Value | Description |
|-------|-------------|
| `fast` | **ISO 14230 fast init.** Pulls K-line low for 25 ms then sends StartCommunications (0xC1 0x33 0xF1). Fast (< 1 s). **Correct for Nissan MEC07 and most modern Nissan ECUs.** |
| `5baud` | ISO 9141 / KWP2000 slow init. Sends ECU address at 5 bps before starting. Slower (~3 s). Use for older ECUs that do not respond to fast init. |
| `carb` | CARB OBD-II functional init. For OBD-II emission scanners. Not used for Nissan flash/dump. |

If `nc` times out or returns no response with `initmode fast`, try `initmode 5baud`.

---

## `testerid` — source address

The source address placed in K-line frame headers. `0xfc` is the standard
tester ID for nisprog and should not need to change.

---

## `destaddr` and `addrtype` — module targeting

### addrtype phys (physical addressing)

Packet targets exactly the module at `destaddr`. Required for flash, dump,
security access, and all standard nisprog operations.

### addrtype func (functional addressing)

Packet is broadcast to address `0x33` (ISO 14230 functional address) regardless
of `destaddr`. All modules supporting the requested service respond. Used by
OBD-II scan tools to discover available modules. Not suitable for flash/dump.

### Known Nissan KWP2000 module addresses (K-line)

| Address | Module |
|---------|--------|
| `0x10` | ECM / PCM — powertrain. **Flash and dump target.** |
| `0x14` | TCM — automatic transmission control |
| `0x15` | ABS / VDC |
| `0x28` | BCM — body control |
| `0x35` | SRS / airbag |

> Addresses vary by model year and market. Use `diag fastprobe 0x10 0x50` to
> scan a range and identify responding modules on your specific vehicle.

### Connecting to other modules (SRS, ABS, BCM)

Change `destaddr` to the module's address and use `diag sr` for KWP2000 reads:

```
set destaddr 0x35    # target SRS/airbag
up
nc
diag sr 0x18 0x02 0xFF 0x00    # read stored DTCs (SID 0x18)
diag sr 0x14 0xFF 0x00         # clear DTCs (SID 0x14)
```

**Flash and dump operations cannot target non-ECM modules.** The npkern flash
kernel only runs in ECM RAM. `flrom`, `flblock`, and fast `dm` only apply to
the powertrain ECM (`destaddr 0x10`).

---

## `speed` — baud rate

Serial port baud rate. Set automatically to 10400 bps for normal K-line
operation. Only change to `62500` when reconnecting to an already-running
kernel after a crash:

```
set speed 62500    # kernel comms baud rate
nc
initk              # re-initialise kernel comms without reuploading
```

See [`crash_recovery.md`](crash_recovery.md) for the full crash reconnect procedure.

---

## AI notes

- **`setdev` accepts `7051`, `7055`, `7058` only.** `7055_18` and `7055_35`
  appear in kernel *filenames* but are not valid setdev arguments. For any
  SH7055 ECU, always `setdev 7055`.
- **`set` prefix at the prompt vs ini:** `set l2protocol iso14230` is correct
  at `nisprog>`. In `.ini` files, the bare `l2protocol iso14230` form (no
  prefix) is correct inside the set submenu block. Mixing the two forms is
  the most common configuration error.
- **Connecting to non-ECM modules for diagnostics** (SRS, ABS) is possible by
  changing `destaddr`. Flash and fast dump operations cannot target non-ECM
  modules — the kernel only runs in ECM RAM.

---

## `show` — display current settings

`set show` prints all current parameters including L0-layer adapter state.
Run after loading an ini file to verify everything is configured correctly
before `nc`.
