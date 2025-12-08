---
layout: single
title: "Custom Accelerators Integration"
permalink: /tutorials/V-chipyard-system-customization/accelerators/
date: 2025-01-05T04:00:00
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

Custom accelerators enable significant performance and energy improvements for domain-specific workloads. This tutorial covers how to design and integrate custom accelerators in Chipyard using both RoCC (Rocket Custom Coprocessor) and MMIO-based approaches.

## Prerequisites

- Completed [Getting Started with Chipyard](/tutorials/II-chipyard-getting-started/)
- Strong understanding of Chisel HDL
- Familiarity with TileLink and Diplomacy
- Basic knowledge of accelerator architectures

## Accelerator Integration Methods

Chipyard supports three primary methods for accelerator integration:

### 1. RoCC (Rocket Custom Coprocessor)

**Best for:**
- Tightly-coupled accelerators
- Low-latency instruction offload
- Direct register file access
- Sharing L1 cache with CPU

**Characteristics:**
- Custom instruction interface
- Integrated with CPU pipeline
- Access to L1 cache, PTW, FPU
- Single accelerator per tile

### 2. MMIO Peripherals

**Best for:**
- Loosely-coupled accelerators
- Multi-tile shared access
- Large data transfers
- Device-like interfaces

**Characteristics:**
- Memory-mapped control/status registers
- TileLink system bus attachment
- DMA capability (optional)
- Can be shared across cores

### 3. DMA Devices

**Best for:**
- High-bandwidth memory access
- Independent operation
- Streaming data processing

**Characteristics:**
- Direct memory access via FrontendBus
- Can bypass CPU entirely
- Interrupt-driven completion notification

## RoCC Accelerator Design

### Basic RoCC Structure

A RoCC accelerator consists of two classes:

```scala
class CustomAccelerator(opcodes: OpcodeSet)(implicit p: Parameters) 
    extends LazyRoCC(opcodes) {
  override lazy val module = new CustomAcceleratorModule(this)
}

class CustomAcceleratorModule(outer: CustomAccelerator)
    extends LazyRoCCModuleImp(outer) {
  // Accelerator implementation
  val cmd = Queue(io.cmd)
  
  // Command structure:
  // cmd.inst.opcode - the custom opcode
  // cmd.inst.rd     - destination register
  // cmd.inst.rs1    - source register 1
  // cmd.inst.rs2    - source register 2
  // cmd.inst.xd     - is rd being used?
  // cmd.inst.xs1    - is rs1 being used?
  // cmd.inst.xs2    - is rs2 being used?
  // cmd.rs1         - value of rs1
  // cmd.rs2         - value of rs2
  
  // Your accelerator logic here
  val result = cmd.rs1 + cmd.rs2  // Example: simple addition
  
  io.resp.valid := cmd.valid
  io.resp.bits.rd := cmd.bits.inst.rd
  io.resp.bits.data := result
  
  cmd.ready := io.resp.ready
  io.busy := cmd.valid
}
```

### RoCC Interface Signals

**Command Interface (`io.cmd`):**
- `valid`: Command available
- `ready`: Accelerator ready to accept
- `bits.inst`: Instruction fields
- `bits.rs1`, `bits.rs2`: Source operand values

**Response Interface (`io.resp`):**
- `valid`: Result available
- `ready`: CPU ready to accept
- `bits.rd`: Destination register
- `bits.data`: Result value

**Control Signals:**
- `io.busy`: Accelerator is processing
- `io.interrupt`: Request CPU interrupt

### Accessing Memory via L1 Cache

RoCC accelerators can access memory through the core's L1 cache using the `HellaCacheIO` interface:

```scala
class MemAccessAcceleratorModule(outer: CustomAccelerator)
    extends LazyRoCCModuleImp(outer) {
  
  val cmd = Queue(io.cmd)
  val addr = cmd.bits.rs1
  val data = cmd.bits.rs2
  
  // Memory request
  io.mem.req.valid := cmd.valid && !cmd.bits.inst.xd
  io.mem.req.bits.addr := addr
  io.mem.req.bits.tag := cmd.bits.inst.rd  // Track request with rd
  io.mem.req.bits.cmd := M_XRD  // Read command (M_XWR for write)
  io.mem.req.bits.size := log2Ceil(8).U  // 8-byte access
  
  // Memory response
  when(io.mem.resp.valid) {
    io.resp.valid := true.B
    io.resp.bits.rd := io.mem.resp.bits.tag
    io.resp.bits.data := io.mem.resp.bits.data
  }
  
  io.busy := cmd.valid || io.mem.req.valid
}
```

**Key Points:**
- Use `tag` field to track requests (typically 6 bits)
- Responses may arrive out-of-order for multiple requests
- Commands: `M_XRD` (read), `M_XWR` (write), `M_XA_*` (atomic)

### Accessing Memory via TileLink

For higher bandwidth, use the TileLink ports:

```scala
class TileLinkAccelerator(opcodes: OpcodeSet)(implicit p: Parameters)
    extends LazyRoCC(opcodes) {
  // tlNode connects to L1-L2 crossbar (high bandwidth)
  // atlNode connects to tile-local arbiter (shares with L1I backend)
  
  override lazy val module = new TileLinkAcceleratorModule(this)
}

class TileLinkAcceleratorModule(outer: TileLinkAccelerator)
    extends LazyRoCCModuleImp(outer) {
  
  // Use outer.tlNode for TileLink transactions
  // Example: Create Get request
  val (tl, edge) = outer.tlNode.out(0)
  
  val get = edge.Get(
    fromSource = 0.U,
    toAddress = addr,
    lgSize = log2Ceil(64).U
  )._2
  
  tl.a.valid := needsData
  tl.a.bits := get
  
  when(tl.d.valid) {
    // Process response
    val data = tl.d.bits.data
  }
}
```

### Adding RoCC to Configuration

```scala
class WithCustomAccelerator extends Config((site, here, up) => {
  case BuildRoCC => Seq((p: Parameters) => LazyModule(
    new CustomAccelerator(OpcodeSet.custom0 | OpcodeSet.custom1)(p)
  ))
})

class CustomAcceleratorConfig extends Config(
  new WithCustomAccelerator ++
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig
)
```

### Software Interface

Use RoCC macros in C code:

```c
#include "rocc.h"

// Execute custom instruction
// ROCC_INSTRUCTION(opcode, rd, rs1, rs2, funct)
ROCC_INSTRUCTION(0, result, addr, data, 0);

// Variants:
// ROCC_INSTRUCTION_S  - single source (rs1 only)
// ROCC_INSTRUCTION_D  - dual source (rs1, rs2)
// ROCC_INSTRUCTION_R  - with response (waits for rd)
```

## MMIO-Based Accelerators

### MMIO Accelerator Structure

```scala
class MMIOAccelerator(address: BigInt, beatBytes: Int)
    (implicit p: Parameters) extends LazyModule {
  
  val device = new SimpleDevice("accelerator", Seq("ucb,mmio-accel"))
  
  // TileLink node for MMIO
  val node = TLRegisterNode(
    address = Seq(AddressSet(address, 0xFFF)),  // 4KB address space
    device = device,
    beatBytes = beatBytes
  )
  
  lazy val module = new MMIOAcceleratorModule(this)
}

class MMIOAcceleratorModule(outer: MMIOAccelerator)
    extends LazyModuleImp(outer) {
  
  // Control/Status registers
  val control = RegInit(0.U(32.W))
  val status = RegInit(0.U(32.W))
  val dataIn = RegInit(0.U(64.W))
  val dataOut = RegInit(0.U(64.W))
  
  // Register map (using TLRegisterNode)
  outer.node.regmap(
    0x00 -> Seq(RegField(32, control)),      // Control register
    0x04 -> Seq(RegField(32, status)),       // Status register (read-only)
    0x08 -> Seq(RegField(64, dataIn)),       // Input data
    0x10 -> Seq(RegField(64, dataOut))       // Output data
  )
  
  // Accelerator logic
  val computing = RegInit(false.B)
  
  when(control(0)) {  // Start bit
    computing := true.B
    // Perform computation
    dataOut := dataIn * 2.U  // Example operation
  }
  
  when(computing) {
    status := 1.U  // Busy
    when(/* computation done */) {
      computing := false.B
      status := 2.U  // Done
    }
  }
}
```

### Integrating MMIO Accelerator

Create a config fragment:

```scala
class WithMMIOAccelerator(address: BigInt = 0x10000)
    extends Config((site, here, up) => {
  case BuildTop => (clock: Clock, reset: Bool, p: Parameters) => {
    val accel = LazyModule(new MMIOAccelerator(address, site(SystemBusKey).beatBytes)(p))
    
    // Connect to periphery bus
    accel.node := p(PeripheryBusKey).coupleTo("mmio_accel") { 
      TLFragmenter(p(SystemBusKey).beatBytes, p(SystemBusKey).blockBytes) := _
    }
    
    InModuleBody {
      // Optional: Add interrupts
      // site(ExtIntSourceKey) := accel.module.interrupt
    }
  }
})
```

### Software Access

```c
#include <stdint.h>

#define ACCEL_BASE 0x10000
#define ACCEL_CONTROL  (*(volatile uint32_t*)(ACCEL_BASE + 0x00))
#define ACCEL_STATUS   (*(volatile uint32_t*)(ACCEL_BASE + 0x04))
#define ACCEL_DATA_IN  (*(volatile uint64_t*)(ACCEL_BASE + 0x08))
#define ACCEL_DATA_OUT (*(volatile uint64_t*)(ACCEL_BASE + 0x10))

void use_accelerator(uint64_t input) {
    ACCEL_DATA_IN = input;
    ACCEL_CONTROL = 0x1;  // Start
    
    while (ACCEL_STATUS & 0x1);  // Wait for completion
    
    uint64_t result = ACCEL_DATA_OUT;
}
```

## DMA Accelerators

DMA devices attach to the FrontendBus for direct memory access:

