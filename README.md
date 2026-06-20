# HDGL Router64 Boot Chain — Full Instructions

Covers three things: setting up the toolchain on a fresh machine, rebuilding
and testing in QEMU, and writing the result to real boot media for
PCEngines APU / Protectli hardware. Read the "Hardware gotchas" section
before you write anything to a physical board — there are two settings
that will look like a dead board if you skip them.

---

## 1. Toolchain setup on a fresh machine

You need `nasm` (assembler) always, and `qemu-system-x86` only if you want
to test before touching real hardware.

### Debian/Ubuntu

```sh
sudo apt-get update
sudo apt-get install -y nasm qemu-system-x86
```

### Fedora/RHEL

```sh
sudo dnf install -y nasm qemu-system-x86
```

### macOS (Homebrew)

```sh
brew install nasm qemu
```

Verify:

```sh
nasm -v                    # NASM version 2.x
qemu-system-x86_64 --version
```

This was built and tested against NASM 2.16.01 and QEMU 8.2.2. Anything
from NASM 2.14+ and QEMU 6.0+ should behave identically — nothing here
uses bleeding-edge syntax.

---

## 2. Rebuild from source

Directory layout assumed below: `src/{mbr,stage2,runtime64}.asm` and an
output `bin/` directory.

```sh
mkdir -p bin
nasm -f bin src/mbr.asm       -o bin/mbr.bin
nasm -f bin src/stage2.asm    -o bin/stage2.bin
nasm -f bin src/runtime64.asm -o bin/runtime64.bin
```

