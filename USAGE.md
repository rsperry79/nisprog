# nisprog — USAGE

## Purpose

nisprog is an open-source Nissan ECU flash tool for K-line (ISO 14230 /
KWP2000) equipped ECUs. It can dump ROM contents, verify ROM differences,
and reflash ECUs — including selective block flashing to minimise flash
wear. It runs as an interactive CLI and uses freediag for K-line
communications and npkern SH2 kernels for fast flash operations.

**Supported ECU families:** SH7051, SH7055 (1.8 MB and 3.5 MB variants),
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

## Connection Setup

Run these commands in sequence to establish a K-line connection:

```
set interface dumb
set port \\.\COM19          # Windows — adjust COM port
# set port /dev/ttyUSB0    # Linux
set dumbopts 0x48           # required for most dumb K-line adapters
l2protocol iso14230
initmode fast
testerid 0xfc
destaddr 0x10
addrtype phys
up
```

Then connect to ECU:
```
nc
```

---

## Commands

### Connection

| Command | Description |
|---------|-------------|
| `set` | Enter connection settings submenu (see above), or `set <key> <val>` inline |
| `up` | Bring K-line link up |
| `nc` | Connect to ECU (start diagnostic session) |
| `npdisc` | Disconnect from ECU |
| `quit` | Exit nisprog |

---

### ECU identification

| Command | Description |
|---------|-------------|
| `gk` | Guess security access key for this ECU (brute-force from known keyset DB) |
| `setkeys <s27k> <s36k>` | Manually set security keys if `gk` fails (32-bit hex each) |
| `setdev <device>` | Set ECU/flash device type. Options: `7051`, `7055_18`, `7055_35`, `7058` |

**Example:**
```
nc
gk
setdev 7055
```

---

### ROM dump

| Command / Alias | Description |
|-----------------|-------------|
| `dumpmem <file> <start> <len>` | Dump `<len>` bytes from address `<start>` to `<file>` |
| `dm <file> <start> <len>` | Shorthand for `dumpmem` |
| `dm <file> 0 0` | Dump entire ROM (size from `setdev`) |

Dump is slow without npkern. Run `runkernel` first for fast mode.

**Examples:**
```
dm full_dump.bin 0 0
dm partial.bin 0x10000 0x1000
```

---

### Flash kernel (npkern)

| Command | Description |
|---------|-------------|
| `runkernel <path_to_kernel.bin>` | Upload and start SH2 npkern in ECU RAM |
| `stopkernel` | Stop kernel and restart ECU (equivalent to power cycle) |
| `kspeed <baud>` | Set kernel comms baud rate (default 62500) |
| `npconf <param> <val>` | Tune protocol timing parameters (see `npconf ?`) |

**Example:**
```
runkernel D:\ECU-Toolkit\dist\windows\nisprog\npkern\npk_SH7055_35.bin
```

Pick the `.bin` matching your ECU's CPU (see npkern USAGE.md).

---

### ROM verify & flash

| Command | Description |
|---------|-------------|
| `flverif <file>` | Compare ECU flash contents to `<file>`, report differences |
| `flblock <block> <file>` | Flash a single erase block from `<file>` |
| `flrom <file>` | Flash entire ROM from `<file>`, with selective block detection |

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

### Debug

| Command | Description |
|---------|-------------|
| `debug l1 0x8c` | Enable byte-level hex trace of all K-line traffic |
| `debug l1 0` | Disable debug output |

---

## Typical Full Workflow

```
# 1. Connect
set interface dumb
set port \\.\COM19
set dumbopts 0x48
l2protocol iso14230
initmode fast
testerid 0xfc
destaddr 0x10
addrtype phys
up
nc

# 2. Identify and unlock
gk
setdev 7055

# 3. Start flash kernel
runkernel D:\...\npkern\npk_SH7055_35.bin

# 4. Dump ROM (optional backup)
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
# Reconnect at kernel comms speed
set speed 62500
nc
kspeed 62500
# Continue where you left off
```

The kernel keeps running in ECU RAM even after nisprog exits.

---

## AI Summary & Gotchas

**For AI assistants using nisprog:**

- nisprog is an **interactive CLI**, not a scriptable single-shot tool.
  Commands are entered sequentially at its prompt. Batch scripting requires
  piping stdin.
- **`setdev` is mandatory before any flash or dump operation.** Choosing the
  wrong device type (`7051` vs `7055_18` vs `7055_35` vs `7058`) will cause
  incorrect ROM size assumptions and potentially corrupt a flash.
- **`gk` (guess key)** uses a bundled database of known Nissan keysets. If it
  fails, the ECU variant is not in the DB — use `setkeys` with known values.
  Keys are derived from ECUID; nissutils `keyset_lookup.py` can look them up.
- **`flverif` dry-run always reports write errors** — this is expected; it does
  not flash anything. Run it to preview which blocks will change before `flrom`.
- **Never power off the ECU during a flash.** If flash is interrupted, the ECU
  may be unbootable. A running npkern session can be reconnected (see recovery
  above) as long as power is maintained.
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
