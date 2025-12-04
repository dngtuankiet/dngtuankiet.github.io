---
layout: single
title: "Getting Started with Chipyard"
permalink: /tutorials/chipyard-getting-started/
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

Chipyard has become one of my favorite starting points whenever I need to build or experiment with a RISC-V SoC. It’s an open-source framework packed with generators, peripherals, and a complete path from RTL all the way to GDS using open-source tools. The official documentation is great, but if you’re not used to Scala or Chisel (like I wasn’t at the beginning), it can feel a bit overwhelming.

This tutorial is my attempt to make the first steps a little smoother. I want to walk through the installation, show how to get something running quickly, and explain the pieces in a way that I wish existed when I first touched Chipyard. Once you have a working setup and you see how things connect, the deeper architecture and customization flow will feel much more approachable.

At the time I’m writing this, the latest Chipyard release is [1.13.0](https://github.com/ucb-bar/chipyard/releases/tag/1.13.0). For my own research, I created a separate GitHub repository based on this version and added the modifications I needed. In this tutorial, we’ll work directly with that repo and build things up step by step.


## Prerequisites

Before diving into Chipyard, here are a few things you’ll want to have ready. Most of these come from my own trial and error, so setting them up early will save you a lot of frustration later.

- Ubuntu 22.04 or any Linux distro you’re comfortable with  
- Vivado (and don’t forget to export the Vivado path afterward)
- At least 16GB of RAM — Chipyard builds can get heavy; 32GB feels much smoother
- ~50GB of free disk space
- Basic understanding of Verilog/Chisel and some OOP concepts
- A bit of confidence with command-line tools

Once you have these in place, you’ll be in good shape for the installation.

---

## 1. Installation

### 1.1 System Dependencies

Chipyard needs a handful of development packages before it can build anything. Here’s the list I usually install right away:

```bash
# Ubuntu/Debian
sudo apt-get install autoconf automake autotools-dev curl python3 \
  libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex \
  texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev

sudo apt-get install build-essentials git
```

### 1.2 Install conda or miniconda
Chipyard relies on conda to manage all of its internal dependencies. You can install Miniconda by following the guide here:
https://www.anaconda.com/docs/getting-started/miniconda/install#macos-linux-installation

After installation, activate the base environment. If you see (base) at the front, you're good.
```bash
(base) <username>@<pc-name>:~$
```

### 1.3 Clone gptfdisk
In my setup, I use an SD card to store the software. Because of that, I needed gptfdisk to format and prepare the SD card properly.

```bash
git clone https://github.com/tmagik/gptfdisk.git
cd gptfdisk/
make -j`nproc`
```

This only needs to be built once unless you modify it.

### 1.4 Clone Chipyard and setup the environment

Now we get to the fun part.

Since I work with a modified version of Chipyard for my research, I use my own repository. Feel free to do the same if you prefer tracking your changes separately.

```bash
git clone https://github.com/dngtuankiet/my-chipyard.git chipyard-1.13.0
cd chipyard-1.13.0
```
With your conda base environment active, you can run the Chipyard setup script. This is the basic first-time installation, if you are familiar with the process, please checkout chipyard guidelines to customize your environment.

Chipyard has 11 total setup steps. I skip steps 8 and 9 because I don’t use the Linux-compatible system image in my RISC-V environment. You can adjust this depending on what you need. The installation takes a long time to finish (depending on you system).

```bash
# Inside chipyard-1.13.0 directory
./build-setup.sh -s 8 -s 9
```

Once it finishes, you should see `Setup complete!`. From now on, everytime you start working on chipyard, you have to activate the conda environment.

```bash
# Inside chipyard-1.13.0 directory
source ./env.sh

# You will see the conda environment for chipyard has been activated
# From
(base) <username>@<pc-name>:~<your-pc-path>/chipyard-1.13.0$ 
# To something like
(<your-pc-path>/chipyard-1.13.0/.conda-env) <username>@<pc-name>:~<your-pc-path>/chipyard-1.13.0$ 
```


## 2. Project Structure

### 2.1 Top-Level Overview

Once Chipyard is installed, it helps to get familiar with the directory layout. Here’s the top-level structure:

```
chipyard-1.13.0/
├── base/                 # Main RISC-V system development (similar to fpga/)
├── fpga/                 # Chipyard RISC-V system development
├── generators/           # Hardware generators (Rocket, BOOM, etc.)
├── sims/                 # Simulation targets (Verilator, VCS)
├── tools/                # Additional tools and utilities
└── ...
```

I want to keep track of my modifications and using the chipyard original directory as template. The `fpga/` directory is the chipyard's default RISC-V system developement, therefore, I created copy `base/` which is similar to that. So, most of your hands-on work will happen inside the `base/` directory, so let’s look at that more closely.

### 2.2 Base Directory Structure

The `base/` directory contains everything needed for your RISC-V system development:

```
base/
├── base-fpga-shells/     # FPGA wrapper modules (similar to fpga/fpga-shells)
├── scripts/              # TCL scripts for FPGA programming
├── src/
│   └── main/
│       ├── resources/    # BootROM code
│       └── scala/        # FPGA-specific system configurations
├── sw/                   # Software development and applications
├── Makefile              # Build system for hardware generation
└── upload_SD.sh          # Script to load software onto SD card
```

**Key Components:**
- **base-fpga-shells**: All the FPGA-specific wrapper modules for clocking, I/O, etc.
- **src/main/scala**: Scala/Chisel sources defining system architecture
- **sw**: Application software, drivers, and test programs
- **scripts**: TCL helpers for Vivado
- **upload_SD.sh**: Utility to flash compiled software to SD card storage

## 3. Next Steps

Now that Chipyard is installed and you know your way around the directory tree, you can start experimenting.

Some next steps:
- [Understanding Chipyard RISC-V system](/tutorials/understand-chipyard-system/)
- [Building your first RISC-V system](/tutorials/chipyard-custom-socs/)
- Add your own accelerator as a MMIO peripheral
- Add DMA to your accelrator
- Add your own accelerator as a RoCC accelerator
- Customize the BootROM or software stack
- Customize FPGA Input/Output

## Troubleshooting

### Common Issues

**Resolve any issues if anyone send me questions**


## Resources

- [Chipyard Documentation](https://chipyard.readthedocs.io/)
- [Chipyard GitHub](https://github.com/ucb-bar/chipyard)
- [Chisel/FIRRTL Documentation](https://www.chisel-lang.org/)
