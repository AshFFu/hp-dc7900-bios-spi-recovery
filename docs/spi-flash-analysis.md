# SPI Flash Analysis

[Documentation index](README.md) | [Recovery procedure](recovery-procedure.md) | [Full English report](full-report-en.md) | [中文完整报告](full-report-zh-CN.md)

This document isolates the binary-analysis portion of the HP Compaq dc7900 USDT recovery project.

It focuses on:

- the confirmed 4 MiB SPI layout;
- machine-specific versus replaceable regions;
- the A / B / HP official-image comparisons;
- the controlled experimental images;
- the evidence that excluded the System BIOS body and ME executable CODE / NFTP;
- the final splice logic used to build working recovery images;
- integrity verification and hash records.

It does **not** redistribute complete SPI dumps or third-party firmware binaries.

---

# 1. Scope and evidence boundary

The project involved two HP Compaq dc7900 Ultra-Slim Desktop systems, referred to as:

```text
Machine A
Machine B
```

Both originally exhibited the same late-POST freeze:

```text
POST progresses
memory count completes
F10 is captured
SETUP appears
machine freezes
BIOS Setup never actually opens
```

The binary-analysis objective was to determine which part of the 4 MiB SPI image was causally responsible.

The final evidence supports:

> **a failure in Intel ME 5.x persistent configuration / state, strongly associated with EFFS / NVAR / OEM configuration, rather than the main System BIOS body or ME executable CODE / NFTP alone.**

The project did **not** isolate one exact NVAR or one exact byte sequence as the unique trigger.

---

# 2. Full SPI size and confirmed map

The physical Flash device used by the platform is a 32 Mbit / 4 MiB SPI NOR device.

Complete image size:

```text
4,194,304 bytes
0x400000 bytes
```

Address space:

```text
0x000000–0x3FFFFF
```

Confirmed project layout:

| Region | Inclusive address range | Python-style slice | Size | Final treatment |
|---|---:|---:|---:|---|
| Flash Descriptor | `0x000000–0x000FFF` | `[0x000000:0x001000]` | 4 KiB | preserve target |
| GbE NVM | `0x001000–0x002FFF` | `[0x001000:0x003000]` | 8 KiB | preserve target |
| PDR | `0x003000–0x00AFFF` | `[0x003000:0x00B000]` | 32 KiB | preserve target |
| Intel ME | `0x00B000–0x25FFFF` | `[0x00B000:0x260000]` | `0x255000` bytes | replace with healthy HP OEM ME |
| BIOS runtime / DMI | `0x260000–0x261FFF` | `[0x260000:0x262000]` | 8 KiB | preserve target |
| System BIOS body | `0x262000–0x3FFFFF` | `[0x262000:0x400000]` | `0x19E000` bytes | use official HP BIOS v1.27 body |

Descriptor signature observed at the start of the image:

```text
5A A5 F0 0F
```

This map is the foundation of every later controlled experiment.

---

# 3. Why region boundaries matter

A complete 4 MiB SPI image is **not** interchangeable between machines.

It contains both:

```text
platform firmware
```

and:

```text
machine-specific identity / configuration
```

The final repair therefore had two opposing requirements:

```text
remove the failing persistent ME state
```

while also:

```text
retain the identity of the target machine
```

The final successful policy was:

```text
KEEP:
Descriptor
GbE / MAC
PDR
BIOS runtime / DMI

REPLACE:
complete HP OEM ME Region
System BIOS body
```

---

# 4. Machine-specific regions

## 4.1 Flash Descriptor

Range:

```text
0x000000–0x000FFF
```

The Descriptor defines the SPI region layout and access-control structure.

The project did not require replacing the target machine's Descriptor.

Because some FITC Build outputs changed Descriptor-related data unexpectedly, Descriptor preservation became an explicit safety rule.

---

## 4.2 GbE NVM / MAC

Range:

```text
0x001000–0x002FFF
```

This region contains Ethernet NVM data including the machine-specific MAC address.

Public report values are intentionally redacted.

Example public forms:

