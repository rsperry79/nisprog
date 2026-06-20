# nisprog — USAGE

## Purpose

nisprog is an open-source Nissan ECU flash tool for K-line (ISO 14230 /
KWP2000) equipped ECUs. It can dump ROM contents, verify ROM differences,
and reflash ECUs using selective block flashing. It runs as an interactive
CLI, uses freediag for K-line transport, and npkern SH2 kernels for fast
flash and dump operations.

**Supported ECU families:** SH7051 (256 KB), SH7055 (512 KB), SH7058 (1 MB).

**Deep-dive docs** (load as needed):

- [`docs/set_reference.md`](docs/set_reference.md) — interface names, dumbopts flags, l1/l2protocol, initmode, module addresses
- [`docs/security_access.md`](docs/security_access.md) — `gk` vs `setkeys`, s27-only lookup, how security access works
- [`docs/crash_recovery.md`](docs/crash_recovery.md) — recovering from nisprog crash, bad flash, kernel hang
- [`docs/re_workflow.md`](docs/re_workflow.md) — using nisprog for reverse engineering (watch, diag sr, partial dumps)
- [`docs/debug.md`](docs/debug.md) — debug submenu, flag values, l0test, diagnosing comms issues

---

## Starting nisprog

```
nisprog [config_file]
```

Loads `nisprog.ini` from the current directory if no file is given. All
commands are entered at the `nisprog>` prompt.

---

## Connection Settings — Two Syntaxes

### At the interactive prompt

```
set interface DUMB          # adapter driver (DUMB for any simple K-line cable)
set port \\.\COM19          # Windows serial port — Linux: /dev/ttyUSB0
set dumbopts 0x68           # MEC07 confirmed: MAN_BREAK + BLOCKDUPLEX + FAST_BREAK
set l1protocol ISO14230     # set both l1 AND l2 explicitly for MEC07
set l2protocol iso14230     # KWP2000 framing — required for all Nissan ECU commands
set initmode fast           # ISO 14230 fast init — correct for Nissan MEC07
set testerid 0xfc           # source address placed in K-line frame headers
set destaddr 0x10           # destination: 0x10 = powertrain ECM
set addrtype phys           # physical addressing: targets one module only
up                          # bring K-line link up
nc                          # connect — ECUID is printed automatically on success
```

### In nisprog.ini

`set` on its own line enters the set submenu. All lines that follow are bare
subcommands without the `set` prefix — they run in the submenu context until
`up` exits it.

```ini
# Two-syntax rule: "set" alone opens the submenu.
# Inside the submenu, write the subcommand name directly — no "set" prefix.
# At the interactive nisprog> prompt you would type "set l2protocol iso14230";
# here, inside the submenu block, it is just "l2protocol iso14230".
set
port \\.\COM19              # serial port — adjust COM number for your adapter
interface DUMB              # DUMB = any simple K-line cable (not ELM327)
dumbopts 0x68               # MEC07 confirmed: MAN_BREAK + BLOCKDUPLEX + FAST_BREAK
                            # (try 0x48 first if 0x68 fails on your adapter)
l1protocol ISO14230         # set both l1 AND l2 explicitly for MEC07 — do not rely on auto-select
l2protocol iso14230         # KWP2000 framing — required for Nissan
initmode fast               # ISO 14230 fast init for MEC07
testerid 0xfc               # tester source address (standard)
destaddr 0x10               # ECM address — change to 0x15/0x35 for ABS/SRS
addrtype phys               # physical: address one module, not broadcast
up                          # exit submenu and bring K-line link up
```

Use `set show` to verify all current values after loading a config file.

---

## Set — Quick Reference

| Setting | Default | Common values |
|---------|---------|---------------|
| `interface` | `DUMB` | `DUMB` (most cables), `ELM` (ELM327) — see [docs/set_reference.md](docs/set_reference.md) |
| `port` | — | `\\.\COM19` (Win), `/dev/ttyUSB0` (Linux) |
| `dumbopts` | `0x48` | `0x48` (USB K-line), `0x49` (L-line init), `0x68` (alt fast break) |
| `l1protocol` | auto | `ISO14230` for Nissan K-line; rarely set explicitly |
| `l2protocol` | `iso14230` | `iso14230` (Nissan), `raw` (Subaru SSM) |
| `initmode` | `fast` | `fast` (MEC07), `5baud` (older ECUs) |
| `testerid` | `0xfc` | `0xfc` (standard) |
| `destaddr` | `0x10` | `0x10` (ECM), `0x15` (ABS), `0x35` (SRS) — see [docs/set_reference.md](docs/set_reference.md) |
| `addrtype` | `phys` | `phys` (single module), `func` (broadcast scan) |
| `speed` | auto | Only change to `62500` for kernel crash reconnect |

