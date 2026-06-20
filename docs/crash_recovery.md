# nisprog — Crash Recovery

## Key principle

**The npkern kernel runs entirely in ECU RAM.** As long as ECU power is
maintained, the kernel keeps running even if nisprog crashes, the PC
reboots, or the USB cable is disconnected. You can reconnect and continue
without reuploading the kernel.

**Never cut ECU power during or after a flash that has not been completed.**
If power is lost mid-flash with changed blocks not yet written, the ECU
ROM is in an inconsistent state and may not boot.

---

## Scenario 1: nisprog crashed, kernel still running

The most common scenario — nisprog exits or the PC crashes, but the ECU
has been on continuously.

```
# 1. Reopen nisprog
nisprog nisprog_dump.ini    # or just: nisprog

# 2. Set comms to kernel baud rate (NOT the default K-line 10400)
set speed 62500             # kernel operates at 62500 bps

# 3. Reconnect — skip nc, go straight to kernel init
nc                          # establishes the L2 session
initk                       # re-initialises kernel comms without reuploading

# 4. Re-supply security keys (session may not be cached)
setkeys 0xXXXXXXXX 0xXXXXXXXX

# 5. Continue
setdev 7055
# Verify the kernel is working before attempting any flash
dm verify_test.bin 0 0x1000    # short dump — confirms stable comms
```

If `initk` succeeds and the dump completes cleanly, it is safe to continue
flashing or dumping.

---

## Scenario 2: flash interrupted mid-write

If a flash was interrupted (nisprog crash, USB disconnect, PC crash) while
`flrom` was running:

1. **Keep ECU power on** — do not turn off the ignition.
2. Reconnect as in Scenario 1 (`nc` → `initk`).
3. Verify kernel comms with a short dump before anything else.
4. Run `flverif patched_rom.bin` to identify which blocks were written and
   which were not.
5. Run `flrom patched_rom.bin` again — nisprog will detect already-correct
   blocks and skip them, only rewriting those that still differ.

```
nc
initk
setkeys 0xXXXXXXXX 0xXXXXXXXX
setdev 7055
dm verify.bin 0 0x1000           # confirm kernel is stable
flverif patched_rom.bin           # see which blocks are still wrong
flrom patched_rom.bin             # rewrite only changed blocks
stopkernel
```

---

## Scenario 3: kernel crashed or hung

If `initk` fails or the kernel is not responding (timeout errors, no response
to any command):

1. Attempt to purge the kernel's buffer — open a terminal (e.g. RealTerm) on
   the same COM port at 62500 bps and send several `0x00` bytes until you get
   a `0x7F NN` negative response. This re-syncs the framing.
2. If purge succeeds, try `initk` again in nisprog.
3. If the kernel cannot be recovered, the ECU must be power-cycled. The flash
   state is whatever it was when the kernel last ran.

After a power cycle the ECU boots from flash. If an interrupted flash left
blocks in an inconsistent state, the ECU may not boot normally.

**SH7055 F-ZTAT hardware recovery (board-level):** The SH7055 has a hardware
boot ROM burned into silicon. Pulling the MD pins (JP1/JP2 on PCB) low at
power-on puts the chip into a factory serial programming mode — it accepts a
reflash program over the SCI serial interface and can recover from a completely
blank or corrupted flash state. This requires opening the ECU enclosure.

nisprog's selective flash never writes Block 0 (reset vectors and boot code)
unless `f` (flash all) is confirmed — so a crash during a normal cal/code
flash typically leaves the boot sector intact and the F-ZTAT path may not be
needed.

---

## Scenario 4: USB cable reconnect without PC restart

If the USB cable was unplugged and reconnected while nisprog was open:

1. The serial port handle is invalid — exit and reopen nisprog.
2. The kernel is still running in ECU RAM (ECU power was maintained).
3. Follow Scenario 1.

---

## Scenario 5: PC reboot, ECU still powered

Same as Scenario 1. The kernel survives indefinitely in RAM as long as
ECU power is maintained and the watchdog timer inside the kernel is being
serviced. The kernel's internal WDT kicks the hardware WDT continuously —
if the kernel hangs internally, the hardware WDT will reset the ECU.

---

## Quick reference: recovery command sequence

```
# After crash — kernel still running in ECU RAM
set speed 62500           # kernel comms speed
nc                        # establish L2 session
initk                     # sync to running kernel
setkeys 0xXXXX 0xYYYY    # re-supply security keys
setdev 7055               # re-set device type
dm test.bin 0 0x1000     # short test dump to confirm stable comms
# then continue flverif / flrom / dm as needed
stopkernel                # when fully done — resets ECU
```

---

## AI notes

- **Never cut ECU power during a flash.** The kernel keeps running in RAM as
  long as power is maintained. Use `initk` to reconnect without reuploading.
- **`flverif` "write errors" in dry-run are expected** — it does not write
  anything. The errors show which blocks would be changed.
- **`setdev` must be re-set after reconnect** — it is not persisted across
  nisprog sessions.
- **Re-supply security keys after reconnect** — `setkeys` before any flash
  operation. See [`security_access.md`](security_access.md).

---

## Prevention

- Always run `flverif` before `flrom` to verify the ROM file is correct.
- Run `nisrom --fix <rom>` on any patched ROM before flashing to verify and
  correct checksums. A ROM with wrong checksums may cause ECU misbehaviour
  even after a successful flash.
- Never flash below 12.5V — keep a battery charger on the car throughout.
- Keep the K-line cable connected throughout `flrom` — do not unplug.
- Save `gk` output before starting — needed for reconnect without re-running `gk`.
