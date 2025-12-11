---
layout: single
title: "Chipyard System Customization"
permalink: /tutorials/V-chipyard-system-customization/
date: 2025-01-05
author_profile: false
sidebar:
  nav: "tutorials"
toc: true
toc_label: "On This Page"
toc_icon: "list"
toc_sticky: true
classes: text-justify
---

## Introduction

Now that you have a working Chipyard SoC running on FPGA, the next step is customization. This section covers how to modify and extend your RISC-V system to meet specific design requirements.

System customization in Chipyard involves multiple layers:
- **Core Configuration**: ISA extensions, cache sizes, pipeline depth
- **Memory Subsystem**: Cache hierarchies, memory controllers, scratchpads
- **Peripherals**: Adding UART, SPI, GPIO, custom MMIO devices
- **Custom Accelerators**: RoCC, MMIO-based, or tightly-coupled accelerators
- **Interconnect**: Bus topologies, crossbars, NoC parameters

This tutorial series walks through practical customization scenarios, from simple parameter changes to adding entirely new hardware modules.

## 1. Customization Topics

![Chipyard System Customization Overview](/assets/images/tutorials/custom-rocket-chip.png)
{: .text-center}



### 1.1 [Rocket Core](/tutorials/V-chipyard-system-customization/core/)
**Customize RISC-V core parameters including ISA extensions, cache sizes**

Learn how to:
- Modify core microarchitecture parameters
- Configure ISA extensions (M, A, F, D, C)
- Tune cache sizes and associativity
- Select between in-order (Rocket) and out-of-order (BOOM) cores

Prerequisites: Basic RISC-V ISA knowledge, Scala familiarity

### 1.2 [Memory Subsystem](/tutorials/V-chipyard-system-customization/memory/)
**Optimize the memory hierarchy for your workload requirements.**

Learn how to:
- Configure L1/L2 cache parameters
- Add memory channels for higher bandwidth
- Choose between inclusive L2 and broadcast hub
- Use scratchpads or DRAM IP (FPGA only) for memories

Prerequisites: Understanding of cache organization, memory hierarchies

### 1.3 [Peripherals](/tutorials/V-chipyard-system-customization/peripherals/)
**Integrate standard and custom MMIO peripherals into your SoC.**

Learn how to:
- Integrate standard interfaces (UART, SPI, GPIO)
- Add memory-mapped I/O devices
- Create custom TileLink peripherals
- Add memory-mapped I/O devices with external I/O pins

Prerequisites: Memory-mapped I/O concepts, basic TileLink knowledge

### 1.4 [Custom Accelerators](/tutorials/V-chipyard-system-customization/accelerators/)
**Design and integrate domain-specific accelerators for performance gains.**

Learn how to:
- Create RoCC (Rocket Custom Coprocessor) accelerators
- Build MMIO-based accelerators
- Add DMA devices for high-bandwidth access
- Interface accelerators with software

Prerequisites: Strong Chisel knowledge, accelerator architecture basics

## 2. Before Getting Started

Before diving into customization, ensure you have:
- Completed [Getting Started with Chipyard](/tutorials/II-chipyard-getting-started/)
- Successfully built and deployed a base configuration
- Basic understanding of Scala/Chisel
- Familiarity with the Rocket Chip generator

<!-- ## Configuration Hierarchy

Chipyard uses a layered configuration system built on top of the Rocket Chip parameter system. Understanding this hierarchy is essential for effective customization:

```scala
// Example configuration hierarchy
class MyCustomConfig extends Config(
  new WithMyCustomFeature ++          // Your customization (highest priority)
  new WithBaseArty100TTweaks ++       // Platform-specific settings
  new WithNRV32ICores(1) ++           // Core configuration
  new chipyard.config.AbstractConfig  // Base Chipyard config
)
```

Configurations are composed using the `++` operator, where:
- Left side has **higher priority** (overrides settings from the right)
- Each `WithXXX` class is a configuration fragment
- The final result is a complete parameter set

## Common Customization Patterns

### 1. Changing Core Parameters
Modify ISA, number of cores, or microarchitectural features.

### 2. Memory Subsystem Tuning
Adjust cache sizes, associativity, memory timing parameters.

### 3. Adding Peripherals
Integrate standard peripherals (UART, SPI, I2C) or custom MMIO devices.

### 4. Custom Accelerators
Design and integrate domain-specific accelerators using RoCC or MMIO interfaces.

### 5. Boot Flow Customization
Modify the BootROM, boot device, or initialization sequence.

## Where to Make Changes

Key directories for customization:

```
chipyard/
├── generators/
│   ├── chipyard/src/main/scala/
│   │   ├── config/           # Configuration fragments
│   │   ├── DigitalTop.scala  # Top-level SoC composition
│   │   └── Periphery.scala   # Peripheral integration
│   └── rocket-chip/          # Core generator source
├── base/src/main/scala/      # Your custom FPGA platform configs
└── base/src/main/resources/  # BootROM and initialization code
``` -->

## Next Steps

Choose a customization path based on your project needs:

1. **[Rocket Core Configuration](/tutorials/V-chipyard-system-customization/core/)** - Start here if you need to modify ISA extensions, cache parameters, or core count
2. **[Memory Subsystem](/tutorials/V-chipyard-system-customization/memory/)** - Optimize memory hierarchy for bandwidth or latency requirements
3. **[Peripherals Integration](/tutorials/V-chipyard-system-customization/peripherals/)** - Add standard interfaces or custom MMIO devices
4. **[Custom Accelerators](/tutorials/V-chipyard-system-customization/accelerators/)** - Design domain-specific hardware for performance-critical tasks

Each tutorial builds on the base Chipyard system and can be followed independently or combined for complex designs.

## Resources

- [Chipyard Documentation - Customization](https://chipyard.readthedocs.io/en/latest/Customization/index.html)
- [Rocket Chip Parameter System](https://github.com/chipsalliance/rocket-chip)
- [Chisel/FIRRTL Documentation](https://www.chisel-lang.org/)
- [Diplomacy Reference](https://chipyard.readthedocs.io/en/latest/TileLink-Diplomacy-Reference/index.html)