```scala
class DMAAccelerator(implicit p: Parameters) extends LazyModule {
  // TileLink client node for DMA
  val dmaNode = TLClientNode(Seq(TLMasterPortParameters.v1(
    clients = Seq(TLMasterParameters.v1(
      name = "dma-accel",
      sourceId = IdRange(0, 4)
    ))
  )))
  
  lazy val module = new DMAAcceleratorModule(this)
}

class DMAAcceleratorModule(outer: DMAAccelerator) 
    extends LazyModuleImp(outer) {
  
  val (tl, edge) = outer.dmaNode.out(0)
  
  // Generate DMA transactions
  val readAddr = RegInit(0.U(32.W))
  val writeAddr = RegInit(0.U(32.W))
  
  // Issue read request
  val getReq = edge.Get(
    fromSource = 0.U,
    toAddress = readAddr,
    lgSize = log2Ceil(64).U
  )._2
  
  tl.a.valid := needsRead
  tl.a.bits := getReq
  
  // Handle read response and issue write
  when(tl.d.valid && tl.d.bits.opcode === TLMessages.AccessAckData) {
    val data = tl.d.bits.data
    // Process data, then write back
  }
}
```

## Choosing the Right Approach

| Criterion | RoCC | MMIO | DMA |
|-----------|------|------|-----|
| **Latency** | Lowest | Medium | Medium-High |
| **Bandwidth** | L1 limited | TileLink BW | Full memory BW |
| **Complexity** | Medium | Low-Medium | High |
| **Multi-core sharing** | No | Yes | Yes |
| **CPU coupling** | Tight | Loose | Very loose |
| **Best use case** | Register ops | Control-heavy | Streaming data |

## Advanced Topics

### Integrating Verilog/SystemVerilog Modules

```scala
class VerilogAcceleratorBlackBox extends BlackBox with HasBlackBoxResource {
  val io = IO(new Bundle {
    val clock = Input(Clock())
    val reset = Input(Bool())
    val in = Input(UInt(32.W))
    val out = Output(UInt(32.W))
    val valid = Input(Bool())
    val ready = Output(Bool())
  })
  
  addResource("/path/to/accel.v")
}
```

### Using High-Level Synthesis (HLS)

1. Generate Verilog from C/C++ using HLS tools
2. Wrap in BlackBox
3. Create Chisel wrapper with TileLink interface
4. Integrate as MMIO or DMA device

## Example: Complete Matrix Multiply Accelerator

See Gemmini for a production example:
- Uses RoCC for configuration
- DMA for data movement
- Systolic array for computation
- Scratchpad for temporary storage

Repository: [https://github.com/ucb-bar/gemmini](https://github.com/ucb-bar/gemmini)

## Testing and Verification

### Unit Tests

```scala
class AcceleratorSpec extends FlatSpec with ChiselScalatestTester {
  behavior of "CustomAccelerator"
  
  it should "compute correctly" in {
    test(new CustomAcceleratorModule(params)) { dut =>
      dut.io.cmd.valid.poke(true.B)
      dut.io.cmd.bits.rs1.poke(5.U)
      dut.io.cmd.bits.rs2.poke(3.U)
      dut.clock.step()
      dut.io.resp.bits.data.expect(8.U)
    }
  }
}
```

### Integration Tests

```bash
# Compile test program
cd generators/chipyard/src/main/resources/csrc
riscv64-unknown-elf-gcc -o accel_test accel_test.c

# Run in simulation
cd sims/verilator
make CONFIG=CustomAcceleratorConfig run-binary BINARY=accel_test
```

## Performance Optimization

### Pipeline Stages
- Balance pipeline depth with latency requirements
- Add registers to meet timing closure

### Memory Access Patterns
- Prefetch data to hide latency
- Use burst transactions for sequential access
- Implement double buffering

### Resource Sharing
- Share expensive units (dividers, multipliers)
- Use clock gating for power savings

## Common Issues and Solutions

### Elaboration Failures
- Check parameter consistency
- Verify TileLink address ranges don't overlap
- Ensure node connections are valid

### Simulation Hangs
- Check for TileLink protocol violations
- Verify ready/valid handshaking
- Add timeout logic

### Performance Below Expected
- Profile with waveforms
- Check for backpressure points
- Monitor TileLink utilization

## Additional Resources

- [RoCC Documentation](https://docs.google.com/document/d/1CH2ep4YcL_ojsa3BVHEW-uwcKh1FlFTjH_kg5v8bxVw/edit)
- [Chipyard RoCC Accelerators](https://chipyard.readthedocs.io/en/latest/Customization/RoCC-Accelerators.html)
- [MMIO Peripherals Guide](https://chipyard.readthedocs.io/en/latest/Customization/MMIO-Peripherals.html)
- [Gemmini Accelerator](https://github.com/ucb-bar/gemmini)
- [TileLink Specification](https://sifive.cdn.prismic.io/sifive%2F57f93ecf-2c42-46f7-9818-bcdd7d39400a_tilelink-spec-1.7.1.pdf)

## Next Steps

- Study existing accelerators (Gemmini, FFT, NVDLA)
- Implement a simple custom accelerator
- Profile and optimize your design
- Explore FireSim for full-system evaluation
