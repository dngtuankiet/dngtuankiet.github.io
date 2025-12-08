---
layout: single
title: "Introduction to RISC-V"
permalink: /tutorials/I-risc-v-introduction/
date: 2025-01-01
author_profile: false
sidebar:
  nav: "tutorials"
toc: true
toc_label: "On This Page"
toc_icon: "list"
toc_sticky: true
classes: text-justify
---

## What is RISC-V?

RISC-V (pronounced "risk-five") is an open-source instruction set architecture (ISA) based on established reduced instruction set computer (RISC) principles. Unlike proprietary ISAs, RISC-V is free and open, enabling anyone to implement processors without licensing fees.

### Official Documentation

- **RISC-V Specifications (Latest Releases):**  
  https://riscv.org/technical/specifications/
- **RISC-V Profiles, Extensions, and Standards:**  
  https://riscv.org/technical/technical-documents/
- **RISC-V International GitHub:**  
  https://github.com/riscv

---

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

---

## RISC-V Base ISAs

### RV32I
The 32-bit base integer ISA. It provides:
- 32 general-purpose registers  
- Load/store architecture  
- Basic arithmetic and logical operations  

### RV64I
The 64-bit base integer ISA, extending RV32I to support:
- 64-bit address space  
- 64-bit integer operations  
- Backward compatibility with RV32I  

---

## Standard Extensions

RISC-V uses a letter-based naming convention for extensions:

- **M** – Integer multiplication and division  
- **A** – Atomic memory operations  
- **F** – Single-precision floating-point  
- **D** – Double-precision floating-point  
- **C** – Compressed 16-bit instructions  

A full list of extensions is available here:  
https://riscv.org/technical/specifications/

---

## Popular RISC-V Open-Source Platforms

RISC-V has inspired a large ecosystem of processor implementations and system-level design frameworks.

### Open-Source RISC-V Cores

- **[Rocket Chip (Berkeley RISC-V)](https://github.com/chipsalliance/rocket-chip)**  
  A parameterizable in-order RISC-V core widely used in academia and industry.

- **[BOOM (Berkeley Out-of-Order Machine)](https://github.com/riscv-boom/riscv-boom)**  
  A high-performance out-of-order RISC-V core.

- **[PULP RI5CY & CV32E40P](https://github.com/openhwgroup/cv32e40p)**  
  Designed for energy-efficient embedded systems.

- **[PicoRV32](https://github.com/cliffordwolf/picorv32)**  
  A compact RV32I core ideal for FPGA implementations.

### Full SoC Frameworks

- **[Chipyard](https://chipyard.readthedocs.io)** (Berkeley)  
  A complete SoC design framework integrating Rocket, BOOM, DSP blocks, accelerators, and more.

- **[PULP Platform](https://pulp-platform.org/)**  
  Scalable multi-core RISC-V SoCs with rich peripheral and accelerator support.

- **[OpenTitan](https://opentitan.org/)**  
  A silicon-root-of-trust project using RISC-V for secure hardware.

These projects provide excellent reference designs for anyone learning or prototyping RISC-V systems.

---

## Getting Started

To start working with RISC-V, you'll need:

1. A RISC-V toolchain (compiler, assembler, linker)  
   → https://github.com/riscv-collab/riscv-gnu-toolchain  
2. A simulator or emulator (e.g., Spike, QEMU)  
   → https://github.com/riscv-software-src/riscv-isa-sim  
   → https://www.qemu.org/  
3. Optional: FPGA board or RISC-V hardware (HiFive, Microchip PolarFire, etc.)

---

## Next Steps

Continue to the next tutorial to learn about [Getting Started with Chipyard](/tutorials/chipyard-getting-started/), a complete SoC design framework built around RISC-V.

---

## Resources

- **RISC-V International:** https://riscv.org/  
- **RISC-V Specifications:** https://riscv.org/technical/specifications/  
- **RISC-V Software Tools:** https://github.com/riscv-collab/riscv-gnu-toolchain  
