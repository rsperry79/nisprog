# nisprog — USAGE

## Purpose

nisprog is an open-source Nissan ECU flash tool for K-line (ISO 14230 /
KWP2000) equipped ECUs. It can dump ROM contents, verify ROM differences,
and reflash ECUs — including selective block flashing to minimise flash
wear. It runs as an interactive CLI and uses freediag for K-line
communications and npkern SH2 kernels for fast flash operations.

**Supported ECU families:** SH7051, SH7055 (180 nm and 350 nm variants),
SH7058. Set with `setdev` before flashing.

---

## Starting nisprog

```
nisprog [config_file]
```

Launches the interactive CLI. If no config file is given it looks for
`nisprog.ini` in the current directory. All commands are entered at the
`nisprog>` prompt.

---

## Connection Settings — Two Syntaxes

nisprog has a `set` submenu for all connection parameters. The syntax
differs between the interactive prompt and the `.ini` file.

### At the interactive prompt

At `nisprog>`, prefix every setting with `set`:

```
set interface dumb
set port \\.\COM19          # Windows — adjust COM port
# set port /dev/ttyUSB0    # Linux
set dumbopts 0x48           # required for most dumb K-line adapters
set l1protocol ISO14230     # K-line physical layer (usually not required; auto-selected)
set l2protocol iso14230     # KWP2000 software protocol — required for Nissan ECUs
set initmode fast           # ISO 14230 fast init (5-baud init: use "5baud")
set testerid 0xfc           # source address (tester ID)
set destaddr 0x10           # ECU address
set addrtype phys           # physical addressing (not functional)
```

Bring the link up, then connect:

```
up
nc
```

`nc` automatically retrieves and prints the ECUID (e.g. `ECUID: MEC07-370 C1`)
via KWP2000 SID 0x1A. There is no separate "get ECUID" command — it is always
printed on connect. The ECUID is also used by `gk` for keyset lookup.

### In nisprog.ini (config file)

The ini file is fed line-by-line through the same CLI parser. `set` on its
own line enters the set submenu context. All following lines are executed
as bare subcommands — **no `set` prefix** — until `up` exits the submenu.

```ini
# nisprog.ini — example
set                          # enters the set submenu
port \\.\COM19               # no "set" prefix inside the submenu
interface dumb
dumbopts 0x48
l2protocol iso14230
initmode fast
testerid 0xfc
destaddr 0x10
addrtype phys
up                           # exits the set submenu, brings link up

# nc                         # uncomment to auto-connect on startup
```

This is why `nisprog.ini` uses bare `l2protocol iso14230` while the
interactive prompt requires `set l2protocol iso14230`. They are the same
command — just called from different parser contexts.

---

## Set Subcommand Reference

All of these take effect immediately and are saved for the current session.
Use `set show` to print current values. Use `set <param> ?` for inline help.

---

### `interface` — hardware driver

| Name | Hardware | Notes |
|------|----------|-------|
| `DUMB` | Any dumb serial K-line cable (optocoupler, FT232, CP2102, CH340, etc.) | **Use this for all Nissan K-line work.** Anything that is NOT an ELM327 is probably DUMB. |
| `ELM` | ELM327-based adapters | Smart interpreted adapter. Very slow; flash/dump not supported. |
| `BR1` | B. Roadman BR-1 interface | Dedicated KWP2000 hardware; supports J1850/ISO9141/ISO14230. |
| `MET16` | Multiplex Engineering T16 | Multi-protocol lab interface. |
| `DUMBT` | Dumb interface test/debug driver | Not for ECU comms — use `debug l0test` for signal checks. |

> **Rule of thumb:** if your cable has no chip other than a USB-serial bridge, use `DUMB`. If it has an
> ELM327 chip (check the listing or PCB), use `ELM`. When in doubt, `DUMB` is almost always correct.

---

### `dumbopts` — dumb adapter option flags

`dumbopts` is a bitmask — add the values of the options you need.