```text
Machine A:
00:22:64:XX:XX:XX

Machine B:
00:23:7D:XX:XX:XX
```

Both A and B GbE NVM images passed the project checksum test:

```text
0xBABA
```

Therefore the final recovery image preserves the target machine's original GbE NVM.

---

## 4.3 PDR

Range:

```text
0x003000–0x00AFFF
```

The project found no reason to replace the target machine's PDR as part of the validated repair.

The final image therefore preserves it unchanged.

---

## 4.4 BIOS runtime / DMI

Range:

```text
0x260000–0x261FFF
```

Size:

```text
8 KiB
```

This small area differs from the largely static System BIOS body and contains runtime / identity-related data.

It must not be treated as disposable merely because it sits adjacent to the BIOS body.

The final recovery image preserves the target machine's original 8 KiB block.

---

# 5. Trusted acquisition baseline

Binary conclusions are only meaningful if the original read itself is trustworthy.

For Machine A:

```text
SPI_A_1.bin
SPI_A_2.bin
SPI_A_3.bin
```

All three were:

```text
4,194,304 bytes
```

and:

```text
byte-for-byte identical
```

Machine A original SHA-256:

```text
209577D93CA7B70DE336864A4B771499C7234318955B9C5F1571697E9B9991E4
```

Machine A original MD5:

```text
DA0E898F5831D4521F15A1A1B285AF5E
```

For Machine B:

```text
SPI_B_1.bin
SPI_B_2.bin
SPI_B_3.bin
```

the three reads were also byte-for-byte identical.

Machine B original SHA-256:

```text
70B3A1229420AE9FB7363B2A96C9B4C506EA1CCE5D8A8B266DE2A012B80DB967
```

Machine B original MD5:

```text
968737F068528F86324618A35CC8C25C
```

Project rule:

```text
three identical full reads
↓
only then interpret later binary differences
```

Without that baseline, a difference could be caused by:

- bad clip contact;
- unstable supply;
- ISP loading;
- USB / programmer errors;
- read noise.

---

# 6. A versus B: what differed and what did not

The two machines had different machine-specific identity regions, as expected.

Examples include:

- MAC;
- serial number;
- Product / SKU;
- runtime / DMI records.

However, the project also found important similarities.

The static System BIOS body on A and B was not the main point of divergence.

The most important variable associated with the common original failure was inside the writable Intel ME Region / persistent state layer.

This became clearer through controlled replacement experiments rather than from raw A-versus-B comparison alone.

---

# 7. System BIOS body exclusion

## 7.1 B original BIOS body versus HP official 1.26

Machine B range:

```text
0x262000–0x3FFFFF
```

was compared with the corresponding System BIOS body from HP official BIOS v1.26.

Result:

```text
byte-for-byte identical
```

This is strong evidence against:

```text
random corruption of the main System BIOS body
```

as the cause of B's original POST freeze.

---

## 7.2 B runtime / DMI block

The preceding 8 KiB:

```text
0x260000–0x261FFF
```

was not treated as part of the static BIOS body.

Historical analysis of B's two 4 KiB runtime blocks found only:

```text
2 bytes difference
```

while each 4 KiB block retained a zero byte-sum checksum.

That pattern is more consistent with structured redundant records such as:

```text
generation / state / checksum data
```

than random bit corruption.

---

# 8. HP BIOS 1.26 versus 1.27

An early hypothesis was:

> HP BIOS v1.27 may also contain an updated Intel ME Region.

Direct binary comparison disproved this.

The important result is:

```text
HP official BIOS 1.26 ME Region
=
HP official BIOS 1.27 ME Region
```

They are identical for the region relevant to this project.

Therefore:

> the eventual recovery cannot be explained as "BIOS 1.27 updated ME."

The change from 1.26 to 1.27 is principally in the System BIOS body.

This distinction matters because the successful B experiment combined:

```text
healthy HP OEM ME
+
official BIOS v1.27 System BIOS body
```

but only the complete ME replacement changed the failure behavior.

---

# 9. HP official 1.26 reference image

Extracted project reference:

```text
HP_786G1_v01.26_OFFICIAL_Rom.bin
```

