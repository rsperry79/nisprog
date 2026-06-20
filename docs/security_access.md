# nisprog — Security Access: gk vs setkeys

Nissan KWP2000 ECUs require a security access challenge-response before
allowing any read or write operation. nisprog implements this via two
commands: `gk` (automatic keyset lookup) and `setkeys` (manual key supply).

---

## How security access works

1. nisprog sends SID 0x27 subfunction 0x01 (requestSeed).
2. ECU responds with a 4-byte seed.
3. nisprog computes the response using the secret key algorithm and the s27k
   (SID 27 key) specific to this ECU.
4. nisprog sends SID 0x27 subfunction 0x02 (sendKey) with the computed response.
5. ECU unlocks if the response matches.
6. For write operations (flash), a second challenge using SID 0x36 and s36k
   is required.

The s27k and s36k are ECU-specific values derived from the ECU ID. They are
not derivable from the ROM file alone — they must be known in advance or
guessed from a database.

---

## `gk` — automatic key guess

```
nc          # connect first — ECUID is retrieved and stored
gk          # attempt to guess keyset from bundled DB
```

`gk` scans a bundled database of known Nissan keysets, matched by ECUID
prefix. On success it prints:

```
SID27 key = XXXXXXXX   SID36 key = YYYYYYYY
```

**Always record this output.** Some ECU variants do not cache the security
session across reconnects. If you disconnect and reconnect, you will need to
re-supply the keys with `setkeys` before flash operations will succeed.

If `gk` fails ("keyset unknown"), the ECU variant is not in the bundled DB.
Use `setkeys` with values from `nissutils keyset_lookup.py` or a community
source.

---

## `setkeys` — manual key supply

```
setkeys <s27k> [<s36k>]
```

### Both keys known

If you have both keys (e.g. from a previous `gk` run or from `keyset_lookup.py`):

```
setkeys 0xABCD1234 0x5678EFGH
```

nisprog uses these directly and confirms:
```
Now using SID27 key=ABCD1234, SID36 key1=5678EFGH
```

### s27k only — automatic s36k lookup

If you supply only the s27k, nisprog looks it up in the known keyset DB to
find the matching s36k:

```
setkeys 0xABCD1234
```

If the s27k matches a known keyset, nisprog resolves the s36k and confirms
both:
```
Now using SID27 key=ABCD1234, SID36 key1=5678EFGH
```

If the s27k is not in the DB, nisprog will print:
```
Does not match a known keyset; you will need to provide both s27 and s36 keys
```

In that case, either find the s36k externally and supply both, or use `gk`
again with a fresh connection to try the full brute-force search.

---

## When to use which

| Scenario | Command |
|----------|---------|
| First time with this ECU | `nc` then `gk` — auto-detects and displays both keys |
| Reconnecting in the same session after `nc` | Keys are still cached — no action needed |
| Reconnecting in a new session with known s27k | `setkeys <s27k>` — looks up s36k from DB |
| Reconnecting with both keys known | `setkeys <s27k> <s36k>` — fastest, no DB lookup |
| `gk` fails (unknown ECU) | Find keys via community sources, then `setkeys <s27k> <s36k>` |
| Crash recovery — kernel still running | Must re-supply keys before any flash; use `setkeys` |

---

## Workflow: recording keys for later use

A safe dump session always captures the keys for later flashing:

```
nc
gk
# Output:
#   SID27 key = 432EF55B   SID36 key = 27CD3964
# Copy these to a file or your session notes.
setdev 7055
runkernel D:\...\npk_SH7055_35.bin
dm backup.bin 0 0
stopkernel
npdisc
```

Then in a later flash session, skip `gk` and supply the keys directly:

```
nc
setkeys 0x432EF55B 0x27CD3964
setdev 7055
runkernel D:\...\npk_SH7055_35.bin
flverif patched_rom.bin
flrom patched_rom.bin
stopkernel
npdisc
```

---

## AI notes

- **Always run `gk` and record the output before doing anything else.** Some
  ECU variants do not cache the security session — if you disconnect and
  reconnect, you must re-supply keys with `setkeys`. Losing the `gk` output
  means the write step will fail with a security-access error that can look
  like a comms problem.
- **`gk` must run after `nc`** — it uses the ECUID retrieved on connect to
  match against the keyset DB.
- **`setkeys` with one argument** does a DB lookup by s27k to find the s36k.
  If the s27k is unknown to the DB, both values must be supplied.

---

## Finding keys outside nisprog

If `gk` fails, the ECU variant is not in nisprog's bundled keyset DB. Options:

- Check community sources (nissutils ROM DB, romraider forums) for the ECUID prefix.
- If the s27k is known from a community source, `setkeys <s27k>` will attempt to
  resolve the s36k from the bundled DB. If that also fails, both values must be supplied.
- If no keyset exists anywhere, the keys must be derived through other means
  (hardware-level extraction or seed/key algorithm analysis).
