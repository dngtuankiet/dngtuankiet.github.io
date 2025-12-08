---
layout: single
title: "Peripheral Modifications"
permalink: /tutorials/V-chipyard-system-customization/peripherals/
date: 2025-01-05T02:00:00
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

This tutorial guides you through adding custom memory-mapped I/O (MMIO) peripherals to your Chipyard SoC. You'll learn how to integrate new hardware modules, expose them through the system interconnect, and access them from software.

## Prerequisites

- Completed [Getting Started with Chipyard](/tutorials/II-chipyard-getting-started/)
- Basic understanding of [Chisel HDL](https://www.chisel-lang.org/)
- Familiarity with memory-mapped I/O concepts
- Understanding of TileLink protocol (helpful but not required)

## Types of Peripheral Integration

Chipyard supports several methods for adding peripherals:

1. **MMIO Peripherals**: Memory-mapped devices on the system bus
2. **TileLink Peripherals**: Native TileLink protocol devices
3. **AXI4 Peripherals**: Devices using AXI4 protocol (converted to TileLink)
4. **RoCC Accelerators**: Tightly-coupled accelerators (covered separately)

This tutorial focuses on MMIO peripherals, the most common integration method.

## Example 1: Simple GPIO Peripheral

Let's create a simple GPIO peripheral with configurable width.

### Step 1: Define the Hardware Module

Create `base/src/main/scala/devices/SimpleGPIO.scala`:

```scala
package chipyard.base.devices

import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import freechips.rocketchip.diplomacy._
import freechips.rocketchip.regmapper._
import freechips.rocketchip.tilelink._

// GPIO hardware implementation
class SimpleGPIOModule(val width: Int) extends Module {
  val io = IO(new Bundle {
    val gpio_out = Output(UInt(width.W))
    val gpio_in  = Input(UInt(width.W))
    val gpio_en  = Output(UInt(width.W))
  })
  
  // Registers
  val output_reg = RegInit(0.U(width.W))
  val enable_reg = RegInit(0.U(width.W))
  val input_reg  = RegNext(io.gpio_in)
  
  // Output assignments
  io.gpio_out := output_reg
  io.gpio_en  := enable_reg
  
  // Register interface (exposed via regmap)
  def regmap = Seq(
    0x00 -> Seq(RegField(width, output_reg)),  // Write to GPIO outputs
    0x04 -> Seq(RegField.r(width, input_reg)), // Read GPIO inputs
    0x08 -> Seq(RegField(width, enable_reg))   // Output enable control
  )
}

// TileLink wrapper
class SimpleGPIO(address: BigInt, width: Int = 32)(implicit p: Parameters)
  extends TLRegisterRouter(
    address, "gpio", Seq("chipyard,gpio"),
    beatBytes = 4)(
      new TLRegBundle((), _) {
        val gpio_out = Output(UInt(width.W))
        val gpio_in  = Input(UInt(width.W))
        val gpio_en  = Output(UInt(width.W))
      })(
      new TLRegModule((), _, _) {
        val impl = Module(new SimpleGPIOModule(width))
        
        // Connect bundle to implementation
        bundle.gpio_out := impl.io.gpio_out
        impl.io.gpio_in := bundle.gpio_in
        bundle.gpio_en  := impl.io.gpio_en
        
        // Register map
        regmap(impl.regmap: _*)
      })
```

### Step 2: Add Peripheral to the SoC

Modify `base/src/main/scala/DigitalTop.scala` (or your custom top file):

```scala
// Add trait for GPIO peripheral
trait HasPeripheryGPIO { this: BaseSubsystem =>
  val gpioNode = p(GPIOKey) match {
    case Some(params) => {
      val gpio = LazyModule(new SimpleGPIO(
        address = params.address,
        width = params.width
      ))
      // Connect to peripheral bus
      pbus.coupleTo("gpio") { gpio.node := TLFragmenter(4, pbus.blockBytes) := _ }
      Some(gpio)
    }
    case None => None
  }
}

// Implementation trait
trait HasPeripheryGPIOModuleImp extends LazyModuleImp {
  val outer: HasPeripheryGPIO
  
  val gpio = outer.gpioNode.map { gpio =>
    val io = IO(new Bundle {
      val gpio_out = Output(UInt(gpio.width.W))
      val gpio_in  = Input(UInt(gpio.width.W))
      val gpio_en  = Output(UInt(gpio.width.W))
    })
    io.gpio_out := gpio.module.bundle.gpio_out
    gpio.module.bundle.gpio_in := io.gpio_in
    io.gpio_en := gpio.module.bundle.gpio_en
    io
  }
}
```

### Step 3: Create Configuration Fragment

In `base/src/main/scala/arty100t/Configs.scala`:

```scala
import chipyard.base.devices._

// Configuration case class
case class GPIOParams(
  address: BigInt,
  width: Int = 32
)

// Configuration key
case object GPIOKey extends Field[Option[GPIOParams]](None)

// Configuration fragment
class WithGPIO(address: BigInt = 0x64003000, width: Int = 32) extends Config((site, here, up) => {
  case GPIOKey => Some(GPIOParams(address = address, width = width))
})

// Complete configuration
class GPIORocketConfig extends Config(
  new WithGPIO(address = 0x64003000, width = 16) ++
  new WithBaseArty100TTweaks(isAsicCompatible = false) ++
  new freechips.rocketchip.rocket.WithNRV32ICores(1) ++
  new chipyard.config.AbstractConfig
)
```

### Step 4: Connect Peripheral Pins

In `base/base-fpga-shells/src/main/scala/arty100t/Arty100TGPIO.scala`:

```scala
// Add GPIO overlay
class WithArty100TGPIO extends Config((site, here, up) => {
  case PeripheryGPIOKey => up(PeripheryGPIOKey) :+ GPIOParams(
    address = 0x64003000,
    width = 8
  )
})
```

Connect to FPGA pins in the top-level harness.

### Step 5: Build and Test

```bash
cd base/
make SUB_PROJECT=arty100t CONFIG=GPIORocketConfig verilog
make SUB_PROJECT=arty100t CONFIG=GPIORocketConfig bitstream
```

## Software Access

### C Driver Example

```c
#define GPIO_BASE_ADDR 0x64003000
#define GPIO_OUTPUT  (GPIO_BASE_ADDR + 0x00)
#define GPIO_INPUT   (GPIO_BASE_ADDR + 0x04)
#define GPIO_ENABLE  (GPIO_BASE_ADDR + 0x08)

// Write helper
static inline void write_reg(uint32_t addr, uint32_t value) {
    *((volatile uint32_t *)addr) = value;
}

// Read helper
static inline uint32_t read_reg(uint32_t addr) {
    return *((volatile uint32_t *)addr);
}

// GPIO driver functions
void gpio_set_output(uint32_t value) {
    write_reg(GPIO_OUTPUT, value);
}

void gpio_set_enable(uint32_t mask) {
    write_reg(GPIO_ENABLE, mask);
}

uint32_t gpio_read_input(void) {
    return read_reg(GPIO_INPUT);
}

// Example usage
int main(void) {
    // Configure pins 0-3 as outputs
    gpio_set_enable(0x0F);
    
    // Set output values
    gpio_set_output(0x05);  // Binary: 0101
    
    // Read input pins
    uint32_t inputs = gpio_read_input();
    
    return 0;
}
```

## Example 2: Counter Peripheral

A simple counter that can be started, stopped, and reset:

```scala
class CounterModule extends Module {
  val io = IO(new Bundle {
    val count = Output(UInt(32.W))
  })
  
  val count_reg = RegInit(0.U(32.W))
  val enable = RegInit(false.B)
  
  when(enable) {
    count_reg := count_reg + 1.U
  }
  
  io.count := count_reg
  
  def regmap = Seq(
    0x00 -> Seq(RegField.r(32, count_reg)),        // Read counter value
    0x04 -> Seq(RegField(1, enable)),              // Enable/disable counting
    0x08 -> Seq(RegField.w(1, RegWriteFn((valid, data) => {
      when(valid && data(0)) { count_reg := 0.U }  // Reset counter
      true.B
    })))
  )
}
```

## Debugging Tips

### 1. Verify Address Mapping

Check the generated memory map:
```bash
cat generated-src/*/chipyard.base.*.memmap.json | jq
```

Look for your peripheral's base address.

### 2. Check Device Tree

```bash
cat generated-src/*/chipyard.base.*.dts | grep -A 10 "gpio"
```

### 3. Software Testing

Add debug prints in your driver:
```c
printf("GPIO base address: 0x%x\n", GPIO_BASE_ADDR);
printf("Writing 0x%x to output register\n", value);
```

### 4. Simulation

Before FPGA testing, simulate in Verilator:
```bash
cd sims/verilator
make CONFIG=GPIORocketConfig run-binary BINARY=gpio_test.elf
```

## Common Issues

### Issue 1: Address Conflicts
**Symptom**: System hangs or incorrect peripheral behavior
**Solution**: Check memory map for overlapping addresses

### Issue 2: TileLink Protocol Violations
**Symptom**: Elaboration errors or simulation hangs
**Solution**: Ensure register widths match bus width, use proper RegField types

### Issue 3: Missing Device Tree Entry
**Symptom**: Software can't find peripheral
**Solution**: Verify TLRegisterRouter parameters include device tree strings

## Best Practices

1. **Start simple**: Test basic functionality before adding complexity
2. **Use RegField helpers**: Leverage built-in register abstractions
3. **Document registers**: Add clear comments for register offsets and fields
4. **Test in simulation**: Verify behavior before synthesizing
5. **Follow naming conventions**: Use consistent naming for addresses, widths

## Next Steps

- Add interrupt support to your peripheral
- Create DMA-capable peripherals
- Explore AXI4 to TileLink conversion
- Learn about RoCC accelerator integration

## Resources

- [TileLink Specification](https://chipyard.readthedocs.io/en/latest/TileLink-Diplomacy-Reference/)
- [Chipyard MMIO Peripherals Guide](https://chipyard.readthedocs.io/en/latest/Customization/MMIO-Peripherals.html)
- [Chisel Register Mapping](https://www.chisel-lang.org/api/latest/)
- [Rocket Chip Diplomacy](https://github.com/chipsalliance/rocket-chip)