Size:

```text
4 MiB
```

SHA-256:

```text
EAC16981A32DBFDC312B3C12597522F4520D5A7D16BE4F3422F2983A2652EDAF
```

MD5:

```text
09B5628F42134F0C5F28702C9F3B56C7
```

This image was used for comparison and healthy-region analysis.

It should not be interpreted as a recommendation to overwrite a target machine's complete SPI from offset `0x000000`.

---

# 10. Controlled experiment 1 — BIOS-only v1.27

Generated image:

```text
DC7900_B_v1.27_VALIDATED_BIOS_ONLY.bin
```

Construction:

```text
0x000000–0x261FFF
B original retained

0x262000–0x3FFFFF
official HP BIOS v1.27 body
```

SHA-256:

```text
5357E612C21655EBA27D84190343884556CE148C21A7099A37A53D6870B3F3C4
```

MD5:

```text
3F98191396CB3DE8E9735CB0594EA2CC
```

Physical result:

```text
BIOS reports v1.27
original POST freeze remains unchanged
F10 -> SETUP still freezes
```

Conclusion:

```text
System BIOS body alone
is not the root cause
```

This experiment directly excludes the theory that simply updating the normal BIOS body repairs the documented failure.

---

# 11. Controlled experiment 2 — ME CODE / NFTP only

HP package:

```text
SP54355
```

contained:

```text
52501039.BIN
```

Intel `$MAN` headers identified version:

```text
5.2.50.1039
```

Machine B original CODE / NFTP version:

```text
5.0.1.1111
```

The controlled image replaced executable ME CODE / NFTP while retaining B's original persistent state.

Historical project subregion map:

| ME subregion | ME-relative range | SPI absolute range | Experiment action |
|---|---:|---:|---|
| EFFS | `0x002000–0x0D1FFF` | `0x00D000–0x0DCFFF` | preserve B |
| CODE | `0x0D2000–0x161FFF` | `0x0DD000–0x16CFFF` | replace |
| NFTP | `0x162000–0x241FFF` | `0x16D000–0x24CFFF` | replace |
| Tail / Padding | — | `0x24D000–0x25FFFF` | preserve |

> **Caution:** these ME subregion ranges reflect the project's historical working partitioning. Re-check them against the exact image map before reuse in another forensic experiment.

Generated image:

```text
DC7900_B_v1.27_ME_5.2.50.1039_CODE_NFTP_TEST.bin
```

SHA-256:

```text
884CC0F4C2376ED093702D9797B99E74B1DC79136C007311BD72A22BDC5A7903
```

MD5:

```text
D041C15F5E60BEE6F446020186D8F6A3
```

Physical result:

```text
still froze at exactly the same point
```

Conclusion:

```text
ME executable CODE / NFTP alone
does not explain the failure
```

This shifted the remaining causal focus toward:

```text
EFFS
NVAR
persistent configuration
runtime state
```

---

# 12. FITC evidence

Tool:

```text
Intel Flash Image Tool
5.0.0.1167
```

Platform:

```text
Boulder Creek - EL ICH10
```

On the failing B image, FITC reported:

```text
ME_CFG_DEF NVAR Not found.
KernFixedData NVAR Not found. Adding NVAR.
```

The decomposed:

```text
Configuration.txt
```

was:

```text
0 bytes
```

The same FITC version could parse a healthy HP OEM image and produce a normal OEM configuration output.

This is important because it provides a control:

```text
same FITC
same platform family
healthy image -> parses
failing image -> NVAR/configuration abnormality
```

This strongly supports a persistent ME configuration failure.

---

# 13. FITC Build output and why it was rejected

A successful FITC:

```text
Build
```

does **not** prove that the resulting image is safe to flash.

In this project, FITC Build altered Descriptor / strap-related content that was not part of the intended repair.

Therefore the generated image was rejected.

Project rule:

> **If a build tool changes a region that was not intentionally selected for change, stop and explain the difference before flashing.**

For this recovery, FITC was treated primarily as:

```text
analysis / decomposition tool
```

not an automatically trusted image builder.

