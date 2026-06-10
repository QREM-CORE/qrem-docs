# Poly-Mem Subsystem Interfaces

## 1. Overview

`poly_mem_subsystem` presents three polynomial-memory client interfaces (PAU primary, PAU auxiliary, HSU, Transcoder), two seed/protocol store client interfaces (HSU-side, Transcoder-side), and a sideband control interface to the Main Controller. All client interfaces use a request/stall/valid handshake with 4-lane coefficient vectors; the seed interface uses a request/ready/rvalid protocol.

---

## 2. Clocks and Resets

| Port | Direction | Description |
| :--- | :--- | :--- |
| `clk` | Input | Core clock. All registers are synchronous to the rising edge. |
| `rst` | Input | Active-high synchronous reset. Clears all state including wipe FSM counters and read-owner registers. During reset both seed ports report not-ready. |

---

## 3. Parameters

| Parameter | Default | Source | Description |
| :--- | :--- | :--- | :--- |
| `NUM_POLYS` | `32` | `qrem_global_pkg` | Total polynomial slots in the memory array. |
| `NCOEFF` | `256` | `qrem_global_pkg` | Coefficients per polynomial. Must be divisible by 4. |
| `W` | `16` | Hardcoded | Physical SRAM word width (bits). Only `COEFF_W` LSBs are meaningful. |
| `COEFF_W` | `12` | `qrem_global_pkg` | Coefficient width (`ceil(log2(q))`, q=3329). |
| `SEED_DEPTH` | `32` | `qrem_global_pkg` | Number of 64-bit words in the seed/protocol store. |
| `SEED_W` | `64` | `qrem_global_pkg` | Seed store word width. 256-bit objects occupy 4 consecutive words (`SEED_BEATS=4`). |

---

## 4. Sideband (Main Controller) Interface

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `wipe_i` | Input | 1 | Pulse to trigger a full security wipe of polynomial memory and seed store. Sample on rising `clk`. |
| `wipe_busy_o` | Output | 1 | Asserted for the entire duration of a wipe operation (combinational from FSM state ≠ IDLE). |
| `wipe_done_o` | Output | 1 | Single-cycle pulse when wipe completes. Asserted in the `WIPE_DONE` state, clears the next cycle. |
| `mem_fault_o` | Output | 1 | Sticky fault flag driven from `poly_mem_wrapper_4bank`. Indicates a scheduler bug reached the memory layer. |
| `mem_fault_code_o` | Output | 3 | Fault type: `3'b001` = RW same address, `3'b010` = WW same address, `3'b011` = request conflict. |

---

## 5. Polynomial Memory Client Interfaces

All polynomial clients use the same port pattern. Each client has 4 read-data lanes and 4 write-data lanes. Addresses are expressed as `(poly_id, coeff_index)` pairs; the arbiter maps these to physical bank/row addresses internally.

> **Critical:** Only one operation type (read **or** write) may be asserted per cycle per descriptor. A request with both `rd_en` and `wr_en` active simultaneously is illegal and will result in a memory fault. The PAU can issue a read and a write in the **same** cycle only via its two separate descriptors (primary + auxiliary).

### 5.1 PAU Primary Polynomial Interface

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `pau_req` | Input | 1 | Request strobe. Must be held high for the duration of a multi-beat transaction. |
| `pau_rd_en` | Input | 1 | Asserted with `pau_req` to indicate a read operation. |
| `pau_rd_poly_id` | Input | `clog2(NUM_POLYS)` = 5 | Polynomial slot index for the read (0–31). |
| `pau_rd_idx[3:0]` | Input | 4×`clog2(NCOEFF)` = 4×8 | Coefficient address for each of the 4 read lanes. |
| `pau_rd_lane_valid[3:0]` | Input | 4 | Per-lane read-enable mask. Lanes with this bit cleared will not be driven to memory. |
| `pau_wr_en[3:0]` | Input | 4 | Per-lane write-enable mask. Indicates a write operation when any bit is set. |
| `pau_wr_poly_id` | Input | 5 | Polynomial slot index for the write (0–31). |
| `pau_wr_idx[3:0]` | Input | 4×8 | Coefficient address for each of the 4 write lanes. |
| `pau_wr_data[3:0]` | Input | 4×`COEFF_W` = 4×12 | Write data. Zero-padded to 16 bits internally. |
| `pau_rd_valid` | Output | 1 | Read data valid. Asserted the cycle after a read fires, echoing the request address metadata. |
| `pau_rd_poly_id_o` | Output | 5 | Echoed poly ID from the accepted read request. |
| `pau_rd_idx_o[3:0]` | Output | 4×8 | Echoed coefficient addresses for each lane. |
| `pau_rd_lane_valid_o[3:0]` | Output | 4 | Echoed lane mask from the accepted read request. |
| `pau_rd_data[3:0]` | Output | 4×12 | Read data, masked to `COEFF_W` (12) bits per lane. Valid only when `pau_rd_valid` is high. |
| `pau_stall` | Output | 1 | Backpressure. When asserted, the arbiter did not accept the request this cycle. **DO NOT** combinatorially feed back into `pau_req` or `pau_rd_en`. |

