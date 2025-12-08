---
layout: single
title: "Building Custom SoCs with Chipyard"
permalink: /tutorials/IV-chipyard-custom-socs/
date: 2025-01-04
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

This tutorial walks through the practical workflow for generating RTL, synthesizing bitstreams, preparing SD cards, and deploying a custom bare-metal RISC-V SoC on FPGA prototypes using Chipyard.
While Chipyard natively supports several FPGA platforms (Arty-35T, Arty-100T, VC707, VCU118), this guide emphasizes Arty-100T and KR260, as these platforms were used extensively throughout my projects. Both platforms include additional architectural modifications developed for ASIC-oriented prototyping. For reference, the unmodified Chipyard FPGA configurations can be found in the `fpga/` directory of the Chipyard repository.

This tutorial focuses on the essential end-to-end process:
- RTL generation
- Bitstream generation
- SD-card preparation
- System bring-up on FPGA

A detailed description of the structure of the generated build directory and instructions on where to apply deeper customization are provided in a different page [here]().

## Prerequisites
### Hardware
Although full system simulation is possible in Chipyard, this tutorial concentrates solely on FPGA execution.
For simplicity, starting with a budget FPGA like arty100t is approachable for everyone. The lists of equipments is as follow:
- [Arty100t FPGA](https://digilent.com/shop/arty-a7-100t-artix-7-fpga-development-board/?srsltid=AfmBOoqGBwQYnGmrHKKIr8oUDSaQ6YhhXt9c9vn7jr58Gr7PrNe5fuNK)
- [PMOD SD card](https://digilent.com/shop/pmod-microsd-microsd-card-slot/?srsltid=AfmBOoroPlXlp7gaxir_A5DTaYPv2IFoF5bUb9InJiAeKWULyHnqsvBq)
- MicroSD card (≥4 GB recommended)

### Software
- Chipyard environment set up (see [Getting Started with Chipyard](/tutorials/chipyard-getting-started/))
- Vivado installed and accessible in your environment
- Serial communication software (e.g. microcom)
- Chipyard environment sourced prior to any build operation:

```bash
cd chipyard-1.13.0
source env.sh # Always source the environment before executing any make command
```

## 1. RISC-V Chipyard SoC Workflow

As I mentioned earlier in [Getting Started with Chipyard](/tutorials/chipyard-getting-started/), the `base/` directory is the primary workspace for development. Let's change the working directory to `base`:

```bash
# Change working directory to base before make anything
cd base/
```

### 1.1 Common Make Targets
The top-level `Makefile` lets you specify both:
- FPGA platform (SUB_PROJECT)
- System configuration (CONFIG)

Example usage:
```bash
# Default build
make <command> #default: SUB_PROJECT=arty100t CONFIG=BaseRocketKR260Config

# Explicit specification
make SUB_PROJECT=arty100t CONFIG=BaseRocketKR260Config <command>

# Default kr260
make SUB_PROJECT=kr260 <command> #default: CONFIG=BaseRocketKR260Config

```
**Common Make Commands:**

| Command | Description |
|---------|-------------|
| `make SUB_PROJECT=X CONFIG=Y verilog` | Generate RTL for configuration Y on FPGA X |
| `make SUB_PROJECT=X CONFIG=Y bitstream` | Generate bitstream for configuration Y on FPGA X |
| `make clean` | Clean build |

**Note**: Please make sure to `make clean` before remake any thing

**Available Configurations:**

| SUB_PROJECT | CONFIG |
|-------------|--------|
| arty100t | BaseRocketArty100TConfig |
| arty100t | AsicCompatibleRocketArty100TConfig |
| kr260 | BaseRocketKR260Config |
| kr260 | RAM32KBRocketKR260Config |
| kr260 | DualCoreRocketKR260Config |


### 1.2 RTL Generation
The following command generates Verilog RTL for the Arty-100T RocketChip configuration:

```bash
# Inside base/ directory
make SUB_PROJECT=arty100t verilog
```

This command will:
1. Elaborate the Chisel design
2. Generate Verilog RTL
3. Place output in `base/generated-src/`

If you want to explore the contents of the `generated-src/`, its description is [here](). Or else, go to the next step for generating bitstream on arty100t.


### 1.3 Bitstream Generation
The bitstream generation process takes a while to finish (depending on your system).
Ensure the Vivado environment is correctly sourced, then run:

```bash
# Inside base/ directory
make SUB_PROJECT=arty100t bitstream
```

### 1.4 Load bitstream

#### 1.4.1 Program FPGA

I added a tcl script to load the newly generated bitstream to FPGA. Simply run the make command to execute the script and load the bitstream to the FPGA.

```bash
# Inside base/ directory
make SUB_PROJECT=arty100t download_bitstream
```

Alternatively, you may use Vivado Hardware Manager and manually select the .bit file located at
```yaml
base/generated-src/chipyard.base.<X>.<X>Harness.<Y>/obj/<X>Harness.bit
```

#### 1.4.2 Verifying FPGA Bring-up 

Open another terminal and connect to the FPGA through UART communication.
On arty100t, the power cable is also the UART cable then there is no extra connection. But if using another FPGA, make sure you connect the UART communition between the FPGA and the SoC. Below is the my setup for arty100t and kr260.

**Arty100t setup:**
<img src="/assets/images/tutorials/arty100t-setup.jpeg" alt="Arty100t Connection" width="300">

**KR260 setup:**
<img src="/assets/images/tutorials/kr260-setup.png" alt="KR260 Connection" width="400">


```bash
# 1. Check exisisting COM port
ls /dev/ttyUSB*
# Example output
/dev/ttyUSB0  /dev/ttyUSB2

# 2. Fill in the ? depending on your system
# If you do not know which one is the correct one, just try them all
sudo microcom -s 115200 -p /dev/ttyUSB?
```

At this step, it is expected that there is an error in loading program from the SD card as we have not prepared it yet. However, the terminal should print the error as below. It indicates that the sytem can execute the bootROM and check for the existance of the SD card

```bash
# Example output
connected to /dev/ttyUSB0
Escape character: Ctrl-\
Type the escape character to get to the prompt.

INIT
CMD0
sd_cmd: timeout
ERROR
```


### 1.5 SD-card partitioning (skip if you have already done)

Chipyard’s BootROM assumes the payload resides at a specific sector offset.
Use gptfdisk to create a correctly formatted SiFive-style bare-metal partition.
Install gptfdisk by:
```bash
git clone https://github.com/tmagik/gptfdisk.git
cd gptfdisk/
# Build the tool
make -j `nproc`
```

After that, insert the SD card in to your computer, and identify the device

```bash
# Check whether the SD card is properly inserted
lsblk

# Example of output in terminal (it maybe different in your system)
...
sdd           8:48   0 698.6G  0 disk 
sde           8:64   1   3.6G  0 disk # My 4GB SD card
├─sde1        8:65   1   512M  0 part 
└─sde2        8:66   1   3.1G  0 part 
sr0          11:0    1  1024M  0 rom
...
```

What we need to use is the SD card is mounted as `/dev/sde`. Now let's partition the SD card.

Here are some useful commands of `gptfdisk`:

| Command | Description |
|---------|-------------|
| p | Print the partition table |
| n | Add a new partition |
| w | Write table to disk and exit |
| d | Delete a partition |
| q | Quit without saving changes |
| ? | List all commands |

```bash
# Inside gptfdisk/ directory
# Here is the example sequence of command that you need to run in your terminal
sudo ./gdisk /dev/sd?  # SD card on my machine is /dev/sd
d # delete existing partitions
n # create new partition
(Enter) # set default partition number
+1024 # sector offset
5202 # format the type of partition to 'Sifive bare-metal'
p # print to check the newly created partition
w # write the changes
y # comfirm the changes

# Eventually after partitioning, you want to check partition to be like this with p command
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048            3071   512.0 KiB   5202  SiFive bare-metal (...
```

Remember the partition sector is 2048 (this sector number is configured in the BootROM so that the correct sector of the SDcard is loaded to RAM). 

### 1.6 Load program to SD card

#### 1.6.1 Compiling the helloWorld payload
At this step, you will compile the `helloWorld` program and load it into the SDcard.
Again, everything is inside the `base/` directory. Navigate to the software `base/sw/helloWorld` directory and compile the `helloWorld` program.

```bash
# Inside base/sw/helloWorld directory
make bin # complile the software binary file
# The complied binary is stored in base/sw/helloWorld/build/
```

The software compilation (including the boot code in the bootROM) is configured to match with the hardware configuration(e.g. RV32 vs RV64, microarchitectural parameters, ABI). Any modifications to the core may require adjustments to compilation flags (CFLAGS) such as -march, -mabi. More details about customization is explained [here]().

#### 1.6.2 Writing the binary to the SD card
I created a helper script `uploa_SD.sh` to automate flashing the payload.
To use it, firstly, you have to plug the SD card to your computer and check its mounted device
```bash
# 1.Check the inserted SD card
lsblk

# Example output (it maybe different in your system)
sdc           8:32   0 931.5G  0 disk 
├─sdc1        8:33   0     1G  0 part /boot/efi
└─sdc2        8:34   0 930.5G  0 part /
sdd           8:48   0 698.6G  0 disk 
sde           8:64   1   3.6G  0 disk 
└─sde1        8:65   1   512K  0 part  # This is the partition we created before
sr0          11:0    1  1024M  0 rom  
nvme0n1     259:0    0   3.6T  0 disk 
# So in my machine, the partition to load the SD card is /dev/sde1

# 2. Load .bin to SD card
# Inside base/sw/helloWorld/
./upload_SD sde1

# [Note] You can manually load the .bin to SD card through this command
sudo dd if=<path_to_bin_file> of=/dev/<sd?1> conv=fsync bs=4096

```

After loading the SD card, insert it into the FPGA and press the reset button, the system will execute the bootROM again and try to load the content from SD card to RAM and execute the helloWorld program. A successful boot procedures:

```bash
# Example output after insterting the SD card and reset the system
INIT
CMD0
CMD8
ACMD41
CMD58
CMD16
CMD18
LOADING 0x00003000 B PAYLOAD
LOADING  
BOOT
Executing with Scratchpad RAM
Jumping to payload at address: 0x80000000

Hello world from core 0!!!
```

## 2. General Top-Level Customization

This section explains how the top-level settings work and what you should pay attention to when changing anything from the default setup. I’ll use the Arty100T configuration as the main example.

### 2.1 Custom FPGA and Configuration

Chipyard selects the FPGA platform and SoC configuration through the main Makefile in `base/Makefile`. For Arty100T, the default block looks like this:

```makefile
ifeq ($(SUB_PROJECT),arty100t)
	SBT_PROJECT       ?= base_chipyard_fpga
	MODEL             ?= Arty100THarness
	VLOG_MODEL        ?= Arty100THarness
	MODEL_PACKAGE     ?= chipyard.base.arty100t
	CONFIG            ?= BaseRocketArty100TConfig
	CONFIG_PACKAGE    ?= chipyard.base.arty100t
	GENERATOR_PACKAGE ?= chipyard
	TB                ?= none # unused
	TOP               ?= ChipTop
	BOARD             ?= arty_a7_100
	FPGA_BRAND        ?= xilinx
endif
```
Here's what these variables do:

| Field | Purpose |
|--------|-------------|
| **SBT project and model** | |
| SBT_PROJECT | The scala build tool project name that contains the source files |
| MODEL | The top-level Chisel class name for the design |
| VLOG_MODEL | The Verilog module name (usually same as MODEL) |
| **Package paths** | |
| MODEL_PACKAGE | Scala package path where the MODEL class is defined |
| CONFIG | The configuration class name that defines the SoC parameters |
| CONFIG_PACKAGE | Scala package path where CONFIG is defined |
| GENERATOR_PACKAGE | The package containing the generator infrastructure |
| **Build settings** | |
| TB | Testbench settings (unused for FPGA builds) |
| TOP | The actual top-level module after test harness |
| BOARD | Board identifiers used by Vivado scripts |
| FPGA_BRAND | FPGA Vendor (determines which tool scripts to use) |

<div style="text-align: left;" markdown="1">
These variables work together to tell the build system:
- Which Scala code to compile (chipyard.base.arty100t.Arty100THarness)
- Which configuration to use (chipyard.base.arty100t.BaseRocketArty100TConfig)
- Which FPGA board constraints and scripts to apply (only board & part selection)
</div>

### 2.2 The Configuration Class
Among above configurations, the component of greatest interest is the **CONFIG** class, as it defines the architectural and microarchitectural parameters of the SoC. The class of the `BaseRocketArty100TConfig` can be found at
```yaml
`base/src/main/scala/arty100t/Configs.scala`
```
```scala
// `base/src/main/scala/arty100t/Configs.scala`
class BaseRocketArty100TConfig extends Config(
  new WithBaseArty100TTweaks(isAsicCompatible=false) ++
  new freechips.rocketchip.rocket.WithNRV32ICores(1) ++
  new chipyard.config.AbstractConfig
)
```

This configuration is composed of three layers:

**1. `chipyard.config.AbstractConfig`**  
Base Chipyard configuration providing:
- Default SoC topology (buses, interconnect)
- Standard peripheral subsystems
- Default memory maps
- Basic device tree generation

**2. `freechips.rocketchip.rocket.WithNRV32ICores(1)`**  
Adds a single Rocket core with:
- RV32I ISA (32-bit integer-only instruction set)
- No M/A/F/D/C extensions
- Single-issue, in-order pipeline
- Basic MMU support

**3. `WithBaseArty100TTweaks(isAsicCompatible=false)`**  
Arty100T-specific platform configuration that overrides defaults from `AbstractConfig`. Common customization points include:
- Clock configurations
- Peripherals (UART, SPI, I2C, JTAG)
- Custom MMIO devices
- Memory configuration (cache, RAM, memory interface)
- Boot configuration

### Quick Summary
When customizing your SoC:

- Create another system configuration class similar to `BaseRocketArty100TConfig` in `base/src/main/scala/arty100t/Configs.scala`
- Change whatever you want in the newly created configuration class
- Make sure to run the `make` command with your configuration class `CONFIG=Y` or follow below instruction to change the default
- [Optional] Change the **CONFIG** in the top-level `Makefile` to your configuration class


## 3. Generated Source Description
After generating the bitstream, Chipyard places all elaborated and compiled artifacts into the generated-src/ directory. This folder contains many files, but only some of them are essential when verifying your SoC—either for FPGA bring-up or when preparing for chip fabrication.
This section highlights the important files and what they are used for.

### 3.1 Generated Files' Descriptions
Below is an overview of the major file groups found in generated-src/ and how they help during debugging or validation.

```
generated-src/
│
├── Memory Documentation
│   ├── *.memmap.json                      # Full system memory map
│   ├── 0x*.regmap.json                    # Register maps for each peripheral
│   └── *.dts                              # Device Tree (addresses, clocks, interrupts)
│
├── Hardware Design Files
│   ├── *.fir                              # FIRRTL representation
│   ├── *.anno.json                        # FIRRTL annotations (metadata)
│   ├── *.appended.anno.json               # Extra annotations added during transforms
│   └── *.graphml                          # Visual graph of the SoC topology
│
├── FPGA Build Files
│   ├── *.all.f                            # Full Verilog file list for synthesis
│   ├── *.top.f                            # Top-level hierarchy file list
│   ├── *.model.f                          # Behavioral model file list
│   ├── *.bb.f                             # Black-box IP list
│   ├── *.shell.vivado.tcl                 # Vivado integration script
│   ├── *.shell.xdc                        # FPGA constraint (pinout, timing)
│   ├── *.shell.sdc                        # Synopsys timing constraints
│   └── *.harnessSysPLLNode.vivado.tcl     # Clock/PLL generator script
│
├── Memory Configurations
│   ├── *.mems.conf                        # SRAM/compiler configuration for all memories
│   ├── *.top.mems.*                       # Memory config for ChipTop
│   └── *.model.mems.*                     # Memory config for simulation models
│
├── Build Metadata
│   ├── *.chisel.log                       # Chisel elaboration log
│   ├── *.firtool.log                      # FIRRTL-to-Verilog compilation log
│   ├── *.d                                # Dependency tracking
│   ├── *_module_hierarchy.json            # Full module hierarchy tree
│   └── *.json                             # Additional metadata
│
└── gen-collateral/
  └── *.v                                # All generated Verilog files go here
```

<!-- 1. **Memory Map & Register Documentation**
These files describe how the software will see the hardware:
- *.memmap.json — Complete memory map showing all peripheral base addresses and memory regions.
- 0x*.regmap.json — Register maps for each memory-mapped device (UART, SPI, GPIO, etc.).
- *.dts — Device Tree Source file used by the bootloader/OS; useful for verifying addresses, interrupts, and clock settings.

2. **Hardware Design Files**
These are generated during the Chisel → FIRRTL → Verilog flow:
- *.fir — FIRRTL intermediate representation of the design.
- *.anno.json — Annotations generated during FIRRTL transforms.
- *.appended.anno.json — Additional annotations added later in the pipeline.
- *.graphml — A graph view of the SoC topology (great for understanding interconnect structure).

3. **FPGA Synthesis Files**
These are the files Vivado needs in order to build the FPGA design:
- *.all.f — Full list of Verilog files used for synthesis.
- *.top.f — File list containing only the top-level hierarchy.
- *.model.f — Behavioral/simulation model file list.
- *.bb.f — List of black-box modules (e.g., PLLs, vendor IPs).
- *.shell.vivado.tcl — Vivado integration script for building the FPGA design.
- *.shell.xdc — Xilinx constraints (pinout, timing).
- *.shell.sdc — SDC timing constraints.
- *.harnessSysPLLNode.vivado.tcl — Script for generating clocking/PLL IPs.

4. **Memory Configuration**
These files describe the internal memories in the design:
- *.mems.conf — Memory compiler config for all SRAM/register files.
- *.top.mems.conf / *.top.mems.fir — Memory configuration for the ChipTop hierarchy.
- *.model.mems.conf / *.model.mems.fir — Memory configuration used for simulation models.

5. **Build Metadata**
These files are useful when debugging elaboration or synthesis issues:
- *.d — Dependency tracking file generated during building.
- *.chisel.log — Log file from elaboration; check for warnings.
- *.firtool.log — Log from FIRRTL → Verilog compilation via CIRCT.
- *.plusArgs — Simulation plus-args configuration.
- *_module_hierarchy.json — Full module hierarchy and instantiations.
- *.json — Miscellaneous metadata about the generated SoC. -->

<!-- 6. **Verilog Collateral Directory**
- `gen-collateral/` — Contains all generated Verilog files. This is the main directory Vivado uses during synthesis. -->

### 3.2 Key Verification Steps for FPGA Bring-up
1. **Device Tree (*.dts)**
- Confirm CPU ISA string is correct as intended (e.g rv32i)
- Ensure peripherals match physical connections
- Interrupt sources and routing are correct
2. **Memory Map Verification (*.memmap.json, *.dts)**
- Confirm peripheral base address (example UART @ 0x64000000)
- Verify RAM base @ 0x80000000 (DDR size 256 MB or scratchpad size)
- Check bootrom @ 0x10000
3. **Clock Configuration (*.dts, *.vivado.tcl)**
- Verify system clock
- Check PLL/clock generation settings
4. **Build Artifacts (*.all.f, *.d, *.xdc)**
- Use *.all.f as input to Vivado synthesis
- Check *.d for dependency tracking
- Check *.xdc for system default contraints or your contraints for custom designs
5. **Register Maps (0x64000000.*.regmap.json, etc.)**
- UART registers for baud-rate setup
- SPI registers (important for SD-card initialization)
- Any custom MMIO device registers if you added new hardware.

Based on these files, software developers can write bootROM code and drivers for peripherals and custom MMIOs.

## Next Steps

- Customize your SoC configuration in `generators/chipyard/src/main/scala/config/`
- Add custom accelerators or peripherals
- Optimize for your target FPGA platform
- Explore ASIC design flow with Hammer

## References

- [Chipyard Documentation](https://chipyard.readthedocs.io/)
- [RISC-V Proxy Kernel](https://github.com/riscv/riscv-pk)
- [OpenSBI Documentation](https://github.com/riscv/opensbi)
- [Buildroot User Manual](https://buildroot.org/docs.html)