---

# 14. Controlled experiment 3 — complete healthy HP OEM ME

After BIOS-only and CODE/NFTP-only experiments both failed, Machine B was tested with a complete healthy HP OEM ME Region.

The decisive replacement was:

```text
0x00B000–0x25FFFF
```

while preserving B's own:

```text
Descriptor
GbE / MAC
PDR
BIOS runtime / DMI
```

and using official HP BIOS v1.27 for:

```text
0x262000–0x3FFFFF
```

Physical result:

```text
Machine B crossed the original fixed POST freeze point
```

This is the decisive experiment.

Evidence sequence:

```text
B original
→ freezes

BIOS body only
→ freezes

ME CODE / NFTP only
→ freezes

complete healthy HP OEM ME
→ recovers
```

The minimum supported conclusion is:

> the causal defect is in persistent / configuration state that is replaced only when the full ME Region is replaced.

---

# 15. B validated successful image

Project file:

```text
DC7900_B_v1.27_HP127_ME_TEMPLATE_ONLY_TEST.bin
```

Role:

```text
B validated success baseline
A diagnostic donor
```

SHA-256:

```text
2F42B1A1B12F0D85B8DE4A8546687EBD6A11447AE91FC3FE3E88992EE8927123
```

MD5:

```text
0E05E16CA72EE1CBAD86B53A380B0448
```

This hash is useful for project traceability.

The complete binary itself is not published because it contains a combination of third-party firmware and machine-specific data.

---

# 16. Final A personalized image

The validated B image could prove whether A could boot, but it could not remain on A because it contained B-specific identity data.

The final A image was constructed as:

```text
0x000000–0x00AFFF
A original Descriptor + GbE/MAC + PDR

0x00B000–0x25FFFF
validated healthy HP OEM ME

0x260000–0x261FFF
A original BIOS runtime / DMI / identity

0x262000–0x3FFFFF
official HP BIOS v1.27 body
```

Final output:

```text
DC7900_A_FIXED_v1.27_HP_OEM_ME.bin
```

Size:

```text
4,194,304 bytes
```

SHA-256:

```text
8AFDA8295E4CE3325DD3FD409B444655CDE9D4E0F5B3FF8D79CFB301E2978683
```

MD5:

```text
0AC25FEB2BE91CD15814B710D890EB73
```

The build verification confirmed that A-specific identity remained intact.

Publicly redacted examples:

```text
MAC:
00:22:64:XX:XX:XX

Product:
FX808PA#***

Serial:
JPA84*****

Model:
HP Compaq dc7900 Ultra-Slim Desktop
```

Machine A recovered after its independent CPU / socket problem was also corrected.

---

# 17. Final splice logic

The final binary operation is deliberately simple.

Conceptually:

```python
from pathlib import Path

target = bytearray(Path("target_original.bin").read_bytes())
donor = Path("validated_hp_oem_donor.bin").read_bytes()

assert len(target) == 0x400000
assert len(donor) == 0x400000

target[0x00B000:0x260000] = donor[0x00B000:0x260000]
target[0x262000:0x400000] = donor[0x262000:0x400000]

Path("target_fixed.bin").write_bytes(target)
```

Only two ranges are replaced:

```text
ME Region
0x00B000–0x25FFFF

System BIOS body
0x262000–0x3FFFFF
```

Everything else remains from the target original.

The simplicity is intentional.

Once the causal boundaries had been established, additional automated modifications would only increase risk.

---

# 18. Required provenance validation

A constructed image should not merely have the expected size.

Each range should be checked against its intended source.

For a target image:

```text
000000–00AFFF
must equal target original

00B000–25FFFF
must equal validated healthy HP OEM ME donor

260000–261FFF
must equal target original

262000–3FFFFF
must equal official HP BIOS v1.27 body
```

The final A build produced validations of the form:

```text
[PASS] 000000-00AFFF = A original Descriptor/GbE/PDR
[PASS] 00B000-25FFFF = B validated HP OEM ME
[PASS] 260000-261FFF = A original runtime/DMI
[PASS] 262000-3FFFFF = B validated BIOS v1.27 body
```

