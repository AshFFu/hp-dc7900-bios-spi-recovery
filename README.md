# HP Compaq dc7900 BIOS / SPI Flash Recovery

[English](README.md) | [简体中文](README.zh-CN.md)

A documented investigation and successful recovery of two HP Compaq dc7900 Ultra-Slim Desktop systems affected by the same late-POST freeze associated with Intel ME 5.x persistent configuration / state.

The project records the complete diagnostic and recovery process, including hardware identification, SPI Flash access, repeated dump verification, binary comparison, controlled firmware experiments, external programming, board-level repair, final firmware reconstruction, and post-recovery validation.

The goal is not to provide a generic firmware image for direct flashing, but to document a reproducible method for understanding, diagnosing, and recovering this confirmed failure mode while preserving machine-specific data.

## Project status

**Completed — recovery validated on two physical machines.**

Both machines can now:

- complete POST;
- enter F10 BIOS Setup;
- boot an operating system;
- enumerate HECI / MEI normally under Linux.

The original late-POST freeze has not been reproduced after repair.

## Hardware

- System: HP Compaq dc7900 Ultra-Slim Desktop
- BIOS family: `786G1`
- Motherboard identifiers:
  - HP 462433-001
  - HP 460954-001
- Firmware storage: 4 MiB onboard SPI NOR Flash
- External programmer used during recovery: EZP2019+

## Main finding

The successful experiments isolate the original failure to the Intel ME 5.x persistent configuration / state layer, strongly associated with:

```text
EFFS
NVAR
OEM configuration
```

The following were directly excluded as the primary cause of the original freeze:

- the main System BIOS body;
- simply running BIOS 1.26 instead of 1.27;
- ME executable CODE / NFTP alone;
- a physically dead HECI controller.

The validated recovery principle is:

```text
preserve target Descriptor + GbE/MAC + PDR
replace complete unhealthy ME Region with healthy HP OEM ME
preserve target BIOS runtime / DMI
use official HP BIOS v1.27 System BIOS body
```

## Repository contents

```text
docs/
  README.md
  full-report-en.md
  full-report-zh-CN.md
  recovery-procedure.md
  spi-flash-analysis.md

images/
  README.md
  10 evidence images

tools/
  BUILD_DC7900_A_FINAL_ONECLICK_V2.cmd
```

### Documentation

- [`docs/full-report-en.md`](docs/full-report-en.md) — complete English technical report
- [`docs/full-report-zh-CN.md`](docs/full-report-zh-CN.md) — 完整中文技术报告
- [`docs/recovery-procedure.md`](docs/recovery-procedure.md) — reproducible recovery procedure
- [`docs/spi-flash-analysis.md`](docs/spi-flash-analysis.md) — SPI layout, binary comparisons, experimental images, hashes, and splice logic
- [`images/README.md`](images/README.md) — image evidence archive

### Tool

- [`tools/BUILD_DC7900_A_FINAL_ONECLICK_V2.cmd`](tools/BUILD_DC7900_A_FINAL_ONECLICK_V2.cmd)

The public script is a sanitized version of the final builder used during the project. It preserves the target identity-bearing regions byte-for-byte and does not print or hard-code the machine's private MAC / Product / Serial identifiers.

## Important

Do **not** write any firmware image from this project directly to another machine without first confirming:

- motherboard revision;
- SPI Flash device and capacity;
- firmware layout;
- machine-specific data;
- integrity of the original dump;
- backup and recovery capability.

Always create and verify multiple backups of the original Flash before modifying or writing the chip.

For this project, a trustworthy baseline required:

```text
three complete 4 MiB reads
+
identical SHA-256
```

After programming, the final image was also verified by a complete physical read-back and SHA-256 comparison.

## Firmware binaries

Complete SPI dumps and reconstructed recovery BIN files are **not published** in this repository.

Reasons include:

- machine-specific identifiers and runtime data;
- HP / Intel firmware redistribution considerations;
- the risk of users flashing a machine-specific full image to incompatible hardware.

Instead, the repository publishes:

- exact region layout;
- offsets;
- hashes;
- controlled experiment results;
- reconstruction logic;
- a sanitized builder;
- recovery and verification procedures.

## Evidence archive

The repository includes ten photographs and diagnostic screenshots covering:

- the SPI Flash device;
- EZP2019+ programmer;
- U19 / U21 pad damage;
- signal mapping;
- Intel FITC analysis;
- first successful POST after full OEM ME replacement;
- transient HECI / MEBx errors;
- Linux HECI / MEI validation.

See [`images/README.md`](images/README.md).

## Disclaimer

This repository documents an independent hardware repair and research project.

HP, Intel, and other trademarks belong to their respective owners. This project is not affiliated with or endorsed by HP or Intel.

Firmware modification, soldering, and external SPI programming can render hardware unbootable. The information is provided for technical research and repair reference.
