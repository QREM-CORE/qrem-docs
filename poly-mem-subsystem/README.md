# Polynomial Memory Subsystem (`poly_mem_subsystem`)

## Overview

`poly_mem_subsystem` is the unified memory controller, arbiter, and security wipe controller for the QREM Core v0.9. It manages a 32-polynomial, 256-coefficient, 4-bank SRAM array and an independent 32×64-bit seed/protocol store, arbitrating access across three clients (PAU, HSU, Transcoder) through two generic dual-port memory ports, with strict fixed-priority scheduling and deterministic hazard detection.

## Directory Map

| File | Contents |
| :--- | :--- |
| `interfaces.md` | All port definitions, widths, handshaking rules, and backpressure semantics |
| `fsm-states.md` | Security wipe FSM state diagram, transitions, and control outputs |
| `datapath.md` | Memory bank topology, arbiter/scheduler architecture, hazard detection, and read response routing |

## Integration Notes

> **Priority order is fixed and non-configurable: PAU (0) > HSU (1) > Transcoder (2).** This cannot be changed at runtime.

- **PAU dual-port mode:** The PAU exposes a primary (`pau_*`) and an auxiliary (`pau_aux_*`) descriptor. When both are active (`pau_req && pau_aux_req`) and hazard-free, the PAU atomically claims both generic ports (P0 and P1) in the same cycle. No other client may issue in that cycle.

- **HSU read restriction:** HSU polynomial reads are not general-purpose. The only legal read path is the `KG_HSU_HASH_EK` T-slot readout (poly IDs T0–T3), gated by `hsu_hash_ek_read_en`. Any other HSU read request raises `hsu_poly_rd_unsupported` and is stalled/dropped.

- **Combinatorial loop hazard:** `pau_stall`, `hsu_stall`, and `tr_stall` are backpressure outputs only. Clients **MUST NOT** use stall signals to combinatorially generate `req`, `rd_en`, or `wr_en`. Doing so creates a system-level combinational loop that is not detectable by synthesis tools and will cause functional failure.

- **Security wipe:** Asserting `wipe_i` triggers a blocking sequential wipe of all polynomial memory (zeroed row-by-row across all polys) followed by the seed RAM. All three client stall outputs assert for the entire wipe duration. `wipe_busy_o` is high throughout; `wipe_done_o` pulses for one cycle upon completion.

- **Memory fault reporting:** `mem_fault_o` (sticky) and `mem_fault_code_o` (3-bit) are driven from the underlying `poly_mem_wrapper_4bank` fault detection. Fault codes: `001` = RW same address, `010` = WW same address, `011` = request conflict. These indicate scheduler bugs — they should never fire in a correctly integrated system.

- **Reset:** Active-high synchronous. During reset, both seed ports report not-ready. Seed read data is meaningful only when `hsu_seed_rvalid` / `tr_seed_rvalid` is asserted.

- **Data width:** Physical memory stores 16-bit words (`W=16`). Only the lower 12 bits (`COEFF_W=12`, i.e., coefficients mod q=3329) are meaningful. Reads are masked to 12 bits before being returned to clients; writes are zero-padded to 16 bits internally.