Identity checks also passed.

---

# 19. Programming verification

Programmer software reporting:

```text
Verify OK
```

was not treated as sufficient on its own.

Required project sequence:

```text
Erase
↓
Blank Check
↓
Program
↓
Verify
↓
full physical Read Back
↓
SHA-256
```

The complete read-back file must hash identically to the intended final image.

On Windows:

```powershell
Get-FileHash .\target_fixed.bin -Algorithm SHA256
Get-FileHash .\READBACK.bin -Algorithm SHA256
```

If the hashes differ:

```text
do not boot the chip
```

---

# 20. Useful binary-comparison examples

## 20.1 Compare complete files in Python

```python
from pathlib import Path

a = Path("a.bin").read_bytes()
b = Path("b.bin").read_bytes()

assert len(a) == len(b)

diffs = [i for i, (x, y) in enumerate(zip(a, b)) if x != y]

print("different bytes:", len(diffs))

if diffs:
    print("first difference:", hex(diffs[0]))
    print("last difference :", hex(diffs[-1]))
```

---

## 20.2 Compare one region

```python
from pathlib import Path

a = Path("a.bin").read_bytes()
b = Path("b.bin").read_bytes()

start = 0x262000
end   = 0x400000

print(a[start:end] == b[start:end])
```

---

## 20.3 Verify final splice provenance

```python
from pathlib import Path

target = Path("target_original.bin").read_bytes()
donor  = Path("validated_donor.bin").read_bytes()
fixed  = Path("target_fixed.bin").read_bytes()

assert len(target) == len(donor) == len(fixed) == 0x400000

assert fixed[0x000000:0x00B000] == target[0x000000:0x00B000]
assert fixed[0x00B000:0x260000] == donor [0x00B000:0x260000]
assert fixed[0x260000:0x262000] == target[0x260000:0x262000]
assert fixed[0x262000:0x400000] == donor [0x262000:0x400000]

print("PASS")
```

These snippets are intentionally minimal.

A production builder should additionally validate:

- exact expected input SHA-256;
- exact output size;
- identity retention;
- expected output SHA-256 where known.

---

# 21. Hash reference table

| File | Size | SHA-256 | Result / role |
|---|---:|---|---|
| `SPI_A_1.bin` | 4 MiB | `209577D93CA7B70DE336864A4B771499C7234318955B9C5F1571697E9B9991E4` | A original |
| `SPI_B_1.bin` | 4 MiB | `70B3A1229420AE9FB7363B2A96C9B4C506EA1CCE5D8A8B266DE2A012B80DB967` | B original |
| `HP_786G1_v01.26_OFFICIAL_Rom.bin` | 4 MiB | `EAC16981A32DBFDC312B3C12597522F4520D5A7D16BE4F3422F2983A2652EDAF` | HP 1.26 reference |
| `DC7900_B_v1.27_VALIDATED_BIOS_ONLY.bin` | 4 MiB | `5357E612C21655EBA27D84190343884556CE148C21A7099A37A53D6870B3F3C4` | BIOS-only; still freezes |
| `DC7900_B_v1.27_ME_5.2.50.1039_CODE_NFTP_TEST.bin` | 4 MiB | `884CC0F4C2376ED093702D9797B99E74B1DC79136C007311BD72A22BDC5A7903` | CODE/NFTP; still freezes |
| `DC7900_B_v1.27_HP127_ME_TEMPLATE_ONLY_TEST.bin` | 4 MiB | `2F42B1A1B12F0D85B8DE4A8546687EBD6A11447AE91FC3FE3E88992EE8927123` | B validated success |
| `DC7900_A_FIXED_v1.27_HP_OEM_ME.bin` | 4 MiB | `8AFDA8295E4CE3325DD3FD409B444655CDE9D4E0F5B3FF8D79CFB301E2978683` | A validated success |
| `DC7900_B_127_CLEAN_TEST_V2.bin` | 4 MiB | `91EE3D03E1DF23D31515B7E642713BD0DE6591D05D5ADF496DBB7034D07FBE9B` | deprecated |
| `DC7900_A_127_CLEAN_TEST_V2.bin` | 4 MiB | `1A403F5DAB7FD3631FE6BEEBE3A0343C425BCA6A84C70362BB3FE279176AF4DE` | deprecated |

