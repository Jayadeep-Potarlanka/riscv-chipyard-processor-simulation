# RISC-V Processor Simulation and Analysis with Chipyard

This repository documents a project focused on the simulation and performance analysis of various RISC-V processor configurations using the Chipyard framework. The work involves running simulations on instructional Sodor processor cores and performing custom analysis by modifying standard Chipyard scripts.

The project was conducted on a 64-bit machine with the following specifications:
- Processor: 11th Gen Intel(R) Core(TM) i5-1155G7 @ 2.50GHz
- Installed RAM: 16.0 GB

## Project Overview

The goal of this project was to gain a practical understanding of computer architecture by exploring the trade-offs in different processor pipeline designs. This was achieved by:
1. Setting up the Chipyard simulation environment.
2. Simulating 1-stage, 2-stage, 3-stage, 5-stage, and micro-coded Sodor cores.
3. Collecting and tabulating cycle counts, instruction counts, and CPI for all benchmarks on all cores.
4. Analyzing the performance impact of pipeline depth and data bypassing.
5. Extending the existing tracer script to compute detailed instruction mix statistics and measure the fraction of loads/stores with non-zero offsets.
6. Using these measurements to evaluate a proposed architectural redesign.

---

## 1. Environment Setup

The project began with the installation of Chipyard and its dependencies on Linux.

### 1.1 Key Setup Commands

Install prerequisite packages:
```
sudo apt-get install -y build-essential bison flex sbt git verilator python3
sudo apt-get install -y libgmp-dev libmpfr-dev libmpc-dev zlib1g-dev vim default-jdk default-jre
sudo apt-get install -y texinfo gengetopt libexpat1-dev libusb-dev libncurses5-dev cmake
sudo apt-get install -y python3-pip python3-dev rsync libguestfs-tools device-tree-compiler
sudo apt-get install -y autoconf
```
Clone the Chipyard repository:
```
git clone https://github.com/ucb-bar/chipyard.git
cd chipyard
```

Initialize submodules, including the riscv-sodor generator:
```
./scripts/init-submodules-no-riscv-tools.sh
```

Build the RISC-V toolchain:
```
./scripts/build-toolchains.sh riscv-tools
```

Set up environment variables:
```
source env.sh
```

## 2. Simulation Methodology

All simulations were conducted using the Verilator-based flow in Chipyard.

### 2.1 Core Configurations

The following Sodor cores were used:

- `Sodor1StageConfig`: 1-stage pipeline
- `Sodor2StageConfig`: 2-stage pipeline
- `Sodor3StageConfig`: 3-stage pipeline
- `Sodor5StageConfig`: 5-stage pipeline (with configurable bypassing)
- `SodorUCodeConfig`: micro-coded implementation

### 2.2 Benchmarks

The following benchmarks were run on each configuration:

- dhrystone.riscv
- median.riscv
- multiply.riscv
- qsort.riscv
- rsort.riscv
- towers.riscv (Towers of Hanoi)
- vvadd.riscv

### 2.3 Simulation Commands

From `chipyard/sims/verilator`:

Build a configuration:
```
make CONFIG=Sodor1StageConfig
```

Run a benchmark:
```
make CONFIG=Sodor1StageConfig run-binary BINARY=~/chipyard/generators/riscv-sodor/riscv-bmarks/towers.riscv
```

---

## 3. Complete Simulation Results

### 3.1 1-Stage Processor - All Benchmarks

| Benchmark | mcycle | minstret |
|-----------|--------|----------|
| towers.riscv | 6166 | 6172 |
| dhrystone.riscv | 224524 | 224530 |
| median.riscv | 4149 | 4155 |
| qsort.riscv | 123503 | 123509 |
| rsort.riscv | 171128 | 171134 |
| vvadd.riscv | 2412 | 2418 |


### 3.2 Towers of Hanoi Across All Processor Configurations

