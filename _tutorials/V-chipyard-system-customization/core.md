---
layout: single
title: "Customizing Core Parameters"
permalink: /tutorials/V-chipyard-system-customization/core/
date: 2025-01-05T01:00:00
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

This tutorial demonstrates how to customize RISC-V core parameters in Chipyard, including ISA extensions, cache configurations, and pipeline parameters. We'll walk through practical examples of modifying the Rocket core configuration to suit different application requirements.

## Prerequisites

- Completed [Getting Started with Chipyard](/tutorials/II-chipyard-getting-started/)
- Basic understanding of RISC-V ISA
- Familiarity with Scala syntax
- Working Chipyard environment

## Understanding Core Configuration

Chipyard uses the Rocket Chip generator's configuration system. Core parameters are set through configuration fragments (classes starting with `With...`) that override default settings.

### Common Configuration Fragments

Here are the most frequently used core configuration fragments:

```scala
// Number and type of cores
new WithNRV32ICores(1)        // 1 RV32I core
new WithNRV64GCCores(2)       // 2 RV64GC cores
new WithNBigCores(4)          // 4 big (out-of-order) cores
new WithNSmallCores(8)        // 8 small (in-order) cores

// ISA extensions
new WithRV32               // 32-bit base
new WithRV64               // 64-bit base (default)
new WithoutFPU             // Remove floating-point
new WithRVC                // Compressed instructions
new WithRVA                // Atomic instructions
new WithRVV                // Vector extension
new WithCustom             // Custom extension

// Cache sizes (in KB)
new WithL1ICacheSets(64)   // I-cache sets
new WithL1DCacheSets(64)   // D-cache sets
new WithL1ICacheWays(4)    // I-cache ways
new WithL1DCacheWays(4)    // D-cache ways
```

## Example 1: Creating a Custom RV32I Configuration

Let's create a custom configuration for a simple RV32I system with modified cache sizes.

### Step 1: Define the Configuration

Create or edit `base/src/main/scala/arty100t/Configs.scala`:

```scala
class CustomRV32IConfig extends Config(
  // Custom cache configuration
  new freechips.rocketchip.subsystem.WithL1ICacheSets(32) ++  // 32 sets = 4KB I-cache
  new freechips.rocketchip.subsystem.WithL1DCacheSets(32) ++  // 32 sets = 4KB D-cache
  new freechips.rocketchip.subsystem.WithL1ICacheWays(2) ++   // 2-way set associative
  new freechips.rocketchip.subsystem.WithL1DCacheWays(2) ++   // 2-way set associative
  
  // RV32I core with specific features
  new freechips.rocketchip.rocket.WithNRV32ICores(1) ++
  
  // Platform-specific tweaks
  new WithBaseArty100TTweaks(isAsicCompatible=false) ++
  new chipyard.config.AbstractConfig
)
```

### Step 2: Build the Configuration

```bash
cd base/
make SUB_PROJECT=arty100t CONFIG=CustomRV32IConfig verilog
```

### Step 3: Verify the Configuration

Check the generated device tree to confirm ISA and cache sizes:

```bash
cat generated-src/*/chipyard.base.arty100t.*.dts | grep -A 5 "cpu@0"
```

Expected output should show:
```
cpu@0 {
    device_type = "cpu";
    reg = <0x0>;
    status = "okay";
    compatible = "riscv";
    riscv,isa = "rv32i";  // Confirm RV32I
    ...
}
```

## Example 2: Dual-Core RV64GC Configuration

Create a dual-core system with full ISA support (GC extensions):

```scala
class DualCoreRV64GCConfig extends Config(
  // Two RV64GC cores (G = IMAFD, C = compressed)
  new freechips.rocketchip.subsystem.WithNBigCores(2) ++
  
  // Larger caches for better performance
  new freechips.rocketchip.subsystem.WithL1ICacheSets(64) ++  // 8KB I-cache
  new freechips.rocketchip.subsystem.WithL1DCacheSets(64) ++  // 8KB D-cache
  new freechips.rocketchip.subsystem.WithL1ICacheWays(4) ++   // 4-way
  new freechips.rocketchip.subsystem.WithL1DCacheWays(4) ++   // 4-way
  
  // Platform configuration
  new WithBaseArty100TTweaks(isAsicCompatible=false) ++
  new chipyard.config.AbstractConfig
)
```