---

# 22. SP54355 reference

HP package:

```text
sp54355.exe
```

SHA-256:

```text
E713F0358B06FFFDE5AC9A0B807EF1E58054B3AAF03AB5899749DF92905182BD
```

Contained payload:

```text
52501039.BIN
```

Size:

```text
1,510,212 bytes
0x170B44
```

SHA-256:

```text
298087A41EDD12013C78CE149F5ECCDECC156FE4238E2C5A75800B2118C43A65
```

Identified ME version:

```text
5.2.50.1039
```

Its main forensic value in this project was the controlled demonstration that replacing newer executable ME CODE / NFTP did **not** eliminate the failure while the original persistent state remained.

---

# 23. Deprecated experimental images

Do not reuse:

```text
DC7900_A_127_CLEAN_TEST_V2.bin
DC7900_B_127_CLEAN_TEST_V2.bin
```

These came from an early broad multi-variable replacement route.

Because several regions were changed simultaneously, they are poor causal experiments and were abandoned.

They remain documented only so that future work does not accidentally treat them as validated recovery images.

---

# 24. What is proven versus inferred

## Directly proven

```text
B original BIOS body matches official 1.26
```

```text
BIOS-only v1.27 still freezes
```

```text
ME CODE / NFTP-only replacement still freezes
```

```text
complete healthy HP OEM ME replacement recovers B
```

```text
A recovers by the same firmware route after its separate CPU/socket fault is corrected
```

---

## Strongly supported

The fault is located in the persistent ME configuration / state layer:

```text
EFFS
NVAR
OEM configuration
```

because independent evidence converges there.

---

## Not proven

The project does not yet identify:

```text
one exact failing NVAR
```

or:

```text
one exact minimal byte pattern
```

that alone reproduces the POST deadlock.

That remains a separate ME5 forensic-research problem.

---

# 25. Why complete SPI dumps are not published

A complete target dump can contain:

- MAC address;
- serial number;
- Product / SKU;
- UUID;
- DMI data;
- machine-specific runtime state.

It also contains third-party HP / Intel firmware.

For that reason this repository currently publishes:

```text
layout
offsets
hashes
analysis
reconstruction logic
verification procedure
```

rather than complete machine dumps.

This approach preserves reproducibility without unnecessarily redistributing machine identity data or third-party firmware binaries.

---

# 26. Final binary model

The final recovery can be represented as:

```text
TARGET ORIGINAL
┌──────────────────────────────────────────────┐
│ Descriptor + GbE + PDR                      │  KEEP
├──────────────────────────────────────────────┤
│ failing Intel ME Region                     │  REPLACE
├──────────────────────────────────────────────┤
│ target BIOS runtime / DMI                   │  KEEP
├──────────────────────────────────────────────┤
│ System BIOS body                            │  REPLACE
└──────────────────────────────────────────────┘
```

becoming:

```text
FINAL TARGET IMAGE
┌──────────────────────────────────────────────┐
│ target Descriptor + GbE/MAC + PDR           │
├──────────────────────────────────────────────┤
│ healthy HP OEM Intel ME Region              │
├──────────────────────────────────────────────┤
│ target BIOS runtime / DMI                   │
├──────────────────────────────────────────────┤
│ official HP BIOS v1.27 System BIOS body     │
└──────────────────────────────────────────────┘
```

Exact splice ranges:

```text
KEEP     0x000000–0x00AFFF
REPLACE  0x00B000–0x25FFFF
KEEP     0x260000–0x261FFF
REPLACE  0x262000–0x3FFFFF
```

That is the binary core of the validated repair.

---

## Related documentation

- [Recovery procedure](recovery-procedure.md)
- [Full English technical report](full-report-en.md)
- [中文完整技术报告](full-report-zh-CN.md)
- [Image evidence archive](../images/README.md)
