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

## 1. Understanding Default Core Configuration
Let's take a look at the system perspective
```mathematica
RocketTile
 ├── RocketCore
 ├── ICache
 ├── DCache
 ├── TileLink Master Node
 ├── Interrupt Interface
 └── Debug / RoCC Interfaces
```

### 1.1 RocketCore: Microarchitectural View
RocketCore implements a single-issue, in-order RISC-V core designed for simplicity, formal verifiability, and energy efficiency. It does not directly interface with the memory system or TileLink; that responsibility is delegated to the tile.

RocketCore consists of several tightly-coupled functional units:

| Unit | Function |
|------|----------|
| **Instruction Fetch (IF)** | Program counter generation, branch prediction (BTB, optional BHT/RAS), and instruction cache interface (via tile abstraction) |
| **Decode & Control** | RISC-V instruction decoding, CSR decode with privilege checks, exception and interrupt routing |
| **Execution Units** | Integer ALU, optional Mul/Div unit, optional FPU (single/double precision), optional RoCC interface for custom accelerators |
| **Register File** | 32 × XLEN integer registers, optional floating-point registers |
| **Memory Access** | Load/store pipeline stages with memory requests forwarded to the tile's D-cache |
| **CSR & Privilege Logic** | Machine/Supervisor/User modes, trap handling, and performance counters |

What RocketCore does not contain: cache ($I / $D), coherent logic, tilelink interfaces, debug transport modules. These are all handled at the RocketTile level.

You can find the implementation of Rocket core and its configurable options in the following file, let's focus on its customizable options:
```scala
#Path to file
#\generators\rocket-chip\src\main\scala\rocket\RocketCore.scala

case class RocketCoreParams(
  xLen: Int = 64,
  pgLevels: Int = 3, // sv39 default
  bootFreqHz: BigInt = 0,
  useVM: Boolean = true,
  useUser: Boolean = false,
  useSupervisor: Boolean = false,
  useHypervisor: Boolean = false,
  useDebug: Boolean = true,
  useAtomics: Boolean = true,
  useAtomicsOnlyForIO: Boolean = false,
  useCompressed: Boolean = true,
  useRVE: Boolean = false,
  useConditionalZero: Boolean = false,
  useZba: Boolean = false,
  useZbb: Boolean = false,
  useZbs: Boolean = false,
  nLocalInterrupts: Int = 0,
  useNMI: Boolean = false,
  nBreakpoints: Int = 1,
  useBPWatch: Boolean = false,
  mcontextWidth: Int = 0,
  scontextWidth: Int = 0,
  nPMPs: Int = 8,
  nPerfCounters: Int = 0,
  haveBasicCounters: Boolean = true,
  haveCFlush: Boolean = false,
  misaWritable: Boolean = true,
  nL2TLBEntries: Int = 0,
  nL2TLBWays: Int = 1,
  nPTECacheEntries: Int = 8,
  mtvecInit: Option[BigInt] = Some(BigInt(0)),
  mtvecWritable: Boolean = true,
  fastLoadWord: Boolean = true,
  fastLoadByte: Boolean = false,
  branchPredictionModeCSR: Boolean = false,
  clockGate: Boolean = false,
  mvendorid: Int = 0, // 0 means non-commercial implementation
  mimpid: Int = 0x20181004, // release date in BCD
  mulDiv: Option[MulDivParams] = Some(MulDivParams()),
  fpu: Option[FPUParams] = Some(FPUParams()),
  debugROB: Option[DebugROBParams] = None, // if size < 1, SW ROB, else HW ROB
  haveCease: Boolean = true, // non-standard CEASE instruction
  haveSimTimeout: Boolean = true, // add plusarg for simulation timeout
  vector: Option[RocketCoreVectorParams] = None
) extends CoreParams {
  val lgPauseCycles = 5
  val haveFSDirty = false
  val pmpGranularity: Int = if (useHypervisor) 4096 else 4
  val fetchWidth: Int = if (useCompressed) 2 else 1
  //  fetchWidth doubled, but coreInstBytes halved, for RVC:
  val decodeWidth: Int = fetchWidth / (if (useCompressed) 2 else 1)
  val retireWidth: Int = 1
  val instBits: Int = if (useCompressed) 16 else 32
  val lrscCycles: Int = 80 // worst case is 14 mispredicted branches + slop
  val traceHasWdata: Boolean = debugROB.isDefined // ooo wb, so no wdata in trace
  override val useVector = vector.isDefined
  override val vectorUseDCache = vector.map(_.useDCache).getOrElse(false)
  override def vLen = vector.map(_.vLen).getOrElse(0)
  override def eLen = vector.map(_.eLen).getOrElse(0)
  override def vfLen = vector.map(_.vfLen).getOrElse(0)
  override def vfh = vector.map(_.vfh).getOrElse(false)
  override def vExts = vector.map(_.vExts).getOrElse(Nil)
  override def vMemDataBits = vector.map(_.vMemDataBits).getOrElse(0)
  override val customIsaExt = Option.when(haveCease)("xrocket") // CEASE instruction
  override def minFLen: Int = fpu.map(_.minFLen).getOrElse(32)
  override def customCSRs(implicit p: Parameters) = new RocketCustomCSRs
}
```