| Flag | Value | Description |
|------|-------|-------------|
| `USE_LLINE` | `0x01` | Drive the L-line (RTS pin) during 5-baud init. Only needed for adapters that expose an L-line, e.g. some VAG-type cables. |
| `CLEAR_DTR` | `0x02` | Hold DTR low (negative voltage) continuously. Unusual; default is DTR high. |
| `SET_RTS` | `0x04` | Hold RTS high (positive voltage) continuously. Unusual. Do not combine with `USE_LLINE`. |
| `MAN_BREAK` | `0x08` | Force software-bitbanged 5bps break pulses. **Essential for USB-serial adapters** (CP2102, CH340, FT232). |
| `LLINE_INV` | `0x10` | Invert L-line polarity. Only used with `USE_LLINE`; very rare. |
| `FAST_BREAK` | `0x20` | Alternate fast-init: send `0x00` at 360bps (≈25ms) instead of a hardware break. Try this if `initmode fast` fails with your adapter. |
| `BLOCKDUPLEX` | `0x40` | Remove echoed bytes at the message level (half-duplex cleanup when P4=0). |

**Common combinations:**

| Value | Flags | When to use |
|-------|-------|-------------|
| `0x48` | `MAN_BREAK \| BLOCKDUPLEX` | **Default.** Works for virtually all USB K-line cables with fast init. Start here. |
| `0x49` | `MAN_BREAK \| BLOCKDUPLEX \| USE_LLINE` | Cable has an L-line that must be driven during init (some VAGtool clones). |
| `0x68` | `MAN_BREAK \| BLOCKDUPLEX \| FAST_BREAK` | Fast init fails with `0x48`; try alternate break method. |
| `0x08` | `MAN_BREAK` only | Half-duplex not needed (full-duplex adapter). |

**How to figure out what you need:**
1. Start with `0x48` — this is correct for the vast majority of USB K-line cables.
2. If `nc` fails with framing errors or no response, try `0x68` (alternate fast break).
3. If you have a cable with a separate L-line wire (some older VAGtool types), add `0x01` (`0x49`).
4. Use `debug l0test` (via `set interface DUMBT`) to send test pulses and verify adapter behaviour
   before connecting to the car.

---

### `l1protocol` — physical layer

Selects which hardware framing the adapter uses on the wire. For Nissan K-line
work this is almost always auto-selected by the L2 protocol — you do not normally
need to set this. Set it explicitly only when troubleshooting.

| Value | Description |
|-------|-------------|
| `ISO9141` | ISO 9141 physical framing (UART at 10400 bps). Used with older ECUs and 5-baud init. |
| `ISO14230` | KWP2000 physical framing (same UART as 9141, different init and headers). **Correct for Nissan MEC07 K-line.** |
| `J1850-VPW` | SAE J1850 Variable Pulse Width — used on pre-2003 GM/Chrysler. Not Nissan. |
| `J1850-PWM` | SAE J1850 Pulse Width Modulation — used on pre-2003 Ford. Not Nissan. |
| `CAN` | ISO 15765 CAN bus. Not used with nisprog (K-line tool only). |
| `RAW` | Raw byte passthrough — no framing. Used for Subaru SSM (`l2protocol raw`). |

---

### `l2protocol` — software/framing layer

Selects the protocol dialect used for packet framing and service requests. This
must match what the ECU expects. Use `set l2protocol ?` to list protocols compiled
into your binary.

| Value | Description |
|-------|-------------|
| `iso14230` | KWP2000 (ISO 14230-4). **Required for all Nissan ECU commands.** Provides security access (SID 27/36), ReadMemoryByAddress (SID AC), and the service layer that `gk`, `nc`, `flrom`, etc. depend on. |
| `raw` | Raw L1 passthrough — no L2 framing. Required for Subaru SSM (`ssmprog.ini`). |
| `iso9141` | ISO 9141-2 — older OBD-II protocol, no security access SID. Read-only diagnostics only. |

---

### `initmode` — bus wake-up sequence

Controls how nisprog wakes up the ECU on the K-line before sending any requests.

| Value | Description |
|-------|-------------|
| `fast` | **ISO 14230 fast init.** Pulls K-line low for 25ms then releases for 25ms, followed by StartCommunications (0xC1 0x33 0xF1). Fast, reliable. **Correct for Nissan MEC07 and most modern Nissan ECUs.** |
| `5baud` | ISO 9141 / KWP2000 slow init. Sends the ECU address (0x10) at 5bps before starting comms. Slower (~3 s). Needed for some older ECUs that do not respond to fast init. |
| `carb` | CARB OBD-II functional init. For emission-compliance scanners. Not used for Nissan flash/dump. |

