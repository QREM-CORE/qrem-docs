# Hash Sampler Unit (HSU)

## Overview
The Hash Sampler Unit (HSU) is a high-performance hardware accelerator core responsible for the cryptographic hashing and polynomial sampling required by the ML-KEM (FIPS 203) algorithm. It tightly couples a 1-cycle-per-round Keccak engine with specialized Rejection (NTT) and Centered Binomial Distribution (CBD) samplers, eliminating large intermediate buffers and accelerating matrix/vector generation ($A, s, e$) as well as standard hash functions ($G, H, J, PRF$).

## Directory Map
* [`interfaces.md`](interfaces.md) - Definitions for custom Poly Memory Writer/Reader ports, Seed Memory ports, Raw AXI-Stream bypass, and critical handshaking rules.
* [`datapath.md`](datapath.md) - Pipeline stages, Demux/Mux routing logic, and Keccak integration.
* [`fsm-states.md`](fsm-states.md) - State machine details for the `coeff_to_axis_packer` (multi-phase polynomial absorption and 8-byte gearbox alignment).

## Integration Notes
* **Sticky Status:** The top-level completion signal (`hsu_done_o`) is sticky. It latches high upon operation completion and remains high until cleared by the next `start_i` pulse.
* **Shared Resource:** All hashing (SHA3/SHAKE) and sampling (NTT/CBD) operations multiplex through a single Keccak core to save massive area. Concurrent operations are not supported.
* **Memory Constraints:** The `coeff_to_axis_packer` employs a strict `rd_pending_q` throttle to prevent pipelined read duplication. Downstream memory arbiters MUST respect this single-cycle `hsu_rd_en_o` pulse and respond with `hsu_rd_valid_i` predictably.
