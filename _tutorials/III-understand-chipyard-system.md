---
layout: single
title: "Understanding Chipyard System"
permalink: /tutorials/III-understand-chipyard-system/
date: 2025-01-03
author_profile: false
sidebar:
  nav: "tutorials"
toc: true
toc_label: "On This Page"
toc_icon: "list"
toc_sticky: true
classes: text-justify
---

## Overview

Chipyard is a sophisticated framework that integrates multiple components to enable RISC-V SoC design, simulation, and deployment. Understanding the architecture and organization of Chipyard is essential for effectively customizing and extending your designs. This page provides an overview of the Chipyard system structure, including its generators, tools, and architectural concepts. 

**Rocket Chip** is the key system for further customization. This page also describes some key aspects including:
- Rocket system architecture
- TileLink bus interface

## 1. Chipyard Architecture

At its core, Chipyard is built around the concept of **generators** - parameterized RTL designs written using meta-programming techniques enabled by the Chisel hardware description language. These generators allow for flexible, automated integration of complex hardware designs that can be configured for various use cases without manual RTL modifications.

### 1.1 Core Components

Chipyard organizes its ecosystem into several key categories:

#### 1.1.1 RTL Generators

Generators are the heart of Chipyard, providing configurable hardware components that can be instantiated with different parameters. Key generators include:

**Processor Cores:**
- **Rocket Core**: An in-order 5-stage RISC-V core serving as the default processor
- **BOOM (Berkeley Out-of-Order Machine)**: A superscalar out-of-order RISC-V core for high-performance applications
- **Shuttle Core**: A superscalar in-order RISC-V core
- **CVA6**: An in-order RISC-V core written in SystemVerilog (previously Ariane)
- **Ibex**: A 32-bit in-order RISC-V core in SystemVerilog
- **VexiiRiscv**: A dual-issue in-order 64-bit core implemented in SpinalHDL

**Accelerators:**
- **Gemmini**: A systolic array-based matrix multiplication accelerator optimized for neural networks
- **Saturn**: A vector unit generator
- **FFT Generator**: Configurable Fast Fourier Transform accelerator
- **NVDLA**: Deep learning accelerator

**System Components:**
- **Constellation**: Network-on-Chip (NoC) interconnect generator
- **IceNet**: Network Interface Controller achieving up to 200 Gbps
- **rocket-chip-blocks**: Peripheral devices including UART, SPI, JTAG, I2C, PWM
- **testchipip**: Testing utilities and chip interfacing components

#### 1.1.2 Development Tools

**Chisel & FIRRTL:**
- **Chisel**: A hardware description library embedded in Scala enabling RTL meta-programming
- **FIRRTL**: An intermediate representation that allows circuit manipulation between Chisel elaboration and Verilog generation
- **Tapeout-Tools**: FIRRTL transformations for digital circuit manipulation without changing source RTL

**Dsptools:** Chisel library for signal processing hardware development

#### 1.1.3 Toolchains

**riscv-tools**: Comprehensive collection including:
- Compiler and assembler toolchains
- Spike functional ISA simulator
- Berkeley Boot Loader (BBL) and proxy kernel
- Essential for developing and executing RISC-V software

#### 1.1.4 Simulation Environments

- **Verilator**: Open-source Verilog simulator for software RTL simulation
- **VCS**: Synopsys proprietary simulator (requires license)
- **FireSim**: FPGA-accelerated simulation platform on AWS EC2 F1 instances, providing fast (10s-100s MHz) deterministic simulation

#### 1.1.5 Software Tools

- **FireMarshal**: Default workload generation tool for creating software to run on Chipyard platforms
- **Baremetal-IDE**: Integrated development environment for baremetal C/C++ programs

#### 1.1.6 VLSI & Prototyping

- **Hammer**: VLSI flow providing abstraction between physical design concepts and vendor-specific EDA tools
- **FPGA Prototyping**: Support for various FPGA boards (Xilinx Arty 35T, VCU118) using SiFive's fpga-shells

## 2. Rocket Chip System Architecture
Personally, I've been using **Rocket Chip** for a majority of my research. This tutorial will extend  the customization based on the Rocket system architecture. As the **Rocket Chip** generator forms the foundation of most Chipyard SoCs, understanding its architecture is crucial for system customization. Below is the figure describing the system.

<p align="center">
   <img src="/assets/images/tutorials/rocket-chip.png" alt="Rocket Chip" width="600">
</p>


### 2.1 Tile-Based Organization

Rocket Chip organizes processors into **tiles**. Each Rocket core is wrapped in a tile with its corressponding components:
- A processor core (Rocket, BOOM, or custom)
- L1 cache (instruction & data)
- Page table walker (PTW)
- Optional RoCC (Rocket Custom Coprocessor) accelerator interface

### 2.2 Memory Hierarchy

The memory system is organized hierarchically using TileLink interconnects:

1. **SystemBus**: Connects tiles to the rest of the system
2. **L2 Cache Banks**: Shared last-level cache accessible by all tiles
3. **MemoryBus**: Connects L2 to the DRAM controller via TileLink-to-AXI4 converter
4. **FrontendBus**: Dedicated bus for DMA devices with direct memory access

### 2.3 MMIO Peripheral Buses

Memory-mapped I/O peripherals connect through two specialized buses:

**ControlBus** - Standard system peripherals:
- **BootROM**: Contains first-stage bootloader and Device Tree
- **PLIC (Platform-Level Interrupt Controller)**: Aggregates and routes device interrupts
- **CLINT (Core-Local Interrupts)**: Software and timer interrupts per CPU
- **Debug Unit**: External control interface via DMI or JTAG

**PeripheryBus** - Additional peripherals:
- Network Interface Controllers
- Block storage devices
- External AXI4 ports for vendor IP integration

### 2.4 DMA Architecture

Direct Memory Access devices connect to the **FrontendBus**, allowing them to read/write directly from the memory system without CPU intervention. The bus supports both native TileLink devices and vendor AXI4 devices through protocol converters.

## 3. TileLink & Diplomacy

### 3.1 TileLink Protocol

TileLink is the cache-coherent memory protocol used throughout Chipyard systems. It defines how modules communicate:
- **Caches** maintain coherency
- **Memory controllers** serve requests
- **Peripherals** respond to MMIO accesses
- **DMA devices** initiate transactions

TileLink supports multiple conformance levels from simple memory-mapped interfaces to full cache coherence protocols. You can find more information about TileLink interface in the references.

### 3.2 Diplomacy Framework

**Diplomacy** is a meta-programming framework for negotiating connections between generators during elaboration:

1. **Two-Phase Elaboration**: 
   - Phase 1: Modules negotiate parameters (bus widths, address ranges, supported operations)
   - Phase 2: Actual hardware is generated based on negotiated parameters

2. **Node Types**: Different roles in the network
   - **Client Nodes**: Initiate transactions (CPU, DMA)
   - **Manager Nodes**: Service requests (memory, peripherals)
   - **Adapter/Nexus Nodes**: Transform or route connections

3. **Diplomatic Connections**: Special operators (`:=`, `:*=`, etc.) connect nodes and automatically handle parameter negotiation

This automatic negotiation eliminates manual configuration and reduces integration errors when composing complex systems.

## 4. Directory Structure

Chipyard organizes its source code as followed:

```
chipyard/
├── generators/          # RTL generator source code
│   ├── rocket-chip/    # Rocket Chip SoC generator
│   ├── boom/           # BOOM core
│   ├── chipyard/       # Top-level integration layer, config mixins, test harnesses
|   ├── testchipip/     # Chipyard utilities (TSI, SerialTL, boot ROM helper)
|   ├── diplomacy/      # TileLink/AXI interconnect generation framework
│   ├── constellation/  # NoC generator
│   ├── gemmini/, ara/, nvdla/     # Hardware accelerators
│   └── ...
├── fpga/               # Experimental FPGA flow
|   ├── fpga-shells/    # FPGA board support
|   ├── src/main/
|       ├── scala       # Board-specific Scala configs (arty35t/100t, vc707, etc.)
|       ├── resources/  # Bootrom sources
├── sims/               # Simulation environments
│   ├── verilator/     # Verilator simulator
│   ├── vcs/           # VCS simulator
│   └── firesim/       # FireSim FPGA simulation
├── tools/              # Development tools
│   ├── chisel3/       # Chisel HDL
│   ├── firrtl/        # FIRRTL compiler
│   └── ...
├── vlsi/               # VLSI flow (Hammer)
└── software/           # Software tools (FireMarshal, etc.)
```

## 5. Configuration System

Chipyard uses a powerful configuration system based on Scala case classes and mixins:

- **Parameters**: Key-value stores passed through the design hierarchy
- **Configs**: Combine multiple parameter settings using mixins
- **Fragments**: Reusable configuration pieces that can be mixed together

This approach enables:
- Hierarchical parameterization
- Modular configuration reuse
- Compile-time hardware generation based on software parameters

## Key Takeaways

1. **Modular Architecture**: Chipyard's generator-based approach enables flexible, composable hardware design
2. **TileLink Ubiquity**: Understanding TileLink is essential for adding custom components
3. **Diplomacy Automation**: The Diplomacy framework handles complex parameter negotiation automatically
4. **Multi-Level Organization**: Tiles, buses, and memory hierarchy follow clear organizational patterns
5. **Integrated Ecosystem**: Generators, tools, simulators, and flows work together seamlessly

## Additional Resources

- [Chipyard Components Documentation](https://chipyard.readthedocs.io/en/latest/Chipyard-Basics/Chipyard-Components.html)
- [Rocket Chip Architecture](https://chipyard.readthedocs.io/en/latest/Generators/Rocket-Chip.html)
- [TileLink Specification 1.7](https://sifive.cdn.prismic.io/sifive%2F57f93ecf-2c42-46f7-9818-bcdd7d39400a_tilelink-spec-1.7.1.pdf)
- [Diplomacy Paper (Cook, Terpstra, Lee)](https://carrv.github.io/2017/papers/cook-diplomacy-carrv2017.pdf)
- [TileLink and Diplomacy Reference](https://chipyard.readthedocs.io/en/latest/TileLink-Diplomacy-Reference/index.html)

## Next Steps

Now that you understand the Chipyard system architecture, you're ready to explore:
- **[Part IV](/tutorials/IV-chipyard-custom-socs/)**: Building your custom Rocket Chip
- **[Part V](/tutorials/V-chipyard-system-customization/)**: System customization