| Processor Configuration | mcycle | minstret |
|-------------------------|--------|----------|
| Sodor 1-stage | 6166 | 6172 |
| Sodor 2-stage | 6582 | 6172 |
| Sodor 3-stage | 7414 | 6172 |
| Sodor 5-stage | 7000 | 6172 |
| Sodor UCodeConfig | 45051 | 6172 |

---

## 4. Instruction Mix Analysis (1-Stage Processor)

All benchmarks on the 1-stage processor have CPI = 1.000, IPC = 1.000, and Bubbles = 0.

| Benchmark | Cycles | Instructions | Arithmetic (%) | Ld/St (%) | Branch/Jump (%) | Misc (%) |
|-----------|--------|--------------|----------------|-----------|-----------------|----------|
| dhrystone.riscv | 245738 | 245739 | 40.379 | 35.324 | 23.757 | 0.541 |
| median.riscv | 17173 | 17174 | 31.845 | 32.147 | 35.193 | 0.815 |
| multiply.riscv | 50619 | 50620 | 63.151 | 4.883 | 31.618 | 0.348 |
| qsort.riscv | 236620 | 236621 | 38.382 | 31.471 | 29.825 | 0.322 |
| rsort.riscv | 375222 | 375223 | 59.580 | 34.870 | 4.398 | 1.152 |
| towers.riscv | 19612 | 19613 | 41.702 | 42.197 | 15.388 | 0.714 |
| vvadd.riscv | 12955 | 12956 | 45.878 | 30.573 | 22.468 | 1.081 |

**Key Observations:**
- **Highest arithmetic intensity:** multiply.riscv (63.151%)
- **Most memory-bound:** towers.riscv (42.197% Ld/St)
- **Most branch-dependent:** median.riscv (35.193% branch/jump)

---

## 5. CPI Analysis - 5-Stage Processor

This analysis evaluated the impact of data bypassing (forwarding) on the 5-stage Sodor processor.

### 5.1 Methodology: Code Modification in `consts.scala`

To generate both bypassed and interlocked versions of the processor, the following parameter in the Sodor 5-stage core was modified:

**File:** `generators/riscv-sodor/src/main/scala/rv32_5stage/consts.scala`
```
// For full bypassing (default)
val USE_FULL_BYPASSING = true

// For full interlocking (no bypassing)
val USE_FULL_BYPASSING = false
```

Two complete simulation runs were performed—one with each setting—to collect comparative CPI data across all benchmarks.

### 5.2 Results: Full Bypassing Enabled (USE_FULL_BYPASSING = true)

| Benchmark | CPI | IPC | Cycles | Instructions | Bubbles |
|-----------|-----|-----|--------|--------------|---------|
| dhrystone.riscv | 1.323 | 0.756 | 322021 | 243402 | 78619 |
| median.riscv | 1.469 | 0.681 | 24272 | 16523 | 7749 |
| multiply.riscv | 1.565 | 0.639 | 78288 | 50024 | 28264 |
| qsort.riscv | 1.421 | 0.704 | 335367 | 236008 | 99359 |
| rsort.riscv | 1.083 | 0.923 | 405576 | 374493 | 31083 |
| towers.riscv | 1.250 | 0.800 | 23599 | 18879 | 4720 |
| vvadd.riscv | 1.352 | 0.740 | 16563 | 12251 | 4312 |

### 5.3 Results: Full Interlocking (USE_FULL_BYPASSING = false)

| Benchmark | CPI | IPC | Cycles | Instructions | Bubbles |
|-----------|-----|-----|--------|--------------|---------|
| dhrystone.riscv | 1.986 | 0.504 | 481190 | 242291 | 238899 |
| median.riscv | 1.888 | 0.530 | 30482 | 16145 | 14337 |
| multiply.riscv | 1.910 | 0.524 | 94872 | 49671 | 45201 |
| qsort.riscv | 1.935 | 0.517 | 456012 | 235665 | 220347 |
| rsort.riscv | 2.323 | 0.430 | 869566 | 374329 | 495237 |
| towers.riscv | 1.672 | 0.598 | 30955 | 18514 | 12441 |
| vvadd.riscv | 1.839 | 0.544 | 21928 | 11924 | 10004 |

