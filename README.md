# HP Compaq dc7900 BIOS / SPI Flash Recovery

[English](README.md) | [简体中文](README.zh-CN.md)

A documented investigation and successful recovery of an HP Compaq dc7900 system affected by a BIOS / SPI flash-related failure.

This repository records the complete diagnostic and recovery process, including hardware identification, SPI flash access, firmware dump analysis, binary comparison, external programming, verification, and final recovery.

The goal of this project is not merely to provide a replacement firmware image, but to document a reproducible method for understanding, diagnosing, and recovering the failure.

## Project status

**Completed — recovery successfully verified on real hardware.**

A complete technical report, photographs, binary analysis, verification data, and recovery procedure will be published in this repository.

## Hardware

* System: HP Compaq dc7900
* Motherboard identifiers:

  * HP 462433-001
  * HP 460954-001
* Firmware storage: onboard SPI flash
* External programmer used during recovery: EZP2019

## Repository contents

The repository is being organized into the following sections:

* `docs/` — complete technical reports and recovery documentation
* `images/` — motherboard, SPI flash, programmer and diagnostic photographs
* `analysis/` — firmware structure, hashes and binary comparison data
* `firmware/` — firmware-related documentation and publication notes

## Important

Do **not** write any firmware image from this project directly to another machine without first confirming:

* motherboard revision
* SPI flash device and capacity
* firmware layout
* machine-specific data
* integrity of the original dump
* backup and recovery capability

SPI flash programming can render a motherboard unbootable if an incorrect image or procedure is used.

Always create and verify multiple backups of the original flash contents before modifying or writing the chip.

## Firmware files

Firmware binaries are being reviewed separately before publication.

Complete SPI flash images may contain manufacturer-owned firmware as well as machine-specific information. For this reason, binary files will not be published until their contents, redistribution implications, and device-specific data have been examined.

Hashes, structural analysis, modification details, and reproducible recovery information will be documented regardless of whether complete binary images are distributed.

## Documentation

Detailed documentation will be added progressively:

* Full investigation and recovery report
* SPI flash identification and access
* External programmer connection
* Dump verification
* Firmware binary analysis
* Failure analysis
* Recovery image construction
* Flash write and verification procedure
* Post-recovery validation
* Fast recovery checklist

A complete Simplified Chinese version will be maintained alongside the English documentation.

## Disclaimer

This project documents an independent hardware repair and research process.

HP, Intel, and other trademarks belong to their respective owners. This project is not affiliated with or endorsed by HP or Intel.

Firmware modification and external SPI programming involve a risk of permanent hardware failure. The information in this repository is provided for technical research and repair reference.