### 5.2 PAU Auxiliary Polynomial Interface

Identical port pattern to PAU primary (with `pau_aux_` prefix). Enabled when both `pau_req` and `pau_aux_req` are asserted simultaneously. When active, the PAU owns both generic memory ports atomically.

> Each descriptor must be a pure read **or** a pure write. The dual-port mode is illegal if either descriptor asserts both `rd_en` and `wr_en` (`pau_both_req` or `pau_aux_both_req`).

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `pau_aux_req` | Input | 1 | Request strobe for the auxiliary descriptor. |
| `pau_aux_rd_en` | Input | 1 | Auxiliary read enable. |
| `pau_aux_rd_poly_id` | Input | 5 | Poly ID for auxiliary read. |
| `pau_aux_rd_idx[3:0]` | Input | 4×8 | Coefficient addresses for auxiliary read lanes. |
| `pau_aux_rd_lane_valid[3:0]` | Input | 4 | Per-lane read mask for auxiliary descriptor. |
| `pau_aux_wr_en[3:0]` | Input | 4 | Per-lane write mask for auxiliary descriptor. |
| `pau_aux_wr_poly_id` | Input | 5 | Poly ID for auxiliary write. |
| `pau_aux_wr_idx[3:0]` | Input | 4×8 | Coefficient addresses for auxiliary write lanes. |
| `pau_aux_wr_data[3:0]` | Input | 4×12 | Write data for auxiliary descriptor. |
| `pau_aux_rd_valid` | Output | 1 | Auxiliary read data valid. |
| `pau_aux_rd_poly_id_o` | Output | 5 | Echoed poly ID. |
| `pau_aux_rd_idx_o[3:0]` | Output | 4×8 | Echoed coefficient addresses. |
| `pau_aux_rd_lane_valid_o[3:0]` | Output | 4 | Echoed lane mask. |
| `pau_aux_rd_data[3:0]` | Output | 4×12 | Auxiliary read data (12 bits per lane). |

### 5.3 HSU Polynomial Interface

Same port pattern as PAU primary (with `hsu_` prefix). Important access restrictions apply:

- **Writes:** Always legal during HSU sampling/matrix-fill operations.
- **Reads:** Only legal during the `KG_HSU_HASH_EK` operation. `hsu_hash_ek_read_en` must be asserted, and `hsu_rd_poly_id` must be one of `POLY_ID_T0`–`POLY_ID_T3` (IDs 9–12). Any other read request is flagged as `hsu_poly_rd_unsupported` and stalled.

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `hsu_hash_ek_read_en` | Input | 1 | Authorization gate for T-slot readout. Must be asserted by the main controller only during the `KG_HSU_HASH_EK` phase. |
| `hsu_req` | Input | 1 | Request strobe. |
| `hsu_rd_en` | Input | 1 | Read request (restricted, see above). |
| `hsu_rd_poly_id` | Input | 5 | Must be T0–T3 (9–12) when `hsu_hash_ek_read_en` is high. |
| `hsu_rd_idx[3:0]` | Input | 4×8 | Coefficient addresses for read lanes. |
| `hsu_rd_lane_valid[3:0]` | Input | 4 | Per-lane read mask. |
| `hsu_wr_en[3:0]` | Input | 4 | Per-lane write enable. |
| `hsu_wr_poly_id` | Input | 5 | Poly ID for write. |
| `hsu_wr_idx[3:0]` | Input | 4×8 | Coefficient addresses for write lanes. |
| `hsu_wr_data[3:0]` | Input | 4×12 | Write data. |
| `hsu_rd_valid` | Output | 1 | Read data valid (1 cycle after accepted read). |
| `hsu_rd_poly_id_o` | Output | 5 | Echoed poly ID. |
| `hsu_rd_idx_o[3:0]` | Output | 4×8 | Echoed coefficient addresses. |
| `hsu_rd_lane_valid_o[3:0]` | Output | 4 | Echoed lane mask. |
| `hsu_rd_data[3:0]` | Output | 4×12 | Read data. |
| `hsu_stall` | Output | 1 | Backpressure. DO NOT feed back combinatorially. |

### 5.4 Transcoder Polynomial Interface

Same port pattern as PAU primary (with `tr_` prefix). Transcoder supports both reads and writes but may assert only one per cycle.

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `tr_req` | Input | 1 | Request strobe. |
| `tr_rd_en` | Input | 1 | Read enable. |
| `tr_rd_poly_id` | Input | 5 | Poly ID for read. |
| `tr_rd_idx[3:0]` | Input | 4×8 | Coefficient addresses for read lanes. |
| `tr_rd_lane_valid[3:0]` | Input | 4 | Per-lane read mask. |
| `tr_wr_en[3:0]` | Input | 4 | Per-lane write enable. |
| `tr_wr_poly_id` | Input | 5 | Poly ID for write. |
| `tr_wr_idx[3:0]` | Input | 4×8 | Coefficient addresses for write lanes. |
| `tr_wr_data[3:0]` | Input | 4×12 | Write data. |
| `tr_rd_valid` | Output | 1 | Read data valid. |
| `tr_rd_poly_id_o` | Output | 5 | Echoed poly ID. |
| `tr_rd_idx_o[3:0]` | Output | 4×8 | Echoed addresses. |
| `tr_rd_lane_valid_o[3:0]` | Output | 4 | Echoed lane mask. |
| `tr_rd_data[3:0]` | Output | 4×12 | Read data. |
| `tr_stall` | Output | 1 | Backpressure. DO NOT feed back combinatorially. |

