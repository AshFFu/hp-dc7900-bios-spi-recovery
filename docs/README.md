# Documentation

[English](README.md) | [简体中文](README.zh-CN.md)

This directory contains the detailed technical documentation for the HP Compaq dc7900 BIOS / SPI Flash recovery project.

The documentation is organized to preserve both the complete investigation history and a concise, reproducible recovery procedure.

## Planned documents

### Full technical report

- [`full-report-en.md`](full-report-en.md)  
  Complete English investigation and recovery report.

- [`full-report-zh-CN.md`](full-report-zh-CN.md)  
  完整中文故障调查与恢复报告。

These reports will cover the complete process from the original failure symptoms to final hardware recovery and verification.

### Recovery procedure

- [`recovery-procedure.md`](recovery-procedure.md)

A condensed step-by-step procedure intended for future recovery of another machine with the same confirmed failure mode.

### SPI flash analysis

- [`spi-flash-analysis.md`](spi-flash-analysis.md)

Technical analysis of the SPI flash contents, firmware regions, binary differences, integrity verification, and the logic used to construct the recovery image.

## Documentation principles

The documentation in this repository follows several principles:

1. **Evidence before conclusion**  
   Conclusions should be supported by measurements, repeated firmware reads, binary comparison, hardware observations, or successful recovery results.

2. **Preserve the original dump**  
   The original SPI flash contents must always be backed up before modification.

3. **Separate observation from hypothesis**  
   Confirmed findings and explanations that remain theoretical should be clearly distinguished.

4. **Reproducibility**  
   Another user with the same motherboard and failure condition should be able to understand and reproduce the diagnostic process.

5. **Machine-specific data awareness**  
   Complete SPI images may contain identifiers or configuration data unique to an individual machine. Such data must be identified before firmware files are redistributed.

## Language

The main technical report will be maintained in both English and Simplified Chinese.

English is used as the primary public-facing language of the repository to make the project easier to discover internationally, while the Chinese version preserves the complete technical record in the language in which the original investigation was performed.