If `nc` times out or returns no response, try switching from `fast` to `5baud`.

---

### `destaddr` and `addrtype` — module targeting

`destaddr` is the KWP2000 destination address — the logical address of the ECU
module you want to talk to. `addrtype` controls whether the packet is addressed
to that one module (`phys`) or broadcast to all modules (`func`).

**`addrtype phys` (physical):** packet targets only the module at `destaddr`.
Used for flash, dump, security access, and everything nisprog does with Nissan
ECUs. This is the correct mode for all flash/dump operations.

**`addrtype func` (functional):** packet uses the ISO 14230 broadcast address
(0x33) regardless of `destaddr`. All modules that support the requested service
will respond. Used for OBD-II emission scan tools to discover which modules are
present. Not suitable for flash/dump.

**Known Nissan KWP2000 module addresses (K-line):**

| Address | Module |
|---------|--------|
| `0x10` | ECM / PCM (powertrain — flash/dump target) |
| `0x14` | TCM (automatic transmission) |
| `0x15` | ABS / VDC |
| `0x28` | BCM / body control |
| `0x35` | SRS / airbag |

> **Addresses vary by model year and market.** The table above covers common
> mid-2000s Nissan/Infiniti K-line vehicles but is not universal. Use `diag
> fastprobe 0x10 0x50` to scan a range of addresses and identify which modules
> respond on your car.

**Can I connect to SRS, ABS, or other modules?**

Yes — for diagnostic reads only. Change `destaddr` to the module address, keep
`addrtype phys`, and use `diag sr` to send KWP2000 requests (DTCs, live data).

```
set destaddr 0x35    # SRS/airbag
nc
diag sr 0x18 0x02 0xFF 0x00    # read stored DTCs
```

**You cannot flash or dump non-ECM modules with nisprog.** The flash kernel
(`npkern`) only runs in ECM RAM. Flash operations (`flrom`, `flblock`) and
ROM dump (`dm`) only apply to the powertrain ECM (`destaddr 0x10`).

---

### `speed` — baud rate

Serial port baud rate for K-line comms. The default (10400 bps for K-line
protocol) is set automatically. Only change this when reconnecting to an
already-running kernel after a crash:

```
set speed 62500    # kernel comms speed
nc
initk
```

---

### `show` — display current settings

`set show` prints all current connection settings and adapter state. Run it
after loading an ini file to confirm everything is set correctly before `nc`.

---

## Commands

### Connection

| Command | Description |
|---------|-------------|
| `set` | Enter connection settings submenu (interactive), or `set <key> <val>` inline |
| `up` | Bring K-line link up |
| `nc` / `npconn` | Connect to Nissan ECU. Prints ECUID on success. |
| `npdisc` / `nd` | Disconnect from ECU (does not reset if kernel is running) |
| `source <file>` | Execute commands from a file (batch mode) |
| `quit` / `exit` | Exit nisprog |

---

### ECU identification

`nc` automatically retrieves and displays the ECUID on connect. No separate
command is needed. The ECUID string is also used internally by `gk`.

| Command | Description |
|---------|-------------|
| `gk` | Guess security access key from bundled keyset DB (uses ECUID). Prints s27k / s36k on success. |
| `setkeys <s27k> <s36k>` | Set security keys manually (32-bit hex each). Use when `gk` fails or re-supplying keys after a session restart. |
| `setdev <device>` | Set flash device type. Valid values: `7051` (256 KB), `7055` (512 KB), `7058` (1 MB). **Must be set before any dump or flash.** |
| `writevin <vin>` | Write VIN string to ECU EEPROM. |
| `watch <addr>` | Continuously poll and display 4 bytes at ROM/RAM address `<addr>`. Press Enter to stop. Uses SID 0xAC (ReadMemoryByAddress) before kernel, or kernel RMBA after. |

---

### ROM dump

| Command | Description |
|---------|-------------|
| `dm <file> <start> <len>` | Dump `<len>` bytes from address `<start>` to `<file>`. |
| `dm <file> 0 0` | Dump entire ROM (size determined by `setdev`). |
| `dumpmem <file> <start> <len>` | Same as `dm` (long form). |
| `dm <file> <start> <len> eep` | Dump from EEPROM address space instead of ROM. Requires `npconf eepr` to be set first. |

