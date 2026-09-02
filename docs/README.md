# Documentation

[English](README.md) | [简体中文完整报告](full-report-zh-CN.md)

This directory contains the completed technical documentation for the HP Compaq dc7900 BIOS / SPI Flash recovery project.

## Documents

### [`full-report-en.md`](full-report-en.md)

Complete English investigation and recovery report.

It preserves the complete evidence chain:

```text
symptom
→ acquisition
→ exclusion
→ root cause
→ recovery
→ validation
→ SOP
```

### [`full-report-zh-CN.md`](full-report-zh-CN.md)

完整中文故障调查与恢复报告。

This is the primary project record in the language used during the original investigation.

### [`recovery-procedure.md`](recovery-procedure.md)

A standalone recovery manual intended for another dc7900 with the same confirmed failure mode.

It covers:

- applicability;
- SPI acquisition;
- machine-identity preservation;
- image reconstruction;
- programming and read-back verification;
- U19 / U21 hardware repair;
- first boot;
- Linux HECI / MEI validation;
- hardware-diagnosis branch for black-screen failures.

### [`spi-flash-analysis.md`](spi-flash-analysis.md)

Dedicated binary-analysis record covering:

- confirmed SPI Region layout;
- exact offsets and sizes;
- A / B / HP official-image comparisons;
- BIOS-only and CODE/NFTP-only exclusion experiments;
- FITC evidence;
- final splice logic;
- project hashes;
- proven versus inferred conclusions.

## Related material

- [`../images/README.md`](../images/README.md) — image evidence archive
- [`../tools/BUILD_DC7900_A_FINAL_ONECLICK_V2.cmd`](../tools/BUILD_DC7900_A_FINAL_ONECLICK_V2.cmd) — sanitized verified recovery-image builder

## Documentation principles

The repository follows five rules:

1. **Evidence before conclusion**  
   Conclusions are tied to measurements, repeated firmware reads, binary comparison, hardware observation, or successful physical recovery.

2. **Preserve the original dump**  
   The original SPI contents must be backed up and verified before any modification.

3. **Separate observation from hypothesis**  
   Confirmed findings and mechanism-level interpretations are explicitly distinguished.

4. **Reproducibility**  
   Another user with the same board and matching failure mode should be able to reconstruct the diagnostic logic and recovery method.

5. **Machine-specific data awareness**  
   Complete SPI images can contain MAC, serial, UUID, DMI, and other unique data. Public material is therefore sanitized and complete machine dumps are not published.

## Language

The full technical report is maintained in both English and Simplified Chinese.

English is used as the main public-facing language for international discoverability, while the Chinese report preserves the complete technical record in the language in which the investigation was originally performed.