### Multi-Core Considerations

When adding multiple cores:
1. **Memory bandwidth**: Ensure your memory subsystem can handle concurrent accesses
2. **Cache coherence**: Rocket cores include built-in cache coherence (MESI protocol)
3. **Interrupt routing**: Configure CLINT and PLIC for multiple harts
4. **Boot sequence**: All harts start at reset; use `mhartid` to differentiate

## Example 3: Embedded RV32IMC Configuration

For resource-constrained designs, create a minimal configuration:

```scala
class TinyRV32IMCConfig extends Config(
  // Minimal cache sizes
  new freechips.rocketchip.subsystem.WithL1ICacheSets(16) ++  // 2KB I-cache
  new freechips.rocketchip.subsystem.WithL1DCacheSets(16) ++  // 2KB D-cache
  new freechips.rocketchip.subsystem.WithL1ICacheWays(1) ++   // Direct-mapped
  new freechips.rocketchip.subsystem.WithL1DCacheWays(1) ++   // Direct-mapped
  
  // RV32IMC core (Integer + Multiply + Compressed)
  new freechips.rocketchip.rocket.WithNRV32IMCCores(1) ++
  
  // Disable FPU
  new freechips.rocketchip.subsystem.WithoutFPU ++
  
  // Platform configuration
  new WithBaseArty100TTweaks(isAsicCompatible=false) ++
  new chipyard.config.AbstractConfig
)
```

## Understanding Cache Parameters

### Cache Size Calculation

```
Cache Size = Sets × Ways × Block Size
```

For Rocket cores:
- **Block Size**: Fixed at 64 bytes (cache line size)
- **Sets**: Configurable (power of 2)
- **Ways**: Configurable (typically 1, 2, 4, or 8)

Example calculations:
```
16 sets × 2 ways × 64 bytes = 2 KB
32 sets × 4 ways × 64 bytes = 8 KB
64 sets × 8 ways × 64 bytes = 32 KB
```

### Cache Hierarchy

Rocket cores have separate L1 I-cache and D-cache:
- **L1 I-cache**: Instruction cache (read-only)
- **L1 D-cache**: Data cache (read/write, write-back policy)
- **L2 cache**: Optional, shared across cores

## Verifying Your Configuration

### 1. Check Generated Files

```bash
# Memory map
cat generated-src/*/chipyard.base.*.memmap.json | jq '.[] | select(.name | contains("cache"))'

# Device tree
cat generated-src/*/chipyard.base.*.dts
```

### 2. Resource Utilization

After synthesis, check FPGA resource usage:
```bash
cat generated-src/*/obj/*_utilization.rpt
```

### 3. Software Testing

Compile and run test programs:
```bash
cd base/sw/custom_test/
make bin
# Load to SD card and test
```

## Common Configuration Issues

### Issue 1: Out of BRAM
**Symptom**: Synthesis fails with "insufficient BRAM"
**Solution**: Reduce cache sizes or number of cores

### Issue 2: ISA Mismatch
**Symptom**: Software fails to run, illegal instruction exceptions
**Solution**: Ensure compiler flags match ISA (e.g., `-march=rv32i` for RV32I)

### Issue 3: Performance Degradation
**Symptom**: System slower than expected
**Solution**: Increase cache sizes, check for cache thrashing

## Performance Tuning Guidelines

1. **Start small**: Begin with minimal caches, increase as needed
2. **Profile first**: Use performance counters to identify bottlenecks
3. **Balance resources**: Don't over-provision caches on constrained FPGAs
4. **Consider workload**: Size caches based on working set size

## Next Steps

- [Child Item 2: Adding Custom Peripherals](/tutorials/V-chipyard-system-customization/child2/)
- [Understanding Chipyard System Architecture](/tutorials/III-understand-chipyard-system/)
- Explore advanced cache configurations (prefetchers, replacement policies)

## Resources

- [Rocket Chip Configuration Guide](https://github.com/chipsalliance/rocket-chip)
- [RISC-V ISA Specifications](https://riscv.org/technical/specifications/)
- [Chipyard Cache Documentation](https://chipyard.readthedocs.io/)
