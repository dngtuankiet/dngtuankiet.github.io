---
layout: single
title: "Memory Subsystem Customization"
permalink: /tutorials/V-chipyard-system-customization/memory/
date: 2025-01-05T03:00:00
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

The memory subsystem is critical to SoC performance. This tutorial covers how to customize the memory hierarchy in Chipyard, including L1/L2 caches, memory controllers, scratchpads, and system buses. You'll learn to optimize the memory system for different workload requirements.

## Prerequisites

- Completed [Getting Started with Chipyard](/tutorials/II-chipyard-getting-started/)
- Understanding of cache organization (sets, ways, associativity)
- Familiarity with memory hierarchies
- Basic knowledge of TileLink protocol

## Memory Hierarchy Overview

Chipyard's memory hierarchy consists of several configurable layers:

```
CPU Tiles
  ├── L1 Instruction Cache (per tile)
  ├── L1 Data Cache (per tile)
  └── Page Table Walker (PTW)
       ↓
SystemBus (TileLink crossbar)
       ↓
L2 Cache or Broadcast Hub
       ↓
MemoryBus
       ↓
DRAM Controller (AXI4)
```

Each level can be customized through configuration fragments.

## L1 Cache Configuration

### Changing L1 Cache Size and Associativity

The L1 instruction and data caches are attached to each CPU tile. Default configuration uses 16 KiB, 4-way set-associative caches.

**Modifying L1I Cache:**

```scala
class CustomL1IConfig extends Config(
  new freechips.rocketchip.subsystem.WithL1ICacheSets(128) ++  // 128 sets
  new freechips.rocketchip.subsystem.WithL1ICacheWays(2) ++    // 2-way associative
  new chipyard.config.AbstractConfig
)
```

**Modifying L1D Cache:**

```scala
class CustomL1DConfig extends Config(
  new freechips.rocketchip.subsystem.WithL1DCacheSets(128) ++  // 128 sets
  new freechips.rocketchip.subsystem.WithL1DCacheWays(2) ++    // 2-way associative
  new chipyard.config.AbstractConfig
)
```

### Cache Size Calculation

Cache size = Sets × Ways × Block Size

For example:
- 128 sets × 2 ways × 64 bytes = 16 KiB

### Using Small/Medium Core Presets

Chipyard provides preset cache configurations:

```scala
// Small cores: 4 KiB direct-mapped L1I/L1D
class SmallCoreConfig extends Config(
  new freechips.rocketchip.subsystem.WithNSmallCores(1) ++
  new chipyard.config.AbstractConfig
)

// Medium cores: 16 KiB, 4-way L1I/L1D
class MediumCoreConfig extends Config(
  new freechips.rocketchip.subsystem.WithNMedCores(1) ++
  new chipyard.config.AbstractConfig
)
```

## L1 Data Scratchpad

You can configure the L1 data cache as a scratchpad memory instead of a cache. This provides deterministic access times but requires explicit memory management.

**Limitations:**
- Only supports single-core designs
- Cannot use external DRAM
- Fully removes L2 cache and memory bus

```scala
class ScratchpadOnlyRocketConfig extends Config(
  new chipyard.config.WithL2TLBs(0) ++
  new testchipip.soc.WithNoScratchpads ++              // Remove subsystem scratchpads
  new freechips.rocketchip.subsystem.WithNBanks(0) ++
  new freechips.rocketchip.subsystem.WithNoMemPort ++  // Remove offchip mem port
  new freechips.rocketchip.rocket.WithScratchpadsOnly ++ // Use L1 D$ as scratchpad
  new freechips.rocketchip.rocket.WithNHugeCores(1) ++
  new chipyard.config.AbstractConfig
)
```

## L2 Cache Configuration

### Inclusive Last-Level Cache

The default Chipyard configuration uses RocketChip's `InclusiveCache` generator for the L2. Default parameters:
- **Capacity**: 512 KiB
- **Associativity**: 8-way
- **Banks**: 1

**Customizing L2 Parameters:**

```scala
class CustomL2Config extends Config(
  new freechips.rocketchip.subsystem.WithInclusiveCache(
    nBanks = 2,        // Number of cache banks (must be power of 2)
    nWays = 16,        // Associativity (must be power of 2)
    capacityKB = 1024  // Total capacity in KiB
  ) ++
  new chipyard.config.AbstractConfig
)
```

The L2 cache maintains inclusion with L1 caches and provides cache coherence across multiple tiles.

### Using Broadcast Hub Instead

For resource-constrained designs, you can replace the L2 cache with a simpler broadcast hub:

```scala
class NoCacheConfig extends Config(
  // Simply omit WithInclusiveCache from your config
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig
)
```

**Bufferless Broadcast Hub** (further resource reduction):

```scala
class BufferlessBroadcastConfig extends Config(
  new freechips.rocketchip.subsystem.WithBufferlessBroadcastHub ++
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig
)
```

## System Bus Configuration

The SystemBus connects CPU tiles to L2 agents and MMIO peripherals. By default, it's a fully-connected TileLink crossbar.