`RocketCoreParams` is a case class that defines all configurable parameters for the Rocket RISC-V core. It extends `CoreParams` and serves as the primary configuration interface for customizing the core's ISA features, pipeline behavior, and optional components.
`RocketCoreParams` provides fine-grained control over the Rocket core's capabilities, enabling generation of cores ranging from minimal embedded processors (RV32I) to full-featured application processors (RV64IMAFDC with Hypervisor).

#### Parameter Categories Reference

**a. ISA Architecture & Privilege Modes**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `xLen` | Int | 64 | Register width (32 or 64 bits) |
| `pgLevels` | Int | 3 | Virtual memory page table depth: 2 (Sv32), 3 (Sv39), 4 (Sv48) |
| `useVM` | Boolean | true | Enable virtual memory support |
| `useUser` | Boolean | false | Enable User privilege mode |
| `useSupervisor` | Boolean | false | Enable Supervisor privilege mode |
| `useHypervisor` | Boolean | false | Enable Hypervisor extension (H-extension) |
| `useDebug` | Boolean | true | Enable debug module interface |

**b. Standard ISA Extensions**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `useAtomics` | Boolean | true | A extension: atomic memory operations |
| `useAtomicsOnlyForIO` | Boolean | false | Restrict atomics to I/O regions only |
| `useCompressed` | Boolean | true | C extension: 16-bit compressed instructions |
| `useRVE` | Boolean | false | RVE variant: embedded profile (16 registers) |

**c. Bitmanip Extensions (Zb*)**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `useZba` | Boolean | false | Zba: address generation instructions |
| `useZbb` | Boolean | false | Zbb: basic bit manipulation |
| `useZbs` | Boolean | false | Zbs: single-bit operations |
| `useConditionalZero` | Boolean | false | Conditional zero extension |

**d. Functional Units**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `mulDiv` | Option[MulDivParams] | Some(...) | Hardware multiplier/divider. Set to `None` for software emulation |
| `fpu` | Option[FPUParams] | Some(...) | Floating-point unit. Set to `None` to disable F/D extensions |
| `vector` | Option[RocketCoreVectorParams] | None | Vector extension support (V extension) |

**Note**: When `mulDiv` or `fpu` are set to `None`, those operations fall back to software emulation.

**e. Memory Management Unit (MMU)**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `nL2TLBEntries` | Int | 0 | Number of L2 TLB entries (0 = disabled) |
| `nL2TLBWays` | Int | 1 | L2 TLB associativity |
| `nPTECacheEntries` | Int | 8 | Page table entry cache size |
| `nPMPs` | Int | 8 | Number of physical memory protection (PMP) regions |
| `pmpGranularity` | Int | 4 or 4096 | PMP region granularity (4B default, 4KB with H-ext) |

**f. Branch Prediction**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `branchPredictionModeCSR` | Boolean | false | Expose CSR to control branch prediction mode |

