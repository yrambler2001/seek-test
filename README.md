# seek-test

A single-page WebUSB tool that reads the SPI-flash chip identity and protection
status out of a Seek Thermal Compact running firmware **4.8.1.9**
(USB `289d:0010`, "PIR206 Thermal Camera") — no drivers, no J-Link, browser only.

**Strictly read-only.** The flash never receives a write, erase or write-enable
command; the only opcodes issued to the chip are `0xFF`, `0x9F` (JEDEC ID),
`0x05`/`0x35`/`0x15`/`0x2B` (status registers) and the `0x3D` block-lock *read*.

## Use

1. Open the GitHub Pages URL in **Chrome or Edge** (WebUSB browsers only).
2. Connect the camera (on the bench, via the switchable USB port).
3. Click **Connect camera**, then **Run flash-id probe**.
4. The page reports JEDEC ID, manufacturer, the three status registers with
   per-bit decoding (Winbond / Macronix / Spansion), and the per-block lock bits.

**Unlock Winbond** (separate button, explicit confirmation) clears the chip's
protection bits back to factory `00/02/60` + Global Block Unlock. It writes
ONLY the chip's protection registers — the flash array is never erased or
programmed — and refuses safely on a non-Winbond chip or when SRP1 (hardware
lock) is set. Run on steady power; these are permanent register writes.

Result for the reference unit: `EF 40 16` (Winbond W25Q32), SR1 `00`,
SR2 `02` (QE), SR3 `60` (drive strength), block locks `1 1 1` — matching the
J-Link probe of the same chip byte-for-byte.

## How it works

The page stages a small ARM Thumb probe into the camera's RAM through an RPC
staging window (wire 90 `SetRamDataFeatures` forces a 64 KB write capacity for a
fixed menu of RAM destinations; wire 67 streams chunks), arms the deferred
one-shot timer (wire 89 `ResetDevice`), and overwrites the TIMER1 callback cell
inside the staged window. When the timer fires, the probe runs once from RAM
with interrupts masked: SPIFI leaves memory-mapped mode, the chip is
interrogated, mapped mode is restored, and 16 result bytes are parked where the
page reads them back over the RPC window getter (wire 68). The camera keeps
serving USB throughout; a reboot restores factory RAM state.

Method notes: `docs/` of the main research repo (audit doc 35, sections 3–5 and 11).

## Repo contents

- `index.html` — the whole tool (probe binary embedded, self-contained).

Source probe: `ht-201/flashid_stub.c` in the firmware reconstruction tree.