Dump without a running kernel is slow (direct K-line). Run `runkernel` first for fast mode.

**Examples:**

```
dm full_dump.bin 0 0
dm partial.bin 0x10000 0x1000
dm eeprom.bin 0 512 eep
```

---

### Flash kernel (npkern)

| Command | Description |
|---------|-------------|
| `runkernel <file>` | Upload and start SH2 npkern in ECU RAM (Nissan). |
| `sprunkernel <file>` | Upload and start SSM kernel in ECU RAM (Subaru). |
| `initk` | Re-initialise an already-running kernel (after reconnect, without reuploading). |
| `stopkernel` | Stop kernel and restart ECU (equivalent to power cycle). |
| `kspeed <baud>` | Change kernel comms baud rate (default 62500). Valid: 62500, 31250, 25000. |
| `npconf <param> <val>` | Tune timing/protocol parameters. Use `npconf ?` to list all. |

**npconf parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `p3` | 5 ms | Delay before new request (ISO P3 timing) |
| `rxe` | 20 ms | Read timeout offset — increase to fix timeout errors on slow adapters |
| `eepr` | 0 | Address of `eeprom_read()` function in ROM — required for EEPROM dump |
| `kspeed` | 62500 | Kernel baud rate used by `initk` |

**Kernel file selection — Nissan (`runkernel`):**

| `setdev` | ROM size | Kernel file | Flash cell | ECU examples |
|----------|----------|-------------|------------|--------------|
| `7051` | 256 KB | `npk_SH7051.bin` | SH7051 | Older Nissan ECUs |
| `7055` | 512 KB | `npk_SH7055_35.bin` | 350 nm | MEC07-370, MEC07-390 (VG33/VG33ER); most mid-2000s Nissan SH7055 ECUs |
| `7055` | 512 KB | `npk_SH7055_18.bin` | 180 nm (SH7055S) | Some newer SH7055S-based Nissan ECUs |
| `7058` | 1 MB | `npk_SH7058.bin` | 180 nm | Newer Nissan ECUs (SH7058) |

**Subaru (`sprunkernel`):** use `ssmk_SH7055_18.bin` (SH7055S) or `ssmk_SH7058.bin` (SH7058) with `ssmprog.ini`.

> **How to pick 350 nm vs 180 nm for `setdev 7055`:** the flash cell variant is a hardware property of
> the SH7055 die, not visible in the ROM file itself. MEC07 ECUs (Nissan VG33/VG33ER) use the 350 nm
> cell — use `npk_SH7055_35.bin`. If the kernel uploads but hangs on the first dump, try the other
> variant. Using the wrong kernel risks corrupting the flash write step — always do a test dump before
> any flash operation.

**Example (MEC07):**

```
runkernel D:\ECU-Toolkit\dist\windows\nisprog\npkern\npk_SH7055_35.bin
```

---

### ROM verify & flash

| Command | Description |
|---------|-------------|
| `flverif <file>` | Compare ECU flash contents against `<file>`, report differing blocks. |
| `flrom <file> [orig_rom]` | Flash entire ROM from `<file>`. Detects changed blocks by CRC. If `orig_rom` is given, uses it to determine which blocks to flash instead. |
| `flblock <file> <blockno> [Y]` | Flash a single erase block `<blockno>` from `<file>`. Add `Y` to skip confirmation. |

**flrom workflow:**

```
# 1. Verify what changed (dry run — does NOT modify ECU)
flverif patched_rom.bin

# 2. Flash only changed blocks
flrom patched_rom.bin
# nisprog will list changed blocks and prompt for confirmation

# 3. Stop kernel when done
stopkernel
```

> ⚠ `flverif` run before `runkernel` is slow. Run after kernel for speed.
> ⚠ `flblock` / `flverif` / `flrom` require npkern to be running.

---

### Raw diagnostic requests (`diag`)

`diag` is a freediag submenu for sending raw ISO 14230 / KWP2000 requests
directly to the ECU. Useful for reading data by identifier, custom SIDs,
or verifying ECU responses at the protocol level.

