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
---

## Introduction

Chipyard is an open-source framework for agile development of specialized systems-on-chip (SoCs). It integrates multiple RISC-V generators and provides a complete RTL-to-GDS flow using open-source tools.

## Prerequisites

Before starting with Chipyard, ensure you have:

- Linux or macOS system
- At least 16GB RAM (32GB recommended)
- 50GB+ free disk space
- Basic knowledge of Verilog/Chisel
- Familiarity with command-line tools

## Installation

### System Dependencies

First, install the required system packages:

```bash
# Ubuntu/Debian
sudo apt-get install autoconf automake autotools-dev curl python3 \
  libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex \
  texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev

# macOS (using Homebrew)
brew install gawk gnu-sed gmp mpfr libmpc isl zlib expat
```

### Clone Chipyard

Clone the Chipyard repository:

```bash
git clone https://github.com/ucb-bar/chipyard.git
cd chipyard
./scripts/init-submodules.sh
```

### Build the Toolchain

Build the RISC-V toolchain (this takes a while):

```bash
./scripts/build-toolchains.sh riscv-tools
```

## Your First Chipyard SoC

### Source the Environment

```bash
source env.sh
```

### Generate RTL

Generate Verilog for the default RocketChip configuration:

```bash
cd sims/verilator
make CONFIG=RocketConfig
```

### Run a Simulation

Run a simple test program:

```bash
make CONFIG=RocketConfig run-asm-tests
```

## Project Structure

Key directories in Chipyard:

- `generators/` - Hardware generators (Rocket, BOOM, etc.)
- `sims/` - Simulation targets (Verilator, VCS)
- `tools/` - Additional tools and utilities
- `vlsi/` - Physical design flow scripts

## Next Steps

Now that you have Chipyard running, explore:
- [Building Custom SoCs](/tutorials/chipyard-custom-socs/)
- Customizing processor configurations
- Adding peripheral devices

## Troubleshooting

### Common Issues

**Out of memory during compilation:**
- Reduce parallelism: `make -j4` instead of `make -j`
- Close other applications

**Submodule errors:**
- Ensure git submodules are initialized: `git submodule update --init --recursive`

## Resources

- [Chipyard Documentation](https://chipyard.readthedocs.io/)
- [Chipyard GitHub](https://github.com/ucb-bar/chipyard)
- [Chisel/FIRRTL Documentation](https://www.chisel-lang.org/)
