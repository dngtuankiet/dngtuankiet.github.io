---
layout: single
title: "Introduction to RISC-V"
permalink: /tutorials/risc-v-introduction/
author_profile: false
sidebar:
  nav: "tutorials"
toc: true
toc_label: "On This Page"
toc_icon: "list"
toc_sticky: true
---

## What is RISC-V?

RISC-V (pronounced "risk-five") is an open-source instruction set architecture (ISA) based on established reduced instruction set computer (RISC) principles. Unlike proprietary ISAs, RISC-V is free and open, enabling anyone to implement processors without licensing fees.

## Key Features

### Open and Free
- No licensing fees or royalties
- Open specification available to everyone
- Community-driven development

### Modular Design
- Base integer ISA with optional extensions
- Customizable for different applications
- Scalable from microcontrollers to supercomputers

### Simple and Clean
- Small base instruction set
- Regular instruction encoding
- Easy to implement and optimize

## RISC-V Base ISAs

RISC-V defines several base integer ISAs:

### RV32I
The 32-bit base integer instruction set. It provides:
- 32 general-purpose registers
- Load/store architecture
- Simple arithmetic and logical operations

### RV64I
The 64-bit base integer instruction set, extending RV32I to support:
- 64-bit address space
- 64-bit integer operations
- Backward compatibility with RV32I

## Standard Extensions

RISC-V uses a letter-based naming convention for extensions:

- **M**: Integer multiplication and division
- **A**: Atomic instructions
- **F**: Single-precision floating-point
- **D**: Double-precision floating-point
- **C**: Compressed instructions (16-bit)

## Getting Started

To start working with RISC-V, you'll need:

1. A RISC-V toolchain (compiler, assembler, linker)
2. A simulator or emulator (QEMU, Spike)
3. Optional: FPGA board or RISC-V hardware

## Next Steps

Continue to the next tutorial to learn about [Getting Started with Chipyard](/tutorials/chipyard-getting-started/), a complete SoC design framework built around RISC-V.

## Resources

- [RISC-V International](https://riscv.org/)
- [RISC-V Specifications](https://riscv.org/technical/specifications/)
- [RISC-V Software Tools](https://github.com/riscv-collab/riscv-gnu-toolchain)
