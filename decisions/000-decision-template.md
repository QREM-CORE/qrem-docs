# ADR [000]: [Short, Descriptive Title]

**Date:** YYYY-MM-DD
**Status:** [Proposed | Accepted | Rejected | Superseded by ADR-XXX]
**Author(s):** [Names]
**Component:** [Top-Level | PAU | HSU | Transcoder | Poly-Mem | CI/CD]

---

## 1. Context and Problem Statement
*Describe the engineering context and the specific problem you are trying to solve. What is the technical limitation, requirement, or bottleneck forcing this decision? Keep it objective and factual.*

* **Example:** "The ML-KEM Compress operation requires multiplication by $m \approx 2^{35}/q$. We originally planned to compute this inside the PAU using the existing NTT butterfly multipliers, assuming 12x12-bit precision was sufficient."

## 2. Decision
*State the final architectural decision clearly and concisely. What is the exact change being made to the datapath, FSM, memory map, or infrastructure?*

* **Example:** "We will decouple the Compression and Decompression logic from the PAU. Dedicated 12x24-bit pipelined multipliers will be instantiated exclusively inside the Transcoder Unit."

## 3. Engineering Justification
*Why is this the best path forward? Reference specific constraints (e.g., FIPS 203 compliance, target clock frequency, DSP slice limits).*

* **Example:** "The compression constant requires 24 bits of precision. While an FPGA DSP48 slice handles 25x17 natively, forcing 12x24-bit multipliers across the highly parallelized PAU butterfly units would cause unacceptable combinational area bloat in our target ASIC standard-cell flow."

## 4. Consequences (PPA Impact)
*List the positive and negative trade-offs of this decision across the standard hardware metrics.*

* **Area:** [e.g., Decreases PAU footprint; slightly increases Transcoder footprint. Overall net reduction in ASIC gate count.]
* **Timing ($F_{max}$):** [e.g., Removes critical path from the PAU multiplier tree, improving overall positive slack.]
* **Throughput / Latency:** [e.g., Adds 2 cycles of pipeline latency to the Transcoder, but allows the PAU to accept the next polynomial earlier.]
* **Complexity / Integration:** [e.g., Simplifies PAU control logic; requires updating the AXI-Stream wrapper on the Transcoder.]

## 5. Alternatives Considered
*What other options did you look at, and specifically why were they rejected? This is critical for proving your engineering rigor.*

* **Alternative A:** [e.g., Multi-cycle multiplication in the PAU.]
    * *Why it was rejected:* [e.g., Would stall the NTT pipeline, destroying our throughput targets.]
* **Alternative B:** [e.g., Truncating the constant to 12 bits.]
    * *Why it was rejected:* [e.g., Mathematically breaks FIPS 203 equivalence; failed Python Golden Model unit tests.]

## 6. References & Traceability
*Link to any relevant PRs, GitHub Issues, IEEE papers, or Requirement IDs (e.g., from Section 6.3 of the Capstone Report).*

* **Requirement ID:** [e.g., REQ-TRC-01]
* **Reference:** [e.g., Email correspondence with Inha University UniPAM researchers; FIPS 203 Section 5.1]