| Command | Description |
|---------|-------------|
| `diag sr <byte0> [byte1 ...]` | Send raw bytes to the ECU and print response (alias: `sendreq`). |
| `diag read [timeout_s]` | Read ECU response without sending a request. |
| `diag connect` | Low-level L2 connect (normally use `nc` instead). |
| `diag disconnect` | Low-level L2 disconnect. |
| `diag fastprobe [start [end [func]]]` | Scan K-line for ECUs using ISO 14230 fast init. |

**Examples — Nissan KWP2000 SIDs:**

```
# Read ECU ID (SID 0x1A, local ID 0x81) — same as nc performs automatically
diag sr 0x1A 0x81

# Read data by local identifier (SID 0x21)
diag sr 0x21 <local_id>

# Read fault codes (SID 0x18 0x02 0xFF 0x00)
diag sr 0x18 0x02 0xFF 0x00

# Clear fault codes (SID 0x14 0xFF 0x00)
diag sr 0x14 0xFF 0x00

# Tester present / keep-alive
diag sr 0x3E
```

---

### Debug

`debug` is a submenu for enabling verbose K-line tracing. Use to diagnose
comms issues.

| Command | Description |
|---------|-------------|
| `debug l1 <val>` | Set Layer 1 (physical/byte) debug level. `0x8c` = full hex trace; `0` = off. |
| `debug l2 <val>` | Set Layer 2 (framing) debug level. |
| `debug l0 <val>` | Set Layer 0 (driver) debug level. |
| `debug all <val>` | Set all layers to the same debug level. |
| `debug show` | Print current debug levels for all layers. |

---

## `watch` — when and why

`watch <addr>` polls 4 bytes at a ROM or RAM address repeatedly until you press
Enter, printing the value each loop. Before the kernel is loaded it uses KWP2000
SID 0xAC (ReadMemoryByAddress). After `runkernel` it uses the faster kernel RMBA
protocol.

**Use cases:**

- **Monitor a RAM flag or counter live.** For example, watch a knock counter
  address while revving the engine to see when and how much knock is detected.
- **Verify a cal table value is being read from the address you expect.** Load
  the kernel, watch the address, change operating conditions — if the value
  changes, you've found the right live RAM location.
- **BBRAM / self-learn monitoring.** Watch LTFT, STFT, idle relearn values, or
  VVT learning flags update during an idle relearn cycle.
- **Debug a patch.** After flashing a changed table, watch the RAM mirror of
  that table to confirm the ECU is actually using the new values.

```
# Example: watch 4 bytes at RAM address (kernel must be running for fast polls)
runkernel D:\...\npk_SH7055_35.bin
watch 0xFFFF6000
```

---

## Workflow ini Files

Save these alongside `nisprog.exe` for one-command session setup.

### `nisprog_dump.ini` — ROM backup

```ini
# nisprog_dump.ini
# Sets up connection and dumps the full ROM.
# Edit COM port and kernel path for your setup.
set
port \\.\COM19
interface DUMB
dumbopts 0x48
l2protocol iso14230
initmode fast
testerid 0xfc
destaddr 0x10
addrtype phys
up
nc
# gk runs here — record the s27k/s36k printed in the output
gk
setdev 7055
runkernel D:\ECU-Toolkit\dist\windows\nisprog\npkern\npk_SH7055_35.bin
dm backup.bin 0 0
stopkernel
npdisc
```

### `nisprog_flash.ini` — ROM flash

```ini
# nisprog_flash.ini
# Verifies and flashes a patched ROM.
# Run after editing: change COM port, kernel path, setkeys values, and ROM filename.
set
port \\.\COM19
interface DUMB
dumbopts 0x48
l2protocol iso14230
initmode fast
testerid 0xfc
destaddr 0x10
addrtype phys
up
nc
setkeys 0xXXXXXXXX 0xXXXXXXXX    # replace with values from gk output
setdev 7055
runkernel D:\ECU-Toolkit\dist\windows\nisprog\npkern\npk_SH7055_35.bin
flverif patched_rom.bin
flrom patched_rom.bin
stopkernel
npdisc
```

> Run via: `nisprog nisprog_dump.ini` or `nisprog nisprog_flash.ini`

---

## Typical Full Workflow

