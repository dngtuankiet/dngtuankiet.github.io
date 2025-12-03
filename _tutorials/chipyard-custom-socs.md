---
layout: single
title: "Building Custom SoCs with Chipyard"
permalink: /tutorials/chipyard-custom-socs/
author_profile: false
sidebar:
  nav: "tutorials"
toc: true
toc_label: "On This Page"
toc_icon: "list"
toc_sticky: true
---

## Introduction

This tutorial walks through the process of generating RTL for custom SoCs in Chipyard and creating bootable SD cards for FPGA prototyping.

## Prerequisites

- Chipyard environment set up (see [Getting Started with Chipyard](/tutorials/chipyard-getting-started/))
- Source the Chipyard environment:
  ```bash
  cd chipyard
  source env.sh
  ```

## RTL Generation

### Understanding the Build System

Chipyard uses a Makefile-based build system to generate RTL from high-level Chisel descriptions. The generated RTL can be used for simulation, FPGA prototyping, or ASIC implementation.

### Basic RTL Generation

To generate RTL for a default RocketChip configuration:

```bash
cd sims/vcs  # or sims/verilator
make CONFIG=RocketConfig
```

This command will:
1. Elaborate the Chisel design
2. Generate Verilog RTL
3. Place output in `generated-src/`

### Custom Configuration RTL Generation

For custom SoC configurations:

```bash
# Generate RTL for a custom configuration
make CONFIG=MyCustomConfig

# Specify the project (if not using chipyard)
make PROJECT=myproject CONFIG=MyCustomConfig

# Clean before rebuilding
make CONFIG=MyCustomConfig clean
make CONFIG=MyCustomConfig
```

### FPGA-Specific RTL Generation

For FPGA prototyping (e.g., VCU118, Arty):

```bash
cd fpga
make SUB_PROJECT=vcu118 CONFIG=RocketVCU118Config bitstream
```

### Understanding Build Outputs

After RTL generation, check these directories:
- `generated-src/` - Generated Verilog files
- `generated-src/CONFIG_NAME.v` - Top-level Verilog module
- `generated-src/CONFIG_NAME.*.vh` - Include files and parameters

### Common Make Targets

| Command | Description |
|---------|-------------|
| `make CONFIG=X` | Generate RTL for configuration X |
| `make CONFIG=X clean` | Clean build artifacts |
| `make CONFIG=X debug` | Build with debug symbols |
| `make CONFIG=X run-binary BINARY=test.elf` | Run simulation with binary |

## Creating a Boot SD Card

### Overview

For FPGA prototyping, you need to create a bootable SD card containing:
- First Stage Boot Loader (FSBL)
- Berkeley Boot Loader (BBL) / OpenSBI
- Linux kernel
- Root filesystem

### Prerequisites

- FPGA bitstream generated
- Linux kernel compiled for RISC-V
- Root filesystem image (Buildroot or Debian)
- SD card (8GB+ recommended)

### Step 1: Partition the SD Card

```bash
# Identify SD card device (e.g., /dev/sdX)
lsblk

# Create partition table
sudo fdisk /dev/sdX
# Commands in fdisk:
# o - create new DOS partition table
# n - new partition (boot partition, ~200MB, type: FAT32)
# n - new partition (root partition, remaining space, type: Linux)
# w - write changes

# Format partitions
sudo mkfs.vfat -F 32 /dev/sdX1  # Boot partition
sudo mkfs.ext4 /dev/sdX2         # Root partition
```

### Step 2: Copy Boot Files

```bash
# Mount boot partition
sudo mount /dev/sdX1 /mnt/boot

# Copy bootloader and kernel
sudo cp fpga/generated-src/PROJECT.CONFIG/obj/fsbl.elf /mnt/boot/
sudo cp fpga/generated-src/PROJECT.CONFIG/obj/bbl.bin /mnt/boot/
sudo cp linux-build/Image /mnt/boot/

# Copy device tree (if applicable)
sudo cp fpga/generated-src/PROJECT.CONFIG/obj/system.dtb /mnt/boot/

# Unmount
sudo umount /mnt/boot
```

### Step 3: Install Root Filesystem

```bash
# Mount root partition
sudo mount /dev/sdX2 /mnt/root

# Extract root filesystem
sudo tar -xzf rootfs.tar.gz -C /mnt/root

# Or copy Buildroot output
sudo cp -a buildroot/output/target/* /mnt/root/

# Set permissions
sudo chown -R root:root /mnt/root

# Unmount
sudo umount /mnt/root
```

### Step 4: Configure Boot Settings

Create a boot configuration file on the boot partition:

```bash
sudo mount /dev/sdX1 /mnt/boot

# Create boot.scr or similar configuration
cat << EOF | sudo tee /mnt/boot/boot.cmd
setenv bootargs "console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait"
load mmc 0:1 0x80000000 Image
load mmc 0:1 0x82000000 system.dtb
booti 0x80000000 - 0x82000000
EOF

# Unmount
sudo umount /mnt/boot
```

### Step 5: Verify and Test

```bash
# Safely eject SD card
sync
sudo eject /dev/sdX
```

Insert the SD card into your FPGA board and power on. Connect to the serial console:

```bash
screen /dev/ttyUSB0 115200
# or
minicom -D /dev/ttyUSB0 -b 115200
```

## Alternative: Using Chipyard Scripts

Chipyard provides helper scripts for SD card creation:

```bash
cd fpga/scripts

# Use the SD card creation script
./create-sd-card.sh \
  --device /dev/sdX \
  --bitstream ../generated-src/PROJECT.CONFIG/obj/bitstream.bit \
  --fsbl ../generated-src/PROJECT.CONFIG/obj/fsbl.elf \
  --kernel ../../linux-build/Image \
  --rootfs ../../buildroot/output/images/rootfs.tar.gz
```

## Troubleshooting

### RTL Generation Issues

**Problem:** Chisel elaboration fails
```bash
# Check Scala/Java versions
java -version
scala -version

# Rebuild from clean state
make clean
make CONFIG=YourConfig
```

**Problem:** Out of memory during generation
```bash
# Increase JVM heap size
export JVM_MEMORY=8G
make CONFIG=YourConfig
```

### SD Card Boot Issues

**Problem:** Board doesn't boot
- Check serial console output
- Verify boot partition is FAT32 and marked bootable
- Ensure FSBL and BBL paths are correct

**Problem:** Kernel panic
- Check device tree compatibility
- Verify rootfs partition UUID matches boot arguments
- Check console device in kernel command line

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
