# HP Compaq dc7900 USDT POST Freeze — Recovery Procedure

[Documentation index](README.md) | [Full English report](full-report-en.md) | [中文完整报告](full-report-zh-CN.md) | [Image evidence](../images/README.md)

> **Scope:** HP Compaq dc7900 USDT / BIOS family `786G1`, for the specific failure mode documented in this repository.  
> **Validated on:** two physical machines.  
> **Do not treat this as a universal fix for every dc7900 that fails to POST.**

---

## 1. When this procedure applies

This procedure is intended for a machine that still reaches late POST but freezes in a characteristic way:

```text
POST progresses normally
memory count completes
F9 / F10 / F12 may still be captured
F10 displays SETUP
machine then freezes permanently
BIOS Setup never actually opens
normal boot cannot continue
```

This procedure is **not** the first choice for a machine that is already:

```text
completely black
no meaningful POST
fan at full speed
```

A black-screen machine may have an unrelated hardware problem such as:

- CPU / socket contact;
- power rail failure;
- reset failure;
- missing clock;
- broken SPI CS# / CLK / MISO path;
- bad soldering or damaged pads.

Machine A in this project eventually had both:

```text
original ME persistent-state failure
+
later CPU / socket contact failure
```

Do not assume every later symptom still has the same cause.

---

## 2. Recovery principle

The successful repair used the following region policy:

```text
0x000000–0x00AFFF
KEEP target machine
Descriptor + GbE/MAC + PDR

0x00B000–0x25FFFF
REPLACE
healthy HP OEM Intel ME Region

0x260000–0x261FFF
KEEP target machine
BIOS runtime / DMI / identity data

0x262000–0x3FFFFF
REPLACE
official HP BIOS v1.27 System BIOS body
```

The core rule is:

> **Preserve target identity; replace the failing ME persistent state.**

Do **not** blindly overwrite the complete 4 MiB SPI image with another machine's dump or a generic HP image.

---

# 3. Confirmed SPI layout

Complete SPI size:

```text
4,194,304 bytes
0x000000–0x3FFFFF
```

| Region | Address range | Size | Final policy |
|---|---:|---:|---|
| Flash Descriptor | `0x000000–0x000FFF` | 4 KiB | keep target |
| GbE NVM | `0x001000–0x002FFF` | 8 KiB | keep target |
| PDR | `0x003000–0x00AFFF` | 32 KiB | keep target |
| Intel ME | `0x00B000–0x25FFFF` | `0x255000` bytes | replace with healthy HP OEM ME |
| BIOS runtime / DMI | `0x260000–0x261FFF` | 8 KiB | keep target |
| System BIOS body | `0x262000–0x3FFFFF` | `0x19E000` bytes | use official HP BIOS v1.27 body |

Expected Descriptor signature:

```text
5A A5 F0 0F
```

The GbE NVM should be preserved because it carries machine-specific network configuration including the MAC address.

---

# 4. Required hardware and software

Minimum practical equipment:

- reliable 3.3 V SPI programmer;
- suitable SOIC8 / SOP16 adapter or a stable soldered connection;
- known-good Windows host for programming;
- multimeter;
- soldering equipment if board repair is required;
- Linux Live USB for final HECI / MEI validation.

This project successfully used:

```text
EZP2019+
Windows 10
```

One Windows 11 host in this project produced unreliable **Erase** behavior with both EZP2019+ and EZP2023 even though reads and some programming operations worked.

That does **not** prove universal Windows 11 incompatibility.

It proves only:

> if Erase behaves abnormally, change the programming host before condemning the Flash or programmer.

---

# 5. Hardware baseline before firmware work

Disconnect AC power.

Restore the board to the normal service baseline:

```text
E49 / JP49 = pins 1–2
E1 = open
E14 = open
CR2032 = installed and good
```

For CMOS clear:

```text
disconnect AC
remove CR2032
hold SW50 for about 10 seconds
reinstall battery
```

Remove unnecessary peripherals.

For initial troubleshooting, keep only:

- CPU;
- known-good memory;
- basic display;
- required SPI hardware.

---

# 6. Acquire a trustworthy original SPI dump

This is mandatory.

Do not start by erasing the chip.

Read the original SPI at least three times:

```text
SPI_TARGET_1.bin
SPI_TARGET_2.bin
SPI_TARGET_3.bin
```

Each file must be exactly:

```text
4,194,304 bytes
```

Calculate SHA-256 for all three.

On Windows PowerShell:

```powershell
Get-FileHash .\SPI_TARGET_1.bin -Algorithm SHA256
Get-FileHash .\SPI_TARGET_2.bin -Algorithm SHA256
Get-FileHash .\SPI_TARGET_3.bin -Algorithm SHA256
```

Proceed only when all three hashes are identical.

The logic is:

```text
Read
Read
Read
↓
same SHA-256
↓
trust the dump as a baseline
```

If the three reads differ, stop and fix:

- clip contact;
- adapter contact;
- supply stability;
- board loading;
- programmer connection.

Do not interpret binary differences until acquisition itself is reliable.

---

# 7. Preserve original evidence

Before editing anything, keep at least two untouched copies of the original dump.

Recommended private archive:

```text
original/
  SPI_TARGET_1.bin
  SPI_TARGET_2.bin
  SPI_TARGET_3.bin
  SHA256.txt
```

Do not modify these files.

Do not use an edited file as the only surviving "original."

---

# 8. Record machine identity before building a repair image

Record the target machine's machine-specific data before modifying the image.

At minimum preserve and document:

- MAC address;
- Product / SKU;
- Serial Number;
- UUID if available;
- DMI data;
- GbE NVM.

The public repository deliberately redacts these values.

Your private recovery copy should not.

The final repaired image must retain the target machine's own:

```text
Descriptor
GbE / MAC
PDR
BIOS runtime / DMI
```

---

# 9. Obtain a healthy HP OEM ME source

The required donor is **not** merely an Intel ME update executable or CODE/NFTP payload.

The successful fix required a complete healthy HP OEM ME Region compatible with this platform.

The healthy donor used in this project ultimately provided:

```text
0x00B000–0x25FFFF
healthy HP OEM ME Region
```

The project also verified that HP official BIOS 1.26 and 1.27 contain an identical ME Region.

Therefore:

> the successful repair was not caused by BIOS 1.27 "updating ME."

BIOS 1.27 is used for the System BIOS body, while the healthy OEM ME Region replaces the target's failing ME state.

---

# 10. Why BIOS-only replacement is not enough

A controlled B-machine test replaced only:

```text
0x262000–0x3FFFFF
System BIOS body
```

with official HP BIOS v1.27.

The machine displayed BIOS v1.27 but still froze at the same point.

Therefore:

```text
BIOS-only update
≠
repair for this confirmed failure mode
```

Do not repeatedly flash only the System BIOS and expect this specific ME-state failure to disappear.

---

# 11. Why CODE / NFTP-only replacement is not enough

Another controlled B-machine test replaced the executable ME CODE / NFTP portion with version:

```text
5.2.50.1039
```

while keeping the original persistent state.

The symptom did not change.

Therefore the failure is not explained by:

```text
old ME executable version
```

or:

```text
ME CODE / NFTP alone
```

The evidence points instead toward:

```text
EFFS
NVAR
OEM persistent configuration
runtime state
```

---

# 12. FITC diagnostic evidence

This project used:

```text
Intel Flash Image Tool
5.0.0.1167
Boulder Creek - EL ICH10
```

On the failing B image, FITC reported:

```text
ME_CFG_DEF NVAR Not found.
KernFixedData NVAR Not found. Adding NVAR.
```

and the decomposed:

```text
Configuration.txt
```