---

## Commands — Quick Reference

### Connection

| Command | Description |
|---------|-------------|
| `nc` / `npconn` | Connect to Nissan ECU. Prints ECUID on success. |
| `spconn` | Connect to Subaru ECU (SSM). |
| `npdisc` / `nd` | Disconnect (does not reset ECU if kernel is running). |
| `source <file>` | Execute commands from a file. |
| `quit` | Exit nisprog. |

### Security and identification

`nc` prints the ECUID automatically — no separate command needed.

| Command | Description |
|---------|-------------|
| `gk` | Guess security key from bundled DB. Prints s27k/s36k on success. |
| `setkeys <s27k> [s36k]` | Set keys manually. s36k optional — omit to look up from DB by s27k. See [docs/security_access.md](docs/security_access.md). |
| `setdev <device>` | Set flash device: `7051` (256 KB), `7055` (512 KB), `7058` (1 MB). **Required before dump or flash.** |
| `writevin <vin>` | Write VIN string to EEPROM. |

### Dump

| Command | Description |
|---------|-------------|
| `dm <file> 0 0` | Dump entire ROM (size from `setdev`). |
| `dm <file> <addr> <len>` | Dump `len` bytes from `addr`. |
| `dm <file> <addr> <len> eep` | Dump from EEPROM (requires `npconf eepr <addr>`). |

Dump without kernel is slow. Run `runkernel` first.

### Flash kernel

| Command | Description |
|---------|-------------|
| `runkernel <file>` | Upload and start npkern in ECU RAM (Nissan). |
| `sprunkernel <file>` | Upload and start SSM kernel (Subaru). |
| `initk` | Re-initialise already-running kernel (use after crash reconnect). |
| `stopkernel` | Stop kernel and restart ECU (power cycle equivalent). |
| `kspeed <baud>` | Change kernel baud rate. Valid: 62500, 31250, 25000. |
| `npconf <param> [val]` | Tune timing/protocol. `npconf ?` lists all params. |

**Kernel file selection (`setdev 7055` — pick by flash cell):**

| File | Cell | ECU examples |
|------|------|--------------|
| `npk_SH7051.bin` | SH7051 | Older Nissan (`setdev 7051`) |
| `npk_SH7055_35.bin` | 350 nm | MEC07-370/390 (VG33/VG33ER) — most mid-2000s Nissan |
| `npk_SH7055_18.bin` | 180 nm | Some SH7055S-based Nissan |
| `npk_SH7058.bin` | 180 nm | Newer Nissan (`setdev 7058`) |

### Flash / verify

| Command | Description |
|---------|-------------|
| `flverif <file>` | Compare ECU flash to file — lists changed blocks, no writes. |
| `flrom <file> [orig]` | Flash ROM. Detects changed blocks by CRC. |
| `flblock <file> <blockno> [Y]` | Flash one block. Add `Y` to skip confirmation. |

> `flverif` / `flrom` / `flblock` require npkern running.

### Diagnostics

| Command | Description |
|---------|-------------|
| `watch <addr>` | Poll and display 4 bytes at address continuously. Enter to stop. See [docs/re_workflow.md](docs/re_workflow.md). |
| `diag sr <byte...>` | Send raw KWP2000 bytes to ECU, print response. |
| `debug <subcommand>` | K-line tracing and adapter tests. See [docs/debug.md](docs/debug.md). |

---

## Workflow INI Files

### `nisprog_dump.ini`

