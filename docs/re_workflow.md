# nisprog — Using nisprog for Reverse Engineering

nisprog provides several capabilities useful during ECU reverse engineering:
targeted memory reads, live RAM monitoring, raw KWP2000 request injection,
and EEPROM access. None of these modify the ECU.

---

## Bench wiring (NS-H11B 112-pin connector)

Minimum pins to power the ECU and establish K-line communication off the vehicle:

| Pin | Function |
|-----|----------|
| 38 (A38) | Battery +12V constant |
| 47 (B7) | Ignition +12V — bridge to +12V to simulate key-on |
| 60 (B20) | ECU Ground |
| 109 (C29) | ECU Ground |
| 116 (C36) | Sensor Ground — required for sensor inputs |
| **14 (A14)** | **K-Line (Consult/OBD) — connect your K-line adapter here** |

K-line adapter connects to Pin 14 (A14). All nisprog operations go through this line.

**Immobilizer (NATS and equivalents):** Some ECUs require an immobilizer confirm
signal before allowing injection. On bench, the ECU will connect via K-line but
will not inject (and may behave erratically) without the confirm. Either loop the
confirm wire or use the immobilizer unit from the same vehicle. Check the FSM
wiring diagram for your ECU to determine if an immobilizer is present.

---

## ROM dump for offline analysis

A full dump is the starting point for any RE session. Plan for ~10 minutes for
a 512 KB ROM via K-line at 10400 baud.

> **If a dump runs for more than ~1 hour**, stop it and check `setdev`. Reading
> past the actual ROM size causes nisprog to round-robin and become extremely
> slow. `setdev 7055` = 512 KB, `setdev 7051` = 256 KB, `setdev 7058` = 1 MB —
> the value must match your ECU or the dump length will be wrong.

```
nc
gk
setdev 7055
runkernel npkern\npk_SH7055_35.bin
dm rom_original.bin 0 0           # full dump — length 0 means "full ROM size"
# or with explicit byte count:
dm rom_original.bin 0 524288      # same result, 512 KB = 0x80000 bytes
```

Then load `rom_original.bin` into Ghidra for static analysis.
Partial dumps are faster for targeted reads during active RE:

```
dm cal_zone.bin 0x10000 0x8000    # dump 32 KB at offset 0x10000
dm ivt.bin 0x0 0x200              # dump interrupt vector table (512 B)
```

---

## Live RAM reading: `watch`

`watch <addr>` polls 4 bytes at a ROM or RAM address continuously and prints
the value each loop. Press Enter to stop.

Before the kernel is loaded it uses KWP2000 SID 0xAC (ReadMemoryByAddress —
Nissan proprietary). After `runkernel` it uses the faster kernel RMBA protocol.

### Use cases

**Monitor a RAM variable live:**
```
# Watch a suspected knock counter or status flag at 0xFFFF6000
runkernel npkern\npk_SH7055_35.bin
watch 0xFFFF6000
# Rev the engine and observe the value change
```

**Confirm a table address is live:**
After identifying a cal table in Ghidra, watch its RAM mirror while changing
operating conditions to verify the ECU is reading from that address:
```
watch 0xFFFF7200    # suspected LTFT base table in RAM
```

**BBRAM / self-learn monitoring:**
Watch LTFT, STFT, idle relearn, VVT target values updating during an idle
relearn cycle:
```
watch 0xFFFF8100    # suspected LTFT cell
```

**Patch verification:**
After flashing a changed table, watch the RAM mirror to confirm the ECU is
using the new values at runtime:
```
watch 0xFFFF6A00    # patched timing table
```

### Targeted range dump (faster than watch for larger areas)

```
# Dump 256 bytes of RAM around an address of interest
dm ram_sample.bin 0xFFFF6000 0x100
```

---

## Raw KWP2000 requests: `diag sr`

`diag sr` sends raw ISO 14230 bytes to the ECU and prints the full response.
Useful for probing vendor-specific SIDs, reading data by local identifier, and
testing ECU responses without writing any driver code.

### Reading data by local identifier (SID 0x21)

```
nc
diag sr 0x21 0x01     # read local ID 0x01 (varies by ECU)
diag sr 0x21 0x10     # read local ID 0x10
```

### ReadMemoryByAddress (SID 0xAC) — manual form

The same SID that `watch` uses internally:

```
# Read 4 bytes at address 0xFFFF6000
# SID 0xAC, addr high-byte first, then length
diag sr 0xAC 0xFF 0xFF 0x60 0x00 0x04
```

### Read ECU ID (SID 0x1A)

```
diag sr 0x1A 0x81    # same request nc sends automatically; response = ECUID bytes
```

### Read stored DTCs (SID 0x18)

```
diag sr 0x18 0x02 0xFF 0x00    # readDiagnosticTroubleCodes, all, all
```

### Tester present / keep-alive

```
diag sr 0x3E    # send every ~4 s to prevent session timeout
```

### Probing unknown SIDs

```
# Try a vendor-specific SID
diag sr 0x1A 0x82    # alternate local ID — observe whether ECU responds or NRC 0x11
diag sr 0x1A 0x83
```

Negative responses return `0x7F <SID> <NRC>` where NRC values include:
- `0x11` — serviceNotSupported (SID not recognised)
- `0x12` — subFunctionNotSupported (SID recognised, sub-function invalid)
- `0x22` — conditionsNotCorrect (service exists but preconditions not met)
- `0x35` — invalidKey (security access failed)

---

## EEPROM read

Some ECU data (VIN, adaptation values, relearn state) is stored in an SPI
EEPROM separate from the main flash. Reading it requires finding the
`eeprom_read()` function address in the ROM:

```
# 1. Find eeprom_read() using nisrom
nisrom -l rom_original.bin    # prints eeprom_read() address if found

# 2. Set the address in nisprog
npconf eepr 0x35250    # example address

# 3. Dump EEPROM (512 bytes typical)
dm eeprom.bin 0 512 eep
```

The `eep` flag on `dm` tells nisprog to treat the address as EEPROM space
rather than ROM/RAM address space. Requires kernel to be running.

---

## Connecting to other modules for diagnostic reads

You can target ABS, SRS, BCM, TCM for KWP2000 reads by changing `destaddr`:

```
set destaddr 0x35    # SRS
up
nc
diag sr 0x18 0x02 0xFF 0x00    # read SRS DTCs
```

This is useful during RE to understand what each module stores and which
services it supports. Note: flash and fast dump (`dm` with kernel) only work
with the ECM (`destaddr 0x10`).

---

## Comparing ROM regions before and after

After flashing, dump the ECU and diff against the patched file to confirm
exactly what changed:

```
dm post_flash.bin 0 0
# On PC:
fc /b backup.bin post_flash.bin        # Windows binary diff
diff <(xxd backup.bin) <(xxd post_flash.bin)   # Linux
```

Or use `flverif` within nisprog — it reports which 64-byte blocks differ
without requiring an external diff tool.

## `flrom` with original ROM reference

`flrom new.bin backup.bin` takes an optional second argument — the original
(pre-patch) ROM. When supplied, nisprog determines which blocks to write by
comparing `new.bin` against `backup.bin` (the original), rather than comparing
`new.bin` against the current ECU flash content.

```
# Standard form — compares new ROM against current ECU flash:
flrom patched.bin

# Reference form — compares new ROM against your known-good original:
flrom patched.bin backup.bin
```

Use the reference form when the ECU flash state is uncertain (e.g. after a
partial flash) — it guarantees you're applying exactly the diff between your
known-good dump and your new ROM, regardless of what's currently on the chip.
