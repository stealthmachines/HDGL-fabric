# HDGL Router64 — Boot Chain Fix (MBR → Stage2 → Runtime64)

Three bugs were found and fixed by actually booting the chain in QEMU with
COM1 captured to a serial console, rather than by static read-through. All
three were invisible in the source and only surfaced at execution time.

## Bug 1 — MBR clobbers the boot drive number before reading the disk

`mbr.asm` prints an "AXIOM" banner over COM1 before issuing `int 0x13` to
load stage2. The banner code does `mov dx, 0x3F8` to address the UART, and
`.com1_tx` only saves/restores `dx` *within its own call* — it never
restores the BIOS-supplied boot drive number that arrives in `DL` at entry.
By the time `int 0x13` runs, `DL` has been left at `0xF8` (the low byte of
`0x3F8`) instead of `0x80`, so every disk read fails with an invalid-drive
error.

Confirmed by a throwaway debug MBR that printed `DL` and the `int 0x13`
error code directly: `DL` arrived correctly as `80` from BIOS, but was gone
by the time of the read in the original code.

**Fix:** latch `dl` into a `boot_drive` byte immediately on entry (before
any COM1 calls), and reload it into `dl` immediately before `int 0x13`.

## Bug 2 — `.com1_hex_dword` rotates the wrong register width

The hex-dword print routine did:

```asm
mov rbx, rax      ; 64-bit register
mov ecx, 8
.chd_loop:
    rol rbx, 4    ; rotates the 64-bit register
    ...
    loop .chd_loop
```

`mov eax, ...` zero-extends the upper 32 bits of `rax`, so a 32-bit value
loaded this way sits in the low half of a 64-bit register. Rotating that
64-bit register by 4 bits, 8 times, only rotates it 32 of its 64 positions
— half a turn. The actual data never reaches the low nibble in 8 steps, so
every digit read back as `0`. In context this silently printed `genome_fp`
as `00000000`, which reads exactly like a plausible (if uninteresting)
passing value rather than an obvious failure.

**Fix:** operate on `ebx`/`rol ebx, 4` (32-bit) instead of `rbx`/`rol rbx, 4`.

## Bug 3 — empty string literal aliases the next label

```asm
.msg_genome_fp    db ""   ; placeholder — computed inline
.msg_prompt       db "Router64> ", 0
```

`db ""` emits zero bytes. `.msg_genome_fp` therefore points to the exact
same address as `.msg_prompt` — it isn't an empty string, it's the *same*
string. The `lea rsi, [rel .msg_genome_fp] / call .com1_str` step printed
`"Router64> "` early, in the middle of the final banner, before the actual
hex digits.

**Fix:** the placeholder served no purpose (the hex value is printed
separately by `.com1_hex_dword` right after); removed the dead label and
the print call referencing it.

## Verified output (QEMU, e1000 NIC, COM1 → stdio)

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

`genome_fp = 0x779B1000` matches the hand-derived expectation in the
source comments: `phi_fold(dn_aggregate=0, phi_lattice_mean=0x1000, seq=0)`
= lower 32 bits of `0x9E3779B1 * 0x1000`.

## Contents

```
src/mbr.asm          corrected MBR (Bug 1 fix)
src/stage2.asm        unchanged from your working rebuild — A20/PM/LM, loads
                       runtime64 at 0x9000 from LBA 4
src/runtime64.asm     corrected runtime (Bug 2 + Bug 3 fixes)
bin/mbr.bin           512 bytes, assembled
bin/stage2.bin        1536 bytes, assembled
bin/runtime64.bin     32768 bytes, assembled
bin/disk.img          34816 bytes — cat of the three .bin files in boot order
```

## Rebuilding and testing

```sh
nasm -f bin src/mbr.asm       -o bin/mbr.bin
nasm -f bin src/stage2.asm    -o bin/stage2.bin
nasm -f bin src/runtime64.asm -o bin/runtime64.bin
cat bin/mbr.bin bin/stage2.bin bin/runtime64.bin > bin/disk.img

# Pad to give SeaBIOS a standard CHS geometry under headless QEMU; not
# needed in your existing image/Makefile if it already targets a larger
# raw image or uses -drive ...,cyls=,heads=,secs= explicitly.
truncate -s 10M bin/disk.img

qemu-system-x86_64 -drive file=bin/disk.img,format=raw,if=ide \
    -nographic -no-reboot -serial mon:stdio
```

On real PCEngines APU / Protectli hardware the boot-drive issue (Bug 1)
would have manifested identically, since it's a BIOS-contract bug, not a
QEMU artifact — worth flagging as the highest-priority fix of the three
for metal bring-up.