```ini
# nisprog_dump.ini — full ROM dump
# Edit COM port and kernel path before use.
# Recommended: connect a battery tender/charger before starting — a dump takes
# ~10 minutes and a low-voltage dropout mid-session will corrupt the kernel state.
# If the dump is taking more than ~1 hour, stop it and verify setdev matches your
# ECU (7051=256KB, 7055=512KB, 7058=1MB). Reading past the actual ROM size causes
# nisprog to round-robin and become extremely slow.
# Run: nisprog nisprog_dump.ini

set                      # enter set submenu (no "set" prefix on lines below)
port \\.\COM19           # serial port — Windows COM port for K-line adapter
interface DUMB           # driver — DUMB for any simple K-line cable
dumbopts 0x68            # MEC07 confirmed: MAN_BREAK + BLOCKDUPLEX + FAST_BREAK
l1protocol ISO14230      # set both l1 AND l2 explicitly for MEC07
l2protocol iso14230      # KWP2000 framing — required for all Nissan ECU commands
initmode fast            # ISO 14230 fast init — correct for MEC07
testerid 0xfc            # source address in K-line frames
destaddr 0x10            # destination — 0x10 = powertrain ECM
addrtype phys            # physical addressing (target one module, not broadcast)
up                       # exit set submenu and bring K-line link up

nc                       # connect to ECU — prints ECUID on success
gk                       # guess security key — RECORD the s27k/s36k printed here
setdev 7055              # set flash device type (7051/7055/7058 — must match ECU)

# Load flash kernel for fast dump (kernel stays in RAM, does not modify flash)
runkernel D:\ECU-Toolkit\dist\windows\nisprog\npkern\npk_SH7055_35.bin

dm backup.bin 0 0        # dump entire ROM to backup.bin (size from setdev)

stopkernel               # stop kernel — resets ECU (clears self-learn, DTCs)
npdisc                   # disconnect from K-line
```

### `nisprog_flash.ini`

```ini
# nisprog_flash.ini — ROM verification and flash
# BEFORE USE:
#   1. Connect a battery charger — flashing REQUIRES stable voltage (>12.5V).
#      A dropout mid-flash leaves the ECU in a partially written state.
#   2. Edit COM port and kernel path
#   3. Replace setkeys values with s27k/s36k from your gk run
#   4. Replace patched_rom.bin with your ROM filename
#   5. Run: nisprog nisprog_flash.ini

set                      # enter set submenu
port \\.\COM19           # serial port
interface DUMB           # adapter driver
dumbopts 0x68            # MEC07 confirmed: MAN_BREAK + BLOCKDUPLEX + FAST_BREAK
l1protocol ISO14230      # set both l1 AND l2 explicitly for MEC07
l2protocol iso14230      # KWP2000 — required for Nissan
initmode fast            # fast init for MEC07
testerid 0xfc            # tester source address
destaddr 0x10            # ECM target address
addrtype phys            # physical (single-module) addressing
up                       # exit submenu, bring link up

nc                       # connect — confirms communication before proceeding
setkeys 0xXXXXXXXX 0xXXXXXXXX  # supply keys from prior gk run (s27k then s36k)
setdev 7055              # flash device type — must match ECU
runkernel D:\ECU-Toolkit\dist\windows\nisprog\npkern\npk_SH7055_35.bin

flverif patched_rom.bin  # dry run — lists changed blocks without writing anything
flrom patched_rom.bin    # flash — prompts for confirmation per block

stopkernel               # reset ECU (exits kernel, starts ECU from flash)
npdisc                   # disconnect
```

---

## Typical Workflow (interactive)

```
# 1. Set up connection
set interface DUMB
set port \\.\COM19
set dumbopts 0x48
set l2protocol iso14230
set initmode fast
set testerid 0xfc
set destaddr 0x10
set addrtype phys
up
nc                             # ECUID printed here

# 2. Security + device
gk                             # record s27k/s36k
setdev 7055

# 3. Kernel
runkernel D:\...\npk_SH7055_35.bin

# 4. Dump (backup)
dm backup.bin 0 0

# 5. Verify patched ROM
flverif patched_rom.bin

# 6. Flash
flrom patched_rom.bin

# 7. Finish
stopkernel
npdisc
quit
```

---

## Quick notes

- `nc` prints the ECUID automatically — no separate command needed.
- `setdev` accepts `7051`, `7055`, `7058` only — `7055_18`/`7055_35` are kernel filenames.
- `stopkernel` is a power cycle — self-learn (LTFT, idle, VVT, knock) is lost.
- Spaces in file paths break `runkernel` — use paths with no spaces.
- `ssmprog.ini` / `SubaruSIDs.txt` are for Subaru SSM only — not Nissan.