### 5.4 Comparing the CPI of different benchmarks with and without bypassing
<img width="580" height="392" alt="image" src="https://github.com/user-attachments/assets/c5fddf61-043b-42d5-ab5f-c0047c790849" />

### 5.5 CPI Improvement from Bypassing

| Benchmark | CPI (No Bypass) | CPI (Bypass) | Improvement (%) |
|-----------|-----------------|--------------|-----------------|
| Dhrystone.riscv | 1.986 | 1.323 | 33.4 |
| Median.riscv | 1.888 | 1.469 | 22.2 |
| Multiply.riscv | 1.910 | 1.565 | 18.1 |
| Qsort.riscv | 1.935 | 1.421 | 26.6 |
| Rsort.riscv | 2.323 | 1.083 | 53.4 |
| Towers.riscv | 1.672 | 1.250 | 25.2 |
| Vvadd.riscv | 1.839 | 1.352 | 26.5 |

**Key Finding:** Full bypassing reduces CPI by 18-53% across all benchmarks by enabling data forwarding between pipeline stages and reducing the number of pipeline stalls. Rsort shows the highest improvement (53.4%), while Multiply shows the smallest (18.1%).

---

## 6. Analysis for Modified Load/Store Behavior

This section evaluates a design modification("Designing own 5-stage processor"). The modification would require that all load and store instructions use a zero offset. Any access with a non-zero offset would first require a separate `add` instruction to compute the address.

### 6.1 Methodology: Tracer Script Extension in `tracer.py`

To quantify the impact of this design change, the existing tracer script was extended with additional analysis logic.

**File:** `generators/riscv-sodor/scripts/tracer.py`

The original tracer already:
- Parsed simulation trace files
- Counted total instructions
- Computed CPI and basic instruction mix (Arithmetic, Load/Store, Branch/Jump, Misc)

The following logic was added:
- Identify load (I-type, opcode `0000011`) and store (S-type, opcode `0100011`) instructions
- Decode the immediate (offset) field from the 32-bit RISC-V instruction encoding
- Separate loads/stores into those with zero offset vs. non-zero offset
- Count the fraction of instructions that are loads/stores with non-zero offsets
- Report these statistics for each benchmark

### 6.2 Results: Impact on Instruction Count

The table below shows how the total number of executed instructions would increase under the modified design, as each non-zero-offset load/store would be decomposed into two instructions (an `add` plus a zero-offset load/store).

| Benchmark | Old Instructions | % Non-Zero Offset Ld/St | New Instructions | % Increase |
|-----------|------------------|-------------------------|------------------|------------|
| Dhrystone.riscv | 245739 | 23.981 | 304670 | 23.981 |
| Median.riscv | 17174 | 16.522 | 20011 | 16.522 |
| Multiply.riscv | 50620 | 1.885 | 51574 | 1.885 |
| Qsort.riscv | 236621 | 17.282 | 277513 | 17.282 |
| Rsort.riscv | 375223 | 13.108 | 424407 | 13.108 |
| Towers.riscv | 19613 | 31.213 | 25735 | 31.213 |
| Vvadd.riscv | 12956 | 8.562 | 14065 | 8.562 |

### 6.3 Results: Detailed Instruction Mix with Zero/Non-Zero Offset Breakdown

The extended tracer produced detailed instruction mix statistics showing the breakdown between loads/stores with zero offset versus non-zero offset:

**Dhrystone (5-stage):**
- Arithmetic: 40.531%
- Ld/St with zero offset: 11.217%
- Ld/St with non-zero offset: 23.981%
- Branch/Jump: 23.725%
- Misc: 0.546%

**Median (5-stage):**
- Arithmetic: 32.470%
- Ld/St with zero offset: 14.882%
- Ld/St with non-zero offset: 16.522%
- Branch/Jump: 35.278%
- Misc: 0.847%