was empty.

The same FITC version could parse a healthy HP OEM image.

This strongly supports a persistent configuration / NVAR abnormality.

However:

> FITC Build output must not be treated as automatically flash-safe.

In this project, FITC Build changed Descriptor-related data and was rejected as a final image source.

Use FITC for analysis unless every generated change is fully understood.

---

# 13. Build the repaired image

Use the target original as the base.

Conceptually:

```python
from pathlib import Path

target = bytearray(Path("target_original.bin").read_bytes())
donor  = Path("validated_hp_oem_donor.bin").read_bytes()

assert len(target) == 0x400000
assert len(donor)  == 0x400000

# Healthy HP OEM ME Region
target[0x00B000:0x260000] = donor[0x00B000:0x260000]

# Official HP BIOS v1.27 body
target[0x262000:0x400000] = donor[0x262000:0x400000]

Path("target_fixed.bin").write_bytes(target)
```

The resulting structure must be:

```text
000000–00AFFF
target original

00B000–25FFFF
healthy HP OEM ME

260000–261FFF
target original

262000–3FFFFF
official HP BIOS v1.27 body
```

This is the only intended splice.

Do not replace additional regions merely because a tool makes it easy.

---

# 14. Validate the constructed image before programming

Before flashing, verify all four provenance ranges.

The fixed image must satisfy:

```text
000000–00AFFF
= target original

00B000–25FFFF
= validated healthy HP OEM donor

260000–261FFF
= target original

262000–3FFFFF
= official HP BIOS v1.27 body
```

Also confirm:

```text
file size = 4,194,304 bytes
```

and record the SHA-256 of the final image.

If machine identity can be parsed, confirm again that:

- target MAC is retained;
- target serial is retained;
- target Product / SKU is retained;
- target DMI is retained.

---

# 15. Program the SPI Flash

Standard sequence:

```text
Erase
↓
Blank Check
↓
Program
↓
Verify
↓
full Read Back
↓
SHA-256
```

Do not stop at the programmer's:

```text
Verify OK
```

After programming, perform a complete fresh read of the chip:

```text
READBACK.bin
```

Calculate:

```powershell
Get-FileHash .\target_fixed.bin -Algorithm SHA256
Get-FileHash .\READBACK.bin -Algorithm SHA256
```

The hashes must match exactly.

If they do not match:

> **do not install or boot the chip.**

---

# 16. U19 / U21 hardware repair if pads are damaged

The dc7900 board provides:

```text
U19 SOIC8
U21 SOP16
```

Confirmed signal mapping:

| U19 SOIC8 | Signal | U21 SOP16 |
|---:|---|---:|
| 1 | CS# | 7 |
| 2 | MISO | 8 |
| 3 | WP# | 9 |
| 4 | GND | 10 |
| 5 | MOSI | 15 |
| 6 | SCLK | 16 |
| 7 | HOLD# | 1 |
| 8 | VCC 3.3 V | 2 |

SOP16 pins:

```text
3–6
11–14
```

are NC in this use.

Reference images:

