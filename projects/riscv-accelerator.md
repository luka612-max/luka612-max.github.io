---
layout: single
title: ""
permalink: /projects/riscv-accelerator/
author_profile: false
classes: wide-project-layout
mathjax: true
---

## Introduction & Motivation

Hi. The main drive behind this project was to design a complete full-stack hardware-software system for high-performance image processing on FPGA. While studying hardware design and processor architecture, I recognized that real-time image filtering operations (like convolution with sharpen, edge detection, or blur kernels) are computationally intensive and involve repetitive matrix multiplications. I learned that these workloads could be dramatically accelerated using specialized hardware—specifically a systolic array architecture. This led me to design a custom 32-bit RISC-V soft-core processor (RV32IMF) with an integrated 4×4 Systolic Array matrix multiplication engine, creating a tightly coupled hardware accelerator for real-world image processing tasks.(Along with me my 2 friends also contributed to this project).

The implementation of the whole project can be found here: [Github Repository](https://github.com/luka612-max/hardware_accelerator)

## Hardware Design Fundamentals

For the hardware architecture, I studied advanced processor design concepts including pipelined execution, hazard resolution, branch prediction, and floating-point arithmetic. I learned how to design efficient data paths, implement pipeline registers, handle data forwarding, and optimize for throughput and latency. These concepts were applied using Verilog HDL on the Xilinx Vivado toolchain for the Nexys A7-100T FPGA platform.

![Nexys-A7-100T FPGA board](/assets/images/nexys.jpg)

## Project Overview

Our implementation included the following:

* Designing a custom 5-stage pipelined RV32IMF RISC-V processor from scratch in Verilog (~25KB core module).
* Implementing a 4×4 Systolic Array with Floating-Point Multiply-Accumulate (FP-MAC) units for parallel matrix computation.
* Implementing advanced features: branch prediction (BHT + BTB), data forwarding, hazard detection, and IEEE-754 floating-point unit.
* Ping-pong double buffering for latency hiding and concurrent CPU-accelerator operation.
* UART communication protocol for real-time image streaming from host PC.
* Full software stack: RISC-V C firmware, Python host scripts, and Makefile automation.
* Multiple operational modes: simulation, hardware acceleration, and live streaming.

## Instruction Set Architecture (RV32IMF)

The processor implements the RISC-V RV32IMF Instruction Set Architecture with three extensions:

| Extension | Instructions | Purpose |
|-----------|--------------|---------|
| **I (Base Integer)** | ADD, SUB, ADDI, LUI, AUIPC, AND, OR, XOR, SLL, SRL, SRA, etc. | Complete integer ALU operations, branching, memory access |
| **M (Multiply/Divide)** | MUL, MULH, MULHSU, MULHU, DIV, DIVU, REM, REMU | Hardware multiplication and multi-cycle division with pipeline control |
| **F (Single-Precision FP)** | FADD.S, FSUB.S, FMUL.S, FDIV.S, FMADD.S, FMSUB.S, FEQ.S, FLT.S, FCVT.* | IEEE-754 compliant floating-point with 32-entry dedicated register file |

## 5-Stage Pipeline Architecture

The processor employs a classic high-performance 5-stage pipeline: IF → ID → EX → MEM → WB

## Dynamic Branch Prediction Subsystem

A deep pipeline suffers heavy penalties (instruction flushes) if branches are resolved late. To mitigate this, the CPU integrates an advanced branch prediction subsystem:

* **Branch History Table (BHT) (BHT.v):** 64-entry table tracking taken/not-taken history with 1-bit prediction scheme
* **Branch Target Buffer (BTB) (BTB.v):** 64-entry cache storing exact 32-bit destination addresses of previously taken branches
* **Branch Processing Unit (BPU) (BPU.v)**: Located in EX stage, calculates actual branch outcomes and updates predictor tables
* **Mechanism:** On every cycle, IF stage queries BHT and BTB. If prediction hits, PC immediately jumps to cached target, eliminating fetch delay for predicted branches

## Hazard Resolution & Data Forwarding

To achieve high Instructions-Per-Cycle (IPC), the pipeline implements sophisticated hazard mitigation:

```cpp
// Data Forwarding Unit (Forwarding_Unit.v)
// Bypasses computed data from MEM/WB stages directly to EX stage ALUs
// Allows consecutive dependent integer operations to execute without stalling
// Example: ADD r1, r2, r3  (writes r1)
//          ADD r4, r1, r5  (reads r1) - Can execute immediately via forwarding

// Hazard Unit (Hazard_Unit.v)
// Detects Load-Use hazards where data isn't ready until MEM stage
// Stalls pipeline by holding IF/ID stages and inserting bubbles
// Monitors multi-cycle operations (FP divide, integer divide)
// Asserts busy signal to halt execution until operation completes
```

## Floating-Point Unit (FPU)

A dedicated IEEE-754 single-precision floating-point execution engine:

```cpp
// FPU Architecture (FPU.v + FPU_RF.v)
// 32-entry Floating-Point Register File (separate from Integer RF)
// 3 read ports (supports Fused Multiply-Add with 3 operands)
// 1 write port for results
// Multi-cycle execution with internal clock stretching
// Operations: FADD.S, FSUB.S, FMUL.S, FDIV.S, FMADD.S, FMSUB.S
// Conversions: FCVT.W.S, FCVT.S.W (Integer ↔ Floating-Point)
// Comparisons: FEQ.S, FLT.S, FLE.S
```

## 4×4 Systolic Array Matrix Accelerator

The crown jewel of the system is matrix_accel.v, an MMIO-attached specialized hardware engine for accelerating matrix multiplication:

Following is the architecture for the 4×4 Systolic Array:

```cpp
Matrix A (Left Input)  →  →  →  →
                        ↓    ↓    ↓    ↓
                      [MAC] [MAC] [MAC] [MAC]
                        ↓    ↓    ↓    ↓
Matrix B (Top Input)  [MAC] [MAC] [MAC] [MAC]
    ↓                   ↓    ↓    ↓    ↓
    ↓                 [MAC] [MAC] [MAC] [MAC]
    ↓                   ↓    ↓    ↓    ↓
    ↓                 [MAC] [MAC] [MAC] [MAC]
                        ↓    ↓    ↓    ↓
                   Matrix C (Output)
```
* **16 Parallel FP-MAC Units:** Each performs Multiply-Accumulate on single-precision floats
* **Wave-Front Execution:** On every clock cycle:
  * New rows from Matrix A shift left-to-right
  * New columns from Matrix B shift top-to-bottom
  * Partial products accumulate in each PE
* **Compute Latency:** Complete 4×4 matrix multiplication in ~18-20 cycles
* **Clock Stretching:** FP-MAC units are multi-cycle; internal FSM stretches clock to synchronize

## Ping-Pong Double Buffering

The accelerator employs dual-bank memory to hide latency and eliminate CPU stalls:

```cpp
// Problem: Loading 4×4 matrix (16 floats) over 32-bit MMIO bus = 16 cycles
// If CPU waits for data, accelerator starves and performance plummets

// Solution: Ping-Pong Double Buffering
// State 1: Hardware computes using "Ping" buffer
//          CPU simultaneously loads next 4×4 matrix into "Pong" buffer
// State 2: Upon computation completion, FSM swaps buffers instantly
//          Hardware begins computing on "Pong" data
//          CPU reads results from "Ping" output and prepares next input
// Result: 100% utilization; CPU and accelerator operate concurrently
```

## MMIO Register Map

The accelerator exposes the following Memory-Mapped I/O registers:


| Address    | Size  | Purpose |
|------------|-------|---------|
| 0x80000000 | 256B  | Matrix A Buffer (Ping-Pong, 16×32-bit floats) |
| 0x80000100 | 256B  | Matrix B Buffer (Ping-Pong, 16×32-bit floats) |
| 0x80000200 | 256B  | Matrix C Results (Ping-Pong, 16×32-bit floats) |
| 0x80000F00 | 4B    | Control Register (Bit 0: Start, Bit 1: Accumulate, Bit 2: Swap) |

## UART Communication Protocol

Real-time image streaming from host PC to FPGA employs a custom synchronization protocol:

Following is the UART communication frame layout:

Byte Offset | Size (Bytes) | Description
------------|--------------|------------------------------
0x00        | 4            | SYNC_CONV Marker (0xC04F0001)
0x04        | 4            | Image Width (W) in pixels
0x08        | 4            | Image Height (H) in pixels
0x0C        | 4            | Normalization Factor (NORM)
0x10        | 36           | 3×3 Kernel Matrix (9×4-byte floats)
0x34        | 4            | ACK_CONV Response (0xAC400001)
0x38        | W×H          | Raw Pixel Data (1 byte/pixel, grayscale)

**Protocol Flow**

```cpp
Host PC:                           FPGA:
1. Send SYNC_CONV marker      →   Verify synchronization
2. Send image metadata        →   Parse width, height, norm
3. Send 3×3 filter kernel     →   Load into firmware
                              ←   Return ACK_CONV
4. Stream raw pixels          →   Buffer and process via Systolic Array
5. Receive filtered output    ←   Stream convolution results
6. Reconstruct output .jpg    ✓   Complete
```

## Real-Time Image Processing Application

The system applies convolutional filters (sharpen, edge detection, blur) to images by decomposing convolution into matrix multiplications:

```cpp
// Convolution Process:
// Input: Grayscale image (H × W pixels)
// Kernel: 3×3 filter matrix K

// 1. Sliding Window: Extract 3×3 patches, pad boundaries
// 2. Vectorize: Flatten each 3×3 patch into 9-element vector
// 3. Matrix Formation: Arrange vectors into 8×8 or 4×4 matrices
// 4. Accelerate: Send matrices to Systolic Array via MMIO
// 5. Collect Results: Read computed values from output buffer
// 6. Reconstruct: Assemble results back into output image

// Performance: ~100× faster than sequential CPU convolution
```

## FPGA Resource Utilization

The design targets the Nexys A7-100T FPGA with the following typical utilization:

| Resource    | Usage      | % of Available |
|-------------|------------|----------------|
| LUTs        | ~14,500    | 18%           |
| FFs         | ~8,000     | 10%           |
| BRAM        | 4 blocks   | 2.5%          |
| DSPs        | 48 units   | 20%           |
| Clock Freq  | 50 MHz     | Meets timing  |

The design leaves substantial headroom for future enhancements (cache, DMA, vector extensions).

## Conclusion and Learnings

This project was an exceptional learning experience in full-stack hardware design. I gained deep insights into:

* **Processor Architecture:** Designing a pipelined RISC-V CPU from scratch, managing hazards, implementing branch prediction
* **Systolic Arrays:** Understanding data-flow architectures for efficient matrix computation
* **Hardware-Software Co-Design:** Bridging compiled C firmware with custom hardware accelerators via MMIO
* **FPGA Development:** Resource optimization, timing closure, Vivado toolchain mastery
* **Real-World Applications:** Applying acceleration to genuine image processing workloads
* **Software Engineering:** Modular Verilog design, templated C++, comprehensive testing

The project demonstrates how specialized hardware can achieve 100× speedups on compute-intensive workloads when properly co-designed with software. The architecture is extensible for future enhancements like vectorization (RV32V), DMA controllers, and caching hierarchies.

Overall, an incredibly rewarding journey from theory to silicon! 