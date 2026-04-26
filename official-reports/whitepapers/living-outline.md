# QREM Core: FIPS 203 ML-KEM Hardware Accelerator
## Living Whitepaper Outline (Brain Dump)

> **How to use this file:**
> This is a living document. We are not writing the final IEEE paper today. If you solve a hard integration bug, hit a specific timing milestone, or make a major architectural trade-off, drop a bullet point under the relevant section. When it is time to write the official LaTeX `main.tex`, we will mine this file for content.

### Abstract
* [Drop thoughts here on what makes our accelerator unique. E.g., area-efficient ASIC/FPGA hybrid, cycle-accurate FIPS 203 compliance, etc.]

### 1. Introduction
* Context: The NIST Post-Quantum Cryptography standardization (FIPS 203).
* Motivation: Why standard software implementations of ML-KEM are bottlenecked by polynomial arithmetic and hashing.
* Contributions: [List our 2-3 main architectural claims to fame here].

### 2. High-Level System Architecture
* Overview of the top-level integration.
* Dataflow: Key Generation -> Encapsulation -> Decapsulation.
* How the AXI-Stream interfaces connect the major datapath units.

### 3. Hardware Subsystems & Microarchitecture
#### 3.1 Core Control Unit (CCU)
* Top-level FSM orchestration.
* Ideas on concurrent scheduling (e.g., running Keccak while PAU computes NTT).

#### 3.2 Polynomial Arithmetic Unit (PAU)
* NTT/INTT mathematical implementation.
* 12x12-bit multiplier radix tree and modular reduction datapath.
* *Note: Compression math was explicitly removed from here to save ASIC area.*

#### 3.3 Hash Sampler Unit (HSU)
* FIPS 202 Keccak hashing implementation (SHA3, SHAKE).
* Matrix A expansion and CBD sampling logic.

#### 3.4 Transcoder Unit
* ByteEncode/ByteDecode FIPS 203 formatting.
* **Key Idea to mention:** The 12x24-bit pipelined multipliers for fixed-point compression ($m \approx 2^{35}/q$) live here to prevent PAU combinational bloat.

#### 3.5 Poly-Mem Subsystem (PMS)
* 4-bank SRAM polynomial memory structure.
* Deterministic 2-port arbitration and priority routing (PAU > HSU > Transcoder).
* Sideband wipe logic for secure zeroization.

### 4. Verification Methodology (TAID)
* **Golden Model:** Custom Python ML-KEM reference suite (bottom-up validation).
* **Simulation:** Verilator regression vs. ModelSim waveforms.
* **Continuous Integration:** The Scatter-Gather Yosys matrix synthesis pipeline (automated LUT/GE metric extraction).

### 5. Results & Evaluation (PPA Metrics)
* **Throughput/Latency:** Cycle counts for KeyGen, Encaps, and Decaps vs. Software benchmarks.
* **Area:** FPGA LUTs/DSPs (Artix-7/Zynq) vs. ASIC Gate Equivalents (Standard Cell).
* **Timing:** Critical paths and $F_{max}$ achievements.

### 6. Conclusion
* Summary of the physical realizability and standards compliance.
* Future work or integration into larger SoC fabrics.

---
**Random "Shower Thoughts" & Unsorted Notes:**
* *Drop anything that doesn't fit neatly above right here.*