```
# 1. Connect (interactive prompt — all set subcommands need "set" prefix)
set interface dumb
set port \\.\COM19
set dumbopts 0x48
set l2protocol iso14230
set initmode fast
set testerid 0xfc
set destaddr 0x10
set addrtype phys
up
nc                                   # nc prints ECUID on success

# 2. Identify, record key, and set device
gk                                   # prints s27k/s36k — save these values
setdev 7055                          # 512 KB ROM

# 3. Start flash kernel
runkernel D:\...\npkern\npk_SH7055_35.bin

# 4. Dump ROM (backup — also verifies kernel is working)
dm backup.bin 0 0

# 5. Verify your patched ROM
flverif patched_rom.bin

# 6. Flash
flrom patched_rom.bin

# 7. Done — stop kernel (restarts ECU)
stopkernel
npdisc
quit
```

---

## Recovering from a crashed session

If nisprog or your machine crashed while npkern was running:

```
# Reconnect at kernel comms speed (do NOT reinitialise the bus — kernel is still running)
set speed 62500
nc
initk                                # re-initialises kernel comms without reuploading
# Continue where you left off
```

The kernel keeps running in ECU RAM even after nisprog exits as long as ECU power is maintained.

If `initk` fails, the kernel may have crashed. Try a ROM dump first to verify comms before
attempting any flash operation.

---

## AI Summary & Gotchas

**For AI assistants using nisprog:**

- nisprog is an **interactive CLI**, not a scriptable single-shot tool.
  Commands are entered sequentially at its prompt. Batch scripting requires
  piping stdin or using `source <file>`.

- **ECUID is printed automatically by `nc`.** There is no separate "get ECUID"
  command. The ECUID string (e.g. `MEC07-370 C1`) is retrieved via KWP2000
  SID 0x1A on connect and is also used internally by `gk` for keyset matching.

- **`set` prefix is required at the interactive prompt for all connection
  parameters.** `set l2protocol iso14230` is correct at `nisprog>`. The
  bare form `l2protocol iso14230` (without `set`) is only valid inside the
  `.ini` file after a lone `set` line has entered the set submenu context.
  Mixing the two forms is the most common configuration error.

- **`l1protocol` vs `l2protocol`:**
  - `l1protocol` is the physical/hardware layer (how bits are encoded on the
    wire). For Nissan K-line: `ISO14230`. Usually auto-selected and does not
    need to be set explicitly unless the adapter requires it.
  - `l2protocol` is the software/framing layer. **Must be `iso14230`** for
    all Nissan ECU commands (`nc`, `gk`, `runkernel`, `flrom`, etc.).
    nisprog will refuse to run ECU commands if L2 is not `iso14230`.

- **`setdev` is mandatory before any flash or dump operation.** Valid values
  are `7051`, `7055`, `7058`. The `7055_18` / `7055_35` notation appears in
  kernel filenames but NOT in `setdev` — always use `setdev 7055` for SH7055
  ECUs regardless of flash cell variant.

- **Always run `gk` before a dump and record the key it finds.** Security
  access is required for both dump and flash operations. `gk` prints the
  discovered s27k / s36k pair — save these values. On some ECU variants the
  key is not cached between sessions, so you must re-supply it with `setkeys
  <s27k> <s36k>` before flash operations (`flverif`, `flblock`, `flrom`) will
  succeed. If you skip `gk` or lose its output, the write will fail with a
  security-access rejection that can look like a comms error. If `gk` fails
  (ECU not in the keyset DB), use `setkeys` with values from nissutils
  `keyset_lookup.py`.

- **`flverif` dry-run always reports write errors** — this is expected; it does
  not flash anything. Run it to preview which blocks will change before `flrom`.

- **Never power off the ECU during a flash.** If flash is interrupted, the ECU
  may be unbootable. A running npkern session can be reconnected with `initk`
  (see recovery above) as long as power is maintained.

- **`stopkernel` = power cycle.** DTCs, self-learn (LTFT/STFT, idle relearn,
  VVT), and knock learning are lost. Budget a drive cycle for relearning after
  a flash.

- **Spaces in filenames are not supported** by `runkernel`. Use paths with no
  spaces.

- `nisprog.ini` sets default port, interface, and timing. Edit it to avoid
  retyping `set` commands each session.

- `ssmprog.ini` is the equivalent config for Subaru SSM protocol sessions.

- `SubaruSIDs.txt` is a reference table for Subaru Service IDs — only relevant
  for SSM/Subaru sessions, not Nissan.