Expected sizes — if any of these don't match, something in the source
changed or `nasm` truncated on an error that didn't set a non-zero exit
code (rare, but check `nasm`'s stderr output if so):

| File             | Size (bytes) | Why                                            |
|------------------|--------------|-------------------------------------------------|
| `mbr.bin`        | 512          | fixed: one boot sector                          |
| `stage2.bin`     | 1536         | `times (3*512)-($-$$) db 0` — 3 sectors          |
| `runtime64.bin`  | 32768        | `times (64*512)-($-$$) db 0` — 64 sectors        |

```sh
wc -c bin/*.bin
```

Concatenate in boot order — this *is* the disk image, sector for sector:

```sh
cat bin/mbr.bin bin/stage2.bin bin/runtime64.bin > bin/disk.img
wc -c bin/disk.img    # 34816 bytes = 68 sectors
```

That's the whole build. No linker, no Makefile dependencies beyond NASM
itself for this three-file chain.

---

## 3. Test in QEMU before touching hardware

### 3a. Why you need to pad the image first

QEMU's BIOS (SeaBIOS) derives a CHS (cylinder/head/sector) disk geometry
from the image size when no geometry is given explicitly. For a tiny
34816-byte raw image it can pick a degenerate geometry where the `int
0x13` reads this boot chain issues (hardcoded `CH=0`, `CL=2`/`5`, `DH=0`)
fall outside the sectors-per-track SeaBIOS thinks exist, and the read
fails — even though the code and the real disk layout are both correct.
Padding the image to a size SeaBIOS recognizes as a "normal" disk
(16 heads × 63 sectors/track is the classic default) sidesteps this. This
is **only a QEMU/SeaBIOS quirk** — a real disk controller and the real
machine's BIOS resolve the boot drive's actual geometry from the hardware
itself, so you do not pad before writing to physical media (see §4).

```sh
cp bin/disk.img bin/disk_test.img
truncate -s 10M bin/disk_test.img
```

### 3b. Boot it with serial captured to your terminal

```sh
qemu-system-x86_64 \
  -drive file=bin/disk_test.img,format=raw,if=ide \
  -nographic \
  -no-reboot \
  -serial mon:stdio
```

Flag-by-flag:
- `-drive file=...,format=raw,if=ide` — attach the image as an IDE hard
  disk, raw format (no qcow2/sparse interpretation).
- `-nographic` — no VGA window; this also redirects the QEMU monitor and
  serial console to your terminal by default, which is why...
- `-serial mon:stdio` — ...this explicitly multiplexes the COM1 serial
  port and the QEMU monitor onto stdio, so you see the firmware's COM1
  output (the `AXIOM`/`S2`/`PM`/`LM`/test banners) directly in your shell.
- `-no-reboot` — halts QEMU instead of resetting when the guest issues a
  reset, useful if a bug causes a triple-fault loop; you'll see it hang
  instead of silently rebooting forever.

Expected output, end to end:

```
AXIOM
S2
RT
PM
LM
[Omega] Omegan+1=T(Omegan) axiom: RUNTIME
  [T1] phi constants (PHI32, PHI32_INV)... PASS
  [T2] zero-clear via mov not xor... PASS
  [T3] PCI NIC scan... (Intel e1000) PASS
  [T4] VGA split chrome... PASS
  [T5] IPC ring init... PASS
  [T6] phi-distance GENOME-LOCK... PASS
  [T7] phi_tick loop (3 ticks)...
    tick=1
    tick=2
    tick=3
PASS
[Fabric] ready  nic=1  store=open  genome_fp=0x779B1000
Router64> 
```

It stops at the `Router64> ` prompt and spins (`jmp $`) — that's the
smoke-test design, not a hang. Press `Ctrl-A` then `X` to quit QEMU
cleanly (`Ctrl-A` is the QEMU escape prefix under `-nographic`; `X`
terminates the emulator).

### 3c. If you want an RTL8139 NIC path instead of e1000

The PCI scan test (T3) reports whichever NIC vendor it finds first. To
exercise the Realtek branch instead of Intel:

```sh
qemu-system-x86_64 \
  -drive file=bin/disk_test.img,format=raw,if=ide \
  -net nic,model=rtl8139 -net user \
  -nographic -no-reboot -serial mon:stdio
```

T3 should then report `(RTL)` instead of `(Intel e1000)`.

### 3d. Troubleshooting

| Symptom | Likely cause |
|---|---|
| `E.` right after `AXIOM`, nothing else | MBR disk read failed. If you're using the corrected `mbr.asm`, this means the geometry-padding step (§3a) was skipped. |
| Boots, prints `AXIOM` then nothing | Stage2 read failed inside the MBR's own retry path — check the image wasn't truncated by a partial `cat`. |
| Hangs after `LM` with no `[Omega]` banner | Stage2's CHS read for runtime64 (LBA 4, 63 sectors) is missing or `runtime64.bin` is shorter than expected — re-check the `wc -c` sizes from §2. |
| `(no NIC)` instead of a vendor name | Expected if you didn't attach a NIC device to the QEMU command line at all; QEMU's default `-net` config usually still provides one, so this is mostly only seen with `-nic none`. |
| Any test reports `FAIL` | This is a real regression — none of the corrected sources should produce a `FAIL` on T1–T7 as shipped. Worth a closer look before going near hardware. |

---

## 4. Writing to real boot media (PCEngines APU / Protectli)

### Hardware gotchas — read this before you write anything

1. **Serial console baud rate: 9600, not 115200.** `runtime64.asm`
   programs the UART divisor as `12`, and `115200 / 12 = 9600`. PCEngines
   APU boards (coreboot/SeaBIOS) and most Protectli BIOS console-redirect
   defaults run at **115200 8N1**. If your terminal program is set to
   115200 you will see correct SeaBIOS POST text, then garbage or
   nothing once this firmware's UART init takes over at boot-sector
   handoff. Set your terminal (`screen`, `minicom`, `putty`, etc.) to
   **9600 8N1** to read this firmware's own output, or expect to
   re-configure mid-session if you're also watching the board's own
   coreboot/SeaBIOS banner first.
2. **Legacy/CSM boot only — no UEFI.** This is a raw MBR boot sector
   (`0xAA55` signature, real-mode entry at `0x7C00`) that hand-rolls its
   own A20/protected-mode/long-mode transition. It will not be discovered
   by a pure-UEFI boot manager. On the APU's SeaBIOS this is the native
   mode, no setting needed. On Protectli boards (AMI/Insyde UEFI), go
   into BIOS setup and enable **Legacy/CSM boot mode**, and if available,
   set legacy boot before UEFI in the boot order.
3. **CHS reads, not LBA48.** The MBR and stage2 both issue classic
   `int 0x13, AH=02` CHS reads, not `AH=42` extended/LBA reads. This
   works on essentially all modern BIOS-CSM disk emulation (it's
   translated to LBA internally), but if you ever swap to a disk larger
   than ~8GB with unusual partitioning, double check the target media is
   being addressed in CHS-translatable range — for a sub-1GB boot device
   this is a non-issue.
4. **No padding step on the real write.** §3a's `truncate -s 10M` was a
   QEMU/SeaBIOS-only workaround. Do not pad before writing to physical
   media — write `bin/disk.img` (34816 bytes) directly. The real board's
   BIOS resolves the actual device geometry from the device itself, not
   from file size heuristics.

### Identify the target device — do this carefully

Writing to the wrong device will destroy data on your build machine.
Confirm the device node before running `dd`.

```sh
# Insert the target media (USB stick / CF card via reader / mSATA-to-USB
# adapter for the APU/Protectli's internal media), then:
lsblk
# or, more verbose:
sudo fdisk -l
```

Identify the target by its **size** matching your media, not by guessing
from device order. On Linux it's commonly `/dev/sdX` or `/dev/mmcblkX`
for SD readers; on macOS, `/dev/diskN` (use the raw `/dev/rdiskN` for
speed). **If you're unsure which device is which, stop and check `lsblk`
output again rather than guess.**

### Write the image

Linux:

```sh
sudo dd if=bin/disk.img of=/dev/sdX bs=4M conv=fsync status=progress
sync
```

macOS (use the raw device for speed, and unmount first):

```sh
diskutil unmountDisk /dev/diskN
sudo dd if=bin/disk.img of=/dev/rdiskN bs=4m
sync
```

Replace `/dev/sdX` / `/dev/diskN` with the device you confirmed above —
never the path to a partition (`/dev/sdX1`) or your build machine's own
disk.

### Boot it

1. Insert the written media into the APU/Protectli's boot device
   (mSATA/CF/eUSB/SD depending on the board).
2. Connect a serial console (DB9 null-modem or USB-serial adapter) at
   **9600 8N1** — see gotcha #1 above. If you want to also see the
   board's own coreboot/SeaBIOS POST banner first, start at 115200, then
   switch your terminal to 9600 right as control passes to this MBR
   (you'll see garble the instant it does — that's your cue).
3. Power on. With Legacy/CSM enabled and this media first in boot order,
   you should see the same `AXIOM` → `S2` → `RT` → `PM` → `LM` →
   `[Omega]...` sequence as the QEMU run, ending at `Router64> `.
4. T3 (PCI NIC scan) will report whichever real NIC the board has wired
   to PCI bus 0 — Intel `8086:*` boards (APU's i211/i225, or Protectli's
   i219/i225) report `(Intel e1000)` and Realtek boards report `(RTL)`,
   using the same vendor-ID check exercised in QEMU (§3.c), now against
   real silicon.

If nothing appears at all on the serial console after power-on, the
first two things to check are exactly the two gotchas above: baud rate
mismatch, and Legacy/CSM not enabled.
