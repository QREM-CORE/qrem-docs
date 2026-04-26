# QREM Core Documentation

## Overview
Welcome to the documentation repository for the **QREM Core**, a cycle-accurate hardware accelerator for the FIPS 203 Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM).

This repository serves as the single source of truth for the system's microarchitecture, interfaces, integration protocols, and architectural decisions. It is structured to be highly modular and easily navigable by both human engineers and agentic AI assistants.

## 🗺️ Quick Start & Integration
If you are looking for top-level integration details, start here:
* [**System Architecture (`SYSTEM-ARCHITECTURE.md`)**](SYSTEM-ARCHITECTURE.md) - Top-level block diagrams, clock domains, global memory maps, and execution dataflow.
* [**Architecture Decision Records (`decisions/`)**](decisions/) - The historical log of *why* major architectural changes were made (e.g., area trade-offs, pipeline decoupling).

## 📁 Subsystem Directory Map
The QREM Core is partitioned into the following isolated hardware modules. Click into any directory for its specific datapath, FSM, and interface documentation:

* [**Core Control Unit (CCU)**](core-control-unit/README.md) - The master orchestration FSM controlling Key Generation, Encapsulation, and Decapsulation sequences.
* [**Hash Sampler Unit (HSU)**](hash-sampler-unit/README.md) - FIPS 202 Keccak core (SHA3/SHAKE) and Centered Binomial Distribution (CBD) samplers.
* [**Polynomial Arithmetic Unit (PAU)**](poly-arith-unit/README.md) - The NTT/INTT butterfly datapath and modular reduction logic.
* [**Transcoder Unit (TRU)**](transcoder-unit/README.md) - ByteEncode/Decode formatting and 12x24-bit fixed-point polynomial compression math.
* [**Poly-Mem Subsystem (PMS)**](poly-mem-subsystem/README.md) - The 4-bank polynomial SRAM, dual-port arbiter, and sideband wipe logic.

## 🛠️ Verification & CI/CD
Our RTL is validated using a strict bottom-up **TAID (Test, Analysis, Inspection, Demonstration)** methodology:
1. **Test:** Bit-wise equivalence checked against our [Python Golden Model](https://github.com/QREM-CORE/mlkem-python) via Verilator regressions.
2. **Analysis & Inspection:** Automated PR gating using a custom Yosys/Slang "Scatter-Gather" synthesis pipeline to extract Area/Timing metrics and prevent inferred latches.

---
*Note to AI Agents: When evaluating system-level changes, always parse `SYSTEM-ARCHITECTURE.md` before making datapath modifications to sub-modules.*
