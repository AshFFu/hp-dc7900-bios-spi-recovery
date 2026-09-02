# Firmware Publication Notes

[Repository home](../README.md) | [中文首页](../README.zh-CN.md) | [SPI flash analysis](../docs/spi-flash-analysis.md) | [Recovery procedure](../docs/recovery-procedure.md)

This directory intentionally does **not** contain complete SPI dumps or reconstructed recovery BIN files.

The dc7900 SPI images used during this project include a mixture of:

- HP / Intel firmware;
- machine-specific configuration;
- MAC address data;
- DMI / serial / product information;
- runtime and persistent state.

For that reason, this repository publishes the recovery method, region layout, hashes, and reconstruction logic rather than complete machine-specific firmware images.

---

## What is published

The repository provides:

- exact SPI Region boundaries and offsets;
- hashes of important project images;
- controlled experiment results;
- the final recovery-image splice logic;
- a sanitized verified builder;
- read-back and integrity-verification procedures;
- image evidence and complete technical documentation.

See:

- [`../docs/spi-flash-analysis.md`](../docs/spi-flash-analysis.md)
- [`../docs/recovery-procedure.md`](../docs/recovery-procedure.md)
- [`../tools/BUILD_DC7900_A_FINAL_ONECLICK_V2.cmd`](../tools/BUILD_DC7900_A_FINAL_ONECLICK_V2.cmd)

---

## What is not published

The following are intentionally not included:

```text
SPI_A_1.bin
SPI_B_1.bin
DC7900_B_v1.27_VALIDATED_BIOS_ONLY.bin
DC7900_B_v1.27_ME_5.2.50.1039_CODE_NFTP_TEST.bin
DC7900_B_v1.27_HP127_ME_TEMPLATE_ONLY_TEST.bin
DC7900_A_FIXED_v1.27_HP_OEM_ME.bin
```

Complete machine dumps may expose identifiers unique to one physical system.

Reconstructed recovery images may also combine:

```text
target-machine data
+
HP / Intel firmware
```

and therefore should not be treated as generic redistribution files.

---

## Important project hashes

These values are retained for traceability and reproduction.

### Machine A original

```text
SHA-256
209577D93CA7B70DE336864A4B771499C7234318955B9C5F1571697E9B9991E4
```

### Machine B original

```text
SHA-256
70B3A1229420AE9FB7363B2A96C9B4C506EA1CCE5D8A8B266DE2A012B80DB967
```

### HP official BIOS 1.26 reference image

```text
HP_786G1_v01.26_OFFICIAL_Rom.bin

SHA-256
EAC16981A32DBFDC312B3C12597522F4520D5A7D16BE4F3422F2983A2652EDAF
```

### B BIOS-only v1.27 test

```text
DC7900_B_v1.27_VALIDATED_BIOS_ONLY.bin

SHA-256
5357E612C21655EBA27D84190343884556CE148C21A7099A37A53D6870B3F3C4
```

Result:

```text
still freezes
```

### B ME CODE / NFTP 5.2.50.1039 test

```text
DC7900_B_v1.27_ME_5.2.50.1039_CODE_NFTP_TEST.bin

SHA-256
884CC0F4C2376ED093702D9797B99E74B1DC79136C007311BD72A22BDC5A7903
```

Result:

```text
still freezes
```

### B validated successful donor / baseline

```text
DC7900_B_v1.27_HP127_ME_TEMPLATE_ONLY_TEST.bin

SHA-256
2F42B1A1B12F0D85B8DE4A8546687EBD6A11447AE91FC3FE3E88992EE8927123
```

Result:

```text
recovery successful
```

### Machine A final personalized recovery image

```text
DC7900_A_FIXED_v1.27_HP_OEM_ME.bin

SHA-256
8AFDA8295E4CE3325DD3FD409B444655CDE9D4E0F5B3FF8D79CFB301E2978683
```

Result:

```text
recovery successful
```

---

## HP SP54355 / Intel ME 5.2.50.1039 reference

The project used HP package:

```text
sp54355.exe
```

SHA-256:

```text
E713F0358B06FFFDE5AC9A0B807EF1E58054B3AAF03AB5899749DF92905182BD
```

It contains:

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

Identified Intel ME version:

```text
5.2.50.1039
```

Its value in this project was forensic rather than as a complete recovery image:

```text
replace ME CODE / NFTP only
→ original POST freeze remained
```

This helped exclude executable ME CODE / NFTP alone as the root cause.

---

## Final reconstruction policy

The validated recovery-image structure is:

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

A complete donor image should not be written directly to another machine.

---

## Why complete BIN files are risky even on the same model

Two dc7900 systems can share the same board family while still carrying different:

- MAC addresses;
- serial numbers;
- Product / SKU values;
- UUID / DMI data;
- writable runtime state.

Therefore:

```text
same model
≠
safe full-image interchangeability
```

The correct recovery method is to preserve the target machine's own identity-bearing regions.

---

## Deprecated project images

Do not use:

```text
DC7900_A_127_CLEAN_TEST_V2.bin
DC7900_B_127_CLEAN_TEST_V2.bin
```

These were produced during an early multi-variable experiment and are not validated recovery images.

They remain documented only to prevent accidental reuse.

---

## Source and redistribution note

HP and Intel firmware remain the property of their respective rights holders.

This repository does not claim ownership of HP or Intel firmware and does not provide complete redistributed firmware packages or complete machine SPI images.

Users should obtain vendor firmware from legitimate original sources and perform their own reconstruction against a verified dump from the target machine.

---

## Practical rule

For another machine with the same confirmed failure mode:

```text
do not search for a complete "fixed BIN"
```

Instead:

```text
read and verify the target original
preserve target identity regions
obtain compatible healthy OEM firmware
construct the recovery image
verify every copied region
program
read back
compare SHA-256
```

That procedure is documented in:

[`../docs/recovery-procedure.md`](../docs/recovery-procedure.md)