---

## 6. Seed / Protocol Store Interfaces

The seed store is an independent 32×64-bit RAM with two dedicated ports: one for HSU (port A) and one for Transcoder (port B). It is addressed using `(seed_id, beat_index)` semantic tuples, which are translated to linear word addresses via `qrem_seed_map_pkg::seed_word_addr()`.

Both ports are gated by `seed_access_ready = ~rst && ~wipe_active`. Neither port can access the seed store during reset or a wipe.

### Seed ID Map

| `seed_id_e` | Base Word Addr | Object | Size |
| :--- | :--- | :--- | :--- |
| `SEED_ID_D` | 0 | Host-provided d | 256-bit (4 words) |
| `SEED_ID_Z` | 4 | Host-provided z | 256-bit (4 words) |
| `SEED_ID_M` | 8 | Message m / raw message | 256-bit (4 words) |
| `SEED_ID_RHO` | 12 | Public seed ρ for matrix A | 256-bit (4 words) |
| `SEED_ID_SIGMA` | 16 | Internal PRF/CBD seed σ | 256-bit (4 words) |
| `SEED_ID_HEK` | 20 | H(ek) 256-bit digest | 256-bit (4 words) |
| `SEED_ID_SS` | 24 | Shared secret / result | 256-bit (4 words) |
| `SEED_ID_TMP` | 28 | Temporary/reserved | 256-bit (4 words) |

### 6.1 HSU Seed Port

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `hsu_seed_req` | Input | 1 | Request strobe. Read or write depending on `hsu_seed_we`. |
| `hsu_seed_we` | Input | 1 | Write enable. Low = read, High = write. |
| `hsu_seed_id` | Input | `seed_id_e` (4-bit enum) | Logical seed object ID. |
| `hsu_seed_idx` | Input | `clog2(SEED_BEATS)` = 2 | Beat offset within the 256-bit seed object (0–3). |
| `hsu_seed_wdata` | Input | 64 | Write data (one 64-bit word). |
| `hsu_seed_ready` | Output | 1 | Asserted when the port can accept a request. Combinational: `~rst && ~wipe_active`. |
| `hsu_seed_rvalid` | Output | 1 | Asserted the cycle after a read request fires. Registered delay of 1 cycle from `hsu_seed_read_fire`. |
| `hsu_seed_rdata` | Output | 64 | Read data. Valid only when `hsu_seed_rvalid` is asserted. |

### 6.2 Transcoder Seed Port

Identical to the HSU seed port (with `tr_seed_` prefix). Uses RAM Port B independently.

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `tr_seed_req` | Input | 1 | Request strobe. |
| `tr_seed_we` | Input | 1 | Write enable. |
| `tr_seed_id` | Input | 4-bit enum | Logical seed object ID. |
| `tr_seed_idx` | Input | 2 | Beat offset (0–3). |
| `tr_seed_wdata` | Input | 64 | Write data. |
| `tr_seed_ready` | Output | 1 | `~rst && ~wipe_active`. |
| `tr_seed_rvalid` | Output | 1 | 1 cycle after accepted read. |
| `tr_seed_rdata` | Output | 64 | Read data. Valid only when `tr_seed_rvalid` is asserted. |

---

## 7. Handshaking and Protocol Rules

### Polynomial Memory
- **Backpressure:** When `*_stall` is asserted, the client must hold its request signals stable. The request has not been accepted.
- **Read Latency:** Exactly **1 cycle** from the cycle a read request fires (stall low, request accepted) to the cycle `*_rd_valid` asserts with valid data.
- **Combinational Loop Rule:** `*_stall` must never be used to gate `*_req`, `*_rd_en`, or `*_wr_en` in the same combinational cone. Stall is a consequence of the request, not an input to it.
- **Both-read-and-write:** Illegal on a single descriptor per cycle. Use PAU dual-port mode if simultaneous RW is needed.

### Seed Store
- **Ready before request:** Check `*_seed_ready` before asserting `*_seed_req`. If not ready (during wipe/reset), transactions are silently dropped.
- **Read latency:** 1 cycle. `*_seed_rvalid` asserts the cycle after `*_seed_read_fire` (i.e., `seed_access_ready && req && ~we`).
- **No arbitration:** The two seed ports (HSU = Port A, Transcoder = Port B) are independent and do not contend. They may both be active in the same cycle.