**Note**: BTB (Branch Target Buffer) parameters are configured in `RocketTileParams`, not `RocketCoreParams`.

**g. Debug & Performance Monitoring**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `nBreakpoints` | Int | 1 | Number of hardware breakpoints |
| `useBPWatch` | Boolean | false | Enable breakpoint watchpoints |
| `nPerfCounters` | Int | 0 | Number of hardware performance counters |
| `haveBasicCounters` | Boolean | true | Enable basic cycle/instret counters |
| `debugROB` | Option[DebugROBParams] | None | Debug reorder buffer for out-of-order analysis |

**h. Control & Status Registers (CSRs)**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `mcontextWidth` | Int | 0 | Machine context register width |
| `scontextWidth` | Int | 0 | Supervisor context register width |
| `mtvecInit` | Option[BigInt] | Some(0) | Initial trap vector base address |
| `mtvecWritable` | Boolean | true | Allow runtime modification of `mtvec` |
| `misaWritable` | Boolean | true | Allow runtime modification of `misa` |
| `mvendorid` | Int | 0 | Vendor ID (0 = non-commercial) |
| `mimpid` | Int | 0x20181004 | Implementation ID (BCD-encoded date) |

**i. Interrupt Handling**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `nLocalInterrupts` | Int | 0 | Number of local interrupt sources |
| `useNMI` | Boolean | false | Enable non-maskable interrupts |

**j. Memory Interface Optimizations**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `fastLoadWord` | Boolean | true | Optimize word-aligned load operations |
| `fastLoadByte` | Boolean | false | Optimize byte loads (requires `fastLoadWord=true`) |
| `haveCFlush` | Boolean | false | Enable cache flush instruction |

**k. Power Management**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `clockGate` | Boolean | false | Enable clock gating for power savings |
| `haveCease` | Boolean | true | Enable CEASE instruction (non-standard, halts core) |

**l. Simulation & Boot**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `haveSimTimeout` | Boolean | true | Add plusarg for simulation timeout |
| `bootFreqHz` | BigInt | 0 | Boot frequency in Hz (informational) |


### 1.2 RocketTile: Tile-level Integration

**RocketTile** serves as the integration boundary between the core microarchitecture (RocketCore), the memory hierarchy, and the TileLink-based SoC fabric. It is the fundamental building block that the subsystem instantiates and replicates when adding multiple cores.

**RocketTileParams** is a case class that defines the complete configuration for a RocketTile, which is the top-level wrapper containing the Rocket core along with its caches, memory interfaces, and peripheral connections. It extends `InstantiableTileParams[RocketTile]`, making it responsible for instantiating the actual RocketTile LazyModule.

#### Key Components

**a. RocketCore Instance**
   - Configured via `RocketCoreParams`
   - Connected to private caches and CSRs
   - Does not directly access memory or TileLink

**b. Private L1 Caches**
   - **Instruction Cache (ICache)**: Read-only, configurable size and associativity
   - **Data Cache (DCache)**: Read/write with write-back policy
   - Cache parameters (sets, ways, TLB) defined at the tile level

**c. TileLink Master Node**
   - Issues memory transactions to the interconnect
   - Participates in cache coherence (L1–L2 protocol)
   - Provides optional uncached MMIO path for device I/O

**d. Debug & Trace Interface**
   - Debug Module Interface (DMI) for external debugger connection
   - Optional instruction trace for profiling and debugging

**e. Interrupt Handling**
   - **Local interrupts**: Core-specific interrupt sources
   - **External interrupts**: Routed from PLIC (Platform-Level Interrupt Controller) and CLINT (Core-Local Interruptor)

**f. RoCC Attachment Point**
   - Connection point for custom accelerators (Rocket Custom Coprocessor)
   - Custom accelerators attach to the tile, not directly to the core



You can find the implementation of Rocket core and its configurable options in the following file, let's focus on its customizable options:

```scala
// Path to file
// generators\rocket-chip\src\main\scala\tile\RocketTile.scala

case class RocketTileParams(
    core: RocketCoreParams = RocketCoreParams(),
    icache: Option[ICacheParams] = Some(ICacheParams()),
    dcache: Option[DCacheParams] = Some(DCacheParams()),
    btb: Option[BTBParams] = Some(BTBParams()),
    dataScratchpadBytes: Int = 0,
    tileId: Int = 0,
    beuAddr: Option[BigInt] = None,
    blockerCtrlAddr: Option[BigInt] = None,
    clockSinkParams: ClockSinkParameters = ClockSinkParameters(),
    boundaryBuffers: Option[RocketTileBoundaryBufferParams] = None
  ) extends InstantiableTileParams[RocketTile] {
  require(icache.isDefined)
  require(dcache.isDefined)
  val baseName = "rockettile"
  val uniqueName = s"${baseName}_$tileId"
  def instantiate(crossing: HierarchicalElementCrossingParamsLike, lookup: LookupByHartIdImpl)(implicit p: Parameters): RocketTile = {
    new RocketTile(this, crossing, lookup)
  }
}
```

#### RocketTileParams Configuration Reference

**a. Core Configuration**

| Parameter | Type | Default | Description |
|:----------|:-----|:--------|:------------|
| `core` | RocketCoreParams | RocketCoreParams() | Core pipeline configuration including ISA, extensions, and functional units. See [RocketCoreParams](#parameter-categories-reference) for details |

**b. Cache Hierarchy**

| Parameter | Type | Default | Description |
|:----------|:-----|:--------|:------------|
| `icache` | Option[ICacheParams] | Some(...) | Instruction cache configuration (required). Configures: sets, ways, TLB, block size, prefetcher, ECC |
| `dcache` | Option[DCacheParams] | Some(...) | Data cache configuration (required). Configures: sets, ways, MSHRs, TLB, scratchpad mode |
| `btb` | Option[BTBParams] | Some(...) | Branch Target Buffer configuration. Configures: entries, ways, RAS depth, BHT. Set to `None` to disable |

**c. Memory-Mapped Peripherals**

| Parameter | Type | Default | Description |
|:----------|:-----|:--------|:------------|
| `dataScratchpadBytes` | Int | 0 | Additional scratchpad memory size beyond cache scratchpad |
| `beuAddr` | Option[BigInt] | None | Bus Error Unit base address. Creates memory-mapped BEU for error reporting and interrupts |
| `blockerCtrlAddr` | Option[BigInt] | None | Bus blocker control register. Allows software to block/unblock tile transactions for power management |

**d. Tile Identity & Naming**

| Parameter | Type | Default | Description |
|:----------|:-----|:--------|:------------|
| `tileId` | Int | 0 | Unique tile identifier (must be unique). Used for hartid assignment and resource binding |
| `baseName` | String | "rockettile" | Base name for this tile type |
| `uniqueName` | String | (computed) | Unique instance name: `${baseName}_${tileId}` |

**e. Clock & Reset**

| Parameter | Type | Default | Description |
|:----------|:-----|:--------|:------------|
| `clockSinkParams` | ClockSinkParameters | ClockSinkParameters() | Clock domain sink configuration. Enables clock domain crossing and frequency specification |

**f. TileLink Buffering**

| Parameter | Type | Default | Description |
|:----------|:-----|:--------|:------------|
| `boundaryBuffers` | Option[RocketTileBoundaryBufferParams] | None | TileLink buffer insertion at tile boundaries for timing closure. `force=true` for full buffering, `force=false` for minimal |







### 1.3 Example RV32I Rocket Tile Configuration

```scala
class WithNRV32ICores(
  n: Int,
  crossing: RocketCrossingParams = RocketCrossingParams(),
) extends Config((site, here, up) => {
  case TilesLocated(InSubsystem) => {
    val prev = up(TilesLocated(InSubsystem), site)
    val idOffset = up(NumTiles)
    val med = RocketTileParams(
    core = RocketCoreParams(
    xLen = 32,
    pgLevels = 2,
    useVM = false,
    fpu = None,
    // mulDiv = Some(MulDivParams(mulUnroll = 8)),
    mulDiv = None,
    useAtomics = false,
    useCompressed = false
    ),
    btb = None,
    dcache = Some(DCacheParams(
    rowBits = site(SystemBusKey).beatBits,
    nSets = 32,
    nWays = 1,
    nTLBSets = 1,
    nTLBWays = 4,
    nMSHRs = 0,
    blockBytes = site(CacheBlockBytes))),
    icache = Some(ICacheParams(
    rowBits = site(SystemBusKey).beatBits,
    nSets = 32,
    nWays = 1,
    nTLBSets = 1,
    nTLBWays = 4,
    blockBytes = site(CacheBlockBytes))))
    List.tabulate(n)(i => RocketTileAttachParams(
    med.copy(tileId = i + idOffset),
    crossing
    )) ++ prev
  }
  case NumTiles => up(NumTiles) + n
})
```

This is my example configuration for RISC-V RV32I (32-bit integer-only cores) as a minimal embedded system. The `WithNRV32ICores` configuration class adds `n` RV32I cores to a Rocket Chip system.

**Core Configuration Breakdown:**

**Architecture Parameters:**
- `xLen = 32`: 32-bit architecture (RV32)
- `pgLevels = 2`: Sv32 virtual memory page table structure (not used since VM is disabled)
- `useVM = false`: Virtual memory disabled—core uses physical addressing only

**ISA Extensions (All Disabled):**
- `fpu = None`: No floating-point unit (no F/D extensions). Floating-point operations require software emulation
- `mulDiv = None`: No hardware multiply/divide unit. Multiplication and division must be handled in software
- `useAtomics = false`: No atomic instructions (no A extension)
- `useCompressed = false`: No compressed instructions (no C extension). All instructions are 32-bit

**Branch Prediction:**
- `btb = None`: No Branch Target Buffer. Uses simplest branch handling without prediction

**Cache Configuration:**

Both instruction cache (I$) and data cache (D$) share the same minimal configuration:
- **Size**: 32 sets × 1 way × 64 bytes (block size) = 2KB per cache
- **Associativity**: Direct-mapped (1-way). Simplest cache replacement policy
- **TLB**: 1 set × 4 ways configured but effectively unused since virtual memory is disabled

Cache sizes can be adjusted at the system level using configuration fragments like `WithL1ICacheSets` and `WithL1DCacheSets`

## 2. Example customization

### 2.1 Modifying core architecture and ISA extensions

The simplest modification to the RocketTile is changing the core configuration from RV32 to RV64 or viceversa. Moreover, depending on the applications, some extensions can be added.

**a. Core customizations**
My recommendation is whenever you want a new core configurations, let's create a new configuration in: 
```scala
// Path to file
// \generators\rocket-chip\src\main\scala\rocket\Configs.scala

class WithNRV32ICores(
  n: Int,
  crossing: RocketCrossingParams = RocketCrossingParams(),
) extends Config((site, here, up) => {
  case TilesLocated(InSubsystem) => {
    val prev = up(TilesLocated(InSubsystem), site)
    val idOffset = up(NumTiles)
    val med = RocketTileParams(
    core = RocketCoreParams(
    xLen = 32, // change to 64 to choose RV64
    pgLevels = 2,
    useVM = false,
    fpu = None,
    // mulDiv = Some(MulDivParams(mulUnroll = 8)),
    mulDiv = None,
    useAtomics = false,
    useCompressed = false
    ),
  ..........
```
- xLen: 32 or 64 to choose RV32 or RV64 respectively
- fpu: example `Some(FPUParams(minFLen = 16))` for floating-point unit (FPU) with half-precision support (16-bit floats)
- mulDiv: example `Some(MulDivParams(mulUnroll = 8))` for a hardware multiply/divide unit with an 8-bit unroll factor for the multiplier
- useAtomics: false or true
- useCompressed: false or true

**b. Cache size customization**

For the cache customization, you can set the value of `nSets` and `nWays` in the RocketTile configurations. However, I think this option affects the memory utilization and chip area on FPGA and ASIC, thus, this option should be change at the system level configuration for easy changing memory usage depending on the hardware that you have.




<!-- 
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
``` -->
<!-- 
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
``` -->

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