For more complex interconnects, you can use Constellation to generate NoC-based topologies. See the [SoCs with NoC-based Interconnects](https://chipyard.readthedocs.io/en/latest/Customization/NoC-SoCs.html) documentation.

## Outer Memory System

### Multiple Memory Channels

Increase memory bandwidth by adding multiple DRAM channels:

```scala
class DualChannelMemConfig extends Config(
  new freechips.rocketchip.subsystem.WithNMemoryChannels(2) ++
  new chipyard.config.AbstractConfig
)
```

The number of channels must be a power of 2.

### Memory-Mapped Scratchpad (MBus)

Replace off-chip DRAM with an on-chip scratchpad attached to the memory bus:

```scala
class MbusScratchpadOnlyRocketConfig extends Config(
  new testchipip.soc.WithMbusScratchpad(banks=2, partitions=2) ++
  new freechips.rocketchip.subsystem.WithNoMemPort ++  // Remove offchip mem port
  new freechips.rocketchip.rocket.WithNHugeCores(1) ++
  new chipyard.config.AbstractConfig
)
```

This configuration:
- Adds 2 partitions with 2 banks each
- Removes the external memory port
- Provides fast, deterministic memory access

## Memory Timing Simulation

### RTL Simulation

In Verilator and VCS simulations, memory is modeled using `SimAXIMem`, which provides single-cycle SRAM behavior. This is fast but unrealistic.

### FireSim Memory Models

For realistic memory timing simulation, use FireSim, which can accurately model DDR3/DDR4 controllers with proper timing parameters. See the [FireSim Memory Models Documentation](https://docs.fires.im/en/latest/).

## Practical Configuration Examples

### High-Performance Configuration

```scala
class HighPerfConfig extends Config(
  new freechips.rocketchip.subsystem.WithL1ICacheSets(256) ++   // 64 KiB L1I
  new freechips.rocketchip.subsystem.WithL1ICacheWays(4) ++
  new freechips.rocketchip.subsystem.WithL1DCacheSets(256) ++   // 64 KiB L1D
  new freechips.rocketchip.subsystem.WithL1DCacheWays(4) ++
  new freechips.rocketchip.subsystem.WithInclusiveCache(
    nBanks = 4,
    nWays = 16,
    capacityKB = 2048  // 2 MiB L2
  ) ++
  new freechips.rocketchip.subsystem.WithNMemoryChannels(4) ++
  new freechips.rocketchip.subsystem.WithNBigCores(4) ++
  new chipyard.config.AbstractConfig
)
```

### Embedded/Resource-Constrained Configuration

```scala
class EmbeddedConfig extends Config(
  new freechips.rocketchip.subsystem.WithL1ICacheSets(32) ++    // 4 KiB L1I
  new freechips.rocketchip.subsystem.WithL1ICacheWays(2) ++
  new freechips.rocketchip.subsystem.WithL1DCacheSets(32) ++    // 4 KiB L1D
  new freechips.rocketchip.subsystem.WithL1DCacheWays(2) ++
  new freechips.rocketchip.subsystem.WithBufferlessBroadcastHub ++  // No L2
  new freechips.rocketchip.subsystem.WithNSmallCores(1) ++
  new chipyard.config.AbstractConfig
)
```

### Real-Time/Deterministic Configuration

```scala
class RealTimeConfig extends Config(
  new testchipip.soc.WithMbusScratchpad(banks=4, partitions=2) ++
  new freechips.rocketchip.subsystem.WithNoMemPort ++
  new chipyard.config.AbstractConfig
)
```

## Performance Analysis

### Cache Hit Rate Analysis

Monitor cache performance using:
- Performance counters in simulation
- HPM (Hardware Performance Monitor) events
- FireSim performance modeling

### Memory Bandwidth Profiling

Evaluate your memory system bandwidth:
1. Run memory-intensive benchmarks (STREAM, SPEC)
2. Monitor TileLink traffic
3. Adjust channels/banks/cache sizes based on bottlenecks

## Verification and Testing

After modifying the memory subsystem:

1. **Run RTL Simulation:**
   ```bash
   cd sims/verilator
   make CONFIG=YourMemConfig run-asm-tests
   ```

2. **Test with Benchmarks:**
   ```bash
   make CONFIG=YourMemConfig run-bmark-tests
   ```

3. **Check for Deadlocks/Livelocks:**
   Enable waveform dumping and inspect TileLink transactions

## Common Pitfalls

### Power-of-2 Constraints

- Cache banks, ways, and memory channels must be powers of 2
- Violating this will cause elaboration errors

### Scratchpad Limitations

- Scratchpad-only configs don't support multicore
- Cannot mix scratchpad with external DRAM in simple configs

### Address Mapping

- Ensure memory regions don't overlap
- Check Device Tree for proper address assignment

## Additional Resources

- [Chipyard Memory Hierarchy Documentation](https://chipyard.readthedocs.io/en/latest/Customization/Memory-Hierarchy.html)
- [InclusiveCache Parameters](https://github.com/chipsalliance/rocket-chip-inclusive-cache)
- [TileLink Specification](https://sifive.cdn.prismic.io/sifive%2F57f93ecf-2c42-46f7-9818-bcdd7d39400a_tilelink-spec-1.7.1.pdf)
- [FireSim Memory Models](https://docs.fires.im/en/latest/)

## Next Steps

- Explore [Custom Accelerators](/tutorials/V-chipyard-system-customization/accelerators/) for domain-specific compute
- Learn about [Peripherals Integration](/tutorials/V-chipyard-system-customization/peripherals/)
- Optimize your design using FireSim performance modeling
