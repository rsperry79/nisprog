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
set dumbopts 0x48           # adapter flags: 0x08 MAN_BREAK + 0x40 BLOCKDUPLEX
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
dumbopts 0x48               # USB adapter flags: MAN_BREAK + BLOCKDUPLEX
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
| `npk_SH7051.bin` | SH705