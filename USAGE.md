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

| Setting | Values | Notes |
|---------|--------|-------|
| `interface <name>` | `dumb`, `br_l0`, `br_l1`, `scl`, `elm`, `me` | Use `set interface ?` to list available drivers |
| `port <port>` | `\\.\COM19` (Win), `/dev/ttyUSB0` (Linux) | Serial port for K-line adapter |
| `dumbopts <hex>` | `0x48` | Required for most dumb K-line adapters; sets timing options |
| `l1protocol <name>` | `ISO9141`, `ISO14230`, `J1850-VPW`, `J1850-PWM`, `CAN`, `RAW` | Hardware (physical layer) protocol. For Nissan K-line use `ISO14230`. Usually auto-selected by the L2 protocol choice; only set explicitly when troubleshooting. |
| `l2protocol <name>` | `iso14230`, `raw`, `iso9141`, `j1850-vpw`, `j1850-pwm` | Software protocol. **Must be `iso14230` for Nissan ECU commands.** Use `set l2protocol ?` to list compiled-in choices. |
| `initmode <mode>` | `fast`, `5baud`, `carb` | ISO 14230 bus init sequence. `fast` (0xC1 0x33 0xF1) is correct for Nissan MEC07. |
| `testerid <hex>` | `0xfc` | Source address sent in K-line frames (tester ID). `0xfc` is standard. |
| `destaddr <hex>` | `0x10` | ECU destination address. `0x10` for Nissan powertrain ECUs. |
| `addrtype <type>` | `phys`, `func` | Physical addressing targets one ECU; functional broadcasts. Use `phys`. |
| `speed <baud>` | `62500` | Comms baud rate. Only change for kernel reconnect after crash (see below). |
| `show` | — | Print all current settings. |

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
| `watch <addr>` | Continuously display 4 bytes at ROM/RAM address `<addr>`. |

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