**Multiply (5-stage):**
- Arithmetic: 63.791%
- Ld/St with zero offset: 2.353%
- Ld/St with non-zero offset: 1.885%
- Branch/Jump: 31.619%
- Misc: 0.352%

**Quicksort (5-stage):**
- Arithmetic: 38.448%
- Ld/St with zero offset: 14.122%
- Ld/St with non-zero offset: 17.282%
- Branch/Jump: 29.826%
- Misc: 0.323%

**Rsort (5-stage):**
- Arithmetic: 59.661%
- Ld/St with zero offset: 21.736%
- Ld/St with non-zero offset: 13.108%
- Branch/Jump: 4.340%
- Misc: 1.155%

**Towers of Hanoi (5-stage):**
- Arithmetic: 42.544%
- Ld/St with zero offset: 10.722%
- Ld/St with non-zero offset: 31.213%
- Branch/Jump: 14.780%
- Misc: 0.742%

**Vvadd (5-stage):**
- Arithmetic: 47.472%
- Ld/St with zero offset: 20.899%
- Ld/St with non-zero offset: 8.562%
- Branch/Jump: 21.926%
- Misc: 1.142%

### 6.4 Design Recommendation

Based on this quantitative analysis, the proposed design modification is **not recommended** for most applications.

**Reasoning:**
- The increase in dynamic instruction count ranges from 1.9% (Multiply) to 31.2% (Towers of Hanoi).
- Most benchmarks experience instruction count increases well above 10%, which would significantly impact performance.
- Memory-bound programs like Towers of Hanoi would be particularly impacted, requiring 31.2% more instructions.

**Trade-offs:**
- While the modified design could simplify the processor's datapath and potentially reduce area and power consumption, the performance degradation in terms of increased instruction count is too severe for most general-purpose workloads.
- The original design, which supports non-zero offsets directly in load/store instructions, is more efficient for typical programs.
- The modified design might only be suitable for specialized applications where power and area efficiency are paramount and performance is secondary.

---

## 7. Key Findings

1. **Pipeline Depth Impact:** Deeper pipelines generally exhibit higher cycle counts for the same instruction count due to increased control and data hazards. The U-Code implementation shows dramatically higher CPI (7-9x) due to its micro-coded nature.

2. **Bypassing is Critical:** Enabling full bypassing in the 5-stage processor reduces CPI by 18-53% depending on the benchmark. Rsort shows the highest improvement (53.4%), demonstrating the critical importance of data forwarding in pipelined processors.

3. **Instruction Mix Variability:** Different benchmarks exhibit vastly different execution characteristics:
   - Multiply is arithmetic-intensive (63% arithmetic operations)
   - Towers of Hanoi is memory-bound (42% load/store operations)
   - Median is branch-heavy (35% branch/jump operations)
   
   This variability highlights the importance of balanced processor design that can handle diverse workloads efficiently.

4. **Design Trade-offs:** The analysis of the modified load/store behavior demonstrates that even seemingly minor architectural changes can have significant performance implications. Most benchmarks would require 10-30% more instructions under the modified design, making it unsuitable for performance-critical applications.

5. **Memory Access Patterns:** A significant fraction (8-31%) of all executed instructions are loads or stores with non-zero offsets, highlighting the importance of efficient address calculation support in processor design. Removing this feature would substantially increase instruction counts across most workloads.

---

## 8. Repository Structure

This repository is documentation-focused and serves as a portfolio artifact. It:

- Describes the Chipyard-based workflow followed in the project
- Summarizes the key simulation and CPI results
- Records the concrete configuration and tracer modifications that were implemented

To reproduce the experiments end-to-end:

1. Clone the official Chipyard repository
2. Apply the `consts.scala` modifications to toggle bypassing as needed
3. Implement the tracer extensions in `tracer.py` to measure load/store instructions with non-zero offsets
4. Build and run all Sodor configurations with the full benchmark suite
5. Run the tracer on generated trace files to collect instruction mix and CPI statistics