- [B-machine four-pad damage](../images/03-b-u19-u21-pad-damage.jpg)
- [B-machine signal map](../images/04-b-u19-u21-signal-map.jpg)
- [A-machine Pin 3 / WP# damage](../images/05-a-u19-pin3-wp-pad-damage.jpg)

### Important B-machine lesson

B lost U19:

```text
Pin 1
Pin 2
Pin 3
Pin 4
```

Some U21-side nets still depended on the damaged U19 area.

Therefore:

> soldering a chip to U21 alone may not restore the circuit.

You must verify real downstream continuity.

For B:

- CS# had to reach the true downstream CS# network;
- MISO had to reach the true downstream data network;
- WP# could be held high through approximately 4.7–10 kΩ to 3.3 V;
- GND had to be tied to verified ground.

Do not assume a visually nearby pad is electrically sufficient.

---

# 17. First boot after programming

Before first boot:

1. restore normal jumpers;
2. clear CMOS;
3. remove programming hardware not required for operation;
4. perform a complete AC-off cycle.

On first boot, initialization messages such as the following may be expected:

```text
162
CMOS checksum invalid
defaults loaded
```

The exact wording may vary.

The important functional check is:

```text
F10
↓
BIOS Setup actually opens
```

The original failure mode is not considered repaired merely because the screen looks different.

---

# 18. Stability test

After the first successful boot, perform at least:

```text
3 warm reboots
+
3 complete AC-off cold boots
```

Confirm each time:

- POST completes;
- F10 BIOS Setup opens;
- boot continues normally;
- the original fixed freeze point does not return.

The transient:

```text
2233
2206
```

HECI / MEBx errors appeared once during the B recovery process but did not persist.

A one-time initialization error should be documented, but persistent repetition requires further diagnosis.

---

# 19. Linux HECI / MEI validation

Boot a Linux Live environment and run:

```bash
lspci -nnk -d 8086:2e14
lsmod | grep -E 'mei|heci'
ls -l /dev/mei*
cat /sys/class/mei/mei0/dev_state
cat /sys/class/mei/mei0/fw_ver
cat /sys/class/mei/mei0/fw_status
cat /sys/class/mei/mei0/hbm_ver
```

The project observed on recovered B:

```text
PCI: 8086:2e14 rev03
Subsystem: HP 103c:3036
Kernel driver: mei_me
/dev/mei0: present
dev_state: ENABLED
fw_status: 300A965A
hbm_ver: 1.0
```

The project also observed:

```text
fw_ver = 0:0.0.0.0
```

Do not interpret that single field alone as proof that ME firmware is absent.

The complete evidence is more important:

```text
HECI PCI device present
+
mei_me bound
+
/dev/mei0 present
+
dev_state ENABLED
+
fw_status readable
+
HBM 1.0 readable
```

Reference:

- [Linux HECI / MEI evidence](../images/09-linux-heci-pci-mei.jpg)
- [Linux MEI sysfs evidence](../images/10-linux-mei-sysfs-status.jpg)

---

# 20. If the machine is still frozen at the original point

If all of the following are true:

- programming read-back matches the intended image;
- target identity regions were preserved;
- healthy HP OEM ME was used;
- official v1.27 BIOS body was used;
- hardware SPI connectivity is confirmed;

and the machine still freezes at exactly the original late-POST point, stop assuming this repository's failure mode is proven for that machine.

Re-check:

- donor compatibility;
- actual SPI region boundaries;
- board revision;
- RAM and CPU baseline;
- chipset state;
- whether the symptom truly matches.

Do not continue generating increasingly broad hybrid BIN files without a controlled hypothesis.

---

# 21. If the machine is black with fan at full speed

This is a different diagnostic branch.

After a known-good SPI image is confirmed, prioritize hardware.

Check:

```text
SPI Pin 1 CS#
SPI Pin 6 CLK
SPI Pin 2 MISO
```

If CS# / CLK show no startup activity, prioritize:

- chipset power;
- reset;
- clock;
- straps;
- CPU;
- socket.

Machine A ultimately recovered only after a CPU / socket contact issue was corrected.

A missing WP# pad by itself did not explain A's full black-screen state.

---

# 22. Do not use these project images

The following experimental files are deprecated:

```text
DC7900_A_127_CLEAN_TEST_V2.bin
DC7900_B_127_CLEAN_TEST_V2.bin
```

They belong to an early multi-variable route and must not be reused as recovery images.

Also avoid:

- unexplained FITC Build output;
- another machine's complete 4 MiB dump as a permanent image;
- a generic full HP image flashed from `0x000000`;
- a donor image that overwrites target MAC / DMI / serial data.

---

# 23. Reference hashes from this project

These hashes are provided for project traceability, not as downloadable firmware.

### Original A

```text
SHA-256
209577D93CA7B70DE336864A4B771499C7234318955B9C5F1571697E9B9991E4
```

### Original B

```text
SHA-256
70B3A1229420AE9FB7363B2A96C9B4C506EA1CCE5D8A8B266DE2A012B80DB967
```

### Official HP 1.26 full ROM used for comparison

```text
SHA-256
EAC16981A32DBFDC312B3C12597522F4520D5A7D16BE4F3422F2983A2652EDAF
```

### B BIOS-only v1.27 experiment

```text
SHA-256
5357E612C21655EBA27D84190343884556CE148C21A7099A37A53D6870B3F3C4
```

### B CODE / NFTP 5.2.50.1039 experiment

```text
SHA-256
884CC0F4C2376ED093702D9797B99E74B1DC79136C007311BD72A22BDC5A7903
```

### B validated successful donor / baseline

```text
SHA-256
2F42B1A1B12F0D85B8DE4A8546687EBD6A11447AE91FC3FE3E88992EE8927123
```

### A final personalized image

```text
SHA-256
8AFDA8295E4CE3325DD3FD409B444655CDE9D4E0F5B3FF8D79CFB301E2978683
```

---

# 24. Compact decision tree

```text
dc7900 freezes after memory count / F10 displays SETUP
        ↓
restore normal jumpers + CMOS baseline
        ↓
obtain 3 identical 4 MiB SPI reads
        ↓
preserve target identity regions
        ↓
confirm main BIOS body / ME evidence
        ↓
build:
target Descriptor+GbE+PDR
+
healthy HP OEM ME
+
target runtime/DMI
+
official HP BIOS v1.27 body
        ↓
program + full read-back + matching SHA-256
        ↓
clear CMOS + cold boot
        ↓
F10 Setup opens?
   ┌────┴────┐
  yes        no
   ↓          ↓
Linux MEI     verify actual symptom,
validation    donor, SPI map, hardware
```

If the machine instead becomes:

```text
black screen + fan full speed
```

branch immediately to hardware diagnosis rather than continuing firmware splicing.

---

# 25. Final checklist

Before calling the repair complete, verify:

- [ ] original SPI was read at least three times;
- [ ] all original reads had identical SHA-256;
- [ ] untouched original dumps are archived;
- [ ] target MAC / DMI / serial identity was recorded;
- [ ] Descriptor was preserved;
- [ ] GbE / MAC was preserved;
- [ ] PDR was preserved;
- [ ] BIOS runtime / DMI `0x260000–0x261FFF` was preserved;
- [ ] complete healthy HP OEM ME replaced `0x00B000–0x25FFFF`;
- [ ] official HP BIOS v1.27 body replaced `0x262000–0x3FFFFF`;
- [ ] final image size is exactly 4 MiB;
- [ ] programming completed with Erase / Blank / Program / Verify;
- [ ] a complete read-back was performed;
- [ ] read-back SHA-256 matches the intended image;
- [ ] jumpers are back in normal state;
- [ ] CMOS was cleared;
- [ ] F10 BIOS Setup actually opens;
- [ ] at least three warm boots pass;
- [ ] at least three AC-off cold boots pass;
- [ ] Linux sees HECI `8086:2e14`;
- [ ] `mei_me` binds;
- [ ] `/dev/mei0` exists;
- [ ] MEI sysfs state is readable;
- [ ] original POST freeze does not reproduce.

---

## Result expected for this confirmed failure mode

```text
Original:
late POST freeze
F10 -> SETUP displayed
Setup never opens

After validated recovery:
POST completes
F10 Setup opens
operating system boots
HECI / MEI communication works
```

For the complete investigation and evidence chain, see:

- [Full English technical report](full-report-en.md)
- [中文完整技术报告](full-report-zh-CN.md)
- [Image evidence archive](../images/README.md)
