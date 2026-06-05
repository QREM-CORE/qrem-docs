# Polynomial Arithmetic Unit — Interfaces

## 1. Overview

The PAU does not use an AXI4-Stream interface. It communicates with the rest of the system via two mechanisms: a sideband control bus for job dispatch and status, and a dual-port memory interface (primary + auxiliary) to the Poly-Mem Subsystem. The PAU owns both ports and drives all addressing; the memory subsystem handles bank mapping and returns read responses exactly one clock cycle after the request is accepted.

## 2. Clocks and Resets

| Port | Direction | Description |
| :--- | :--- | :--- |
| `clk` | Input | Positive-edge synchronous clock. All registers in the PAU are synchronous to this clock. |
| `rst` | Input | Synchronous, active-high reset. |

## 3. Control & Status Interface (Sideband)

These signals are driven by the system-level Core Control Unit.

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `start_i` | Input | 1 | 1-cycle pulse to launch a new job. Latched internally on the rising edge when the FSM is in `S_IDLE`. |
| `op_type_i` | Input | 4 | Operation mode (`pe_mode_e`). Must be held stable from `start_i` until `done_o`. See table below. |
| `primary_poly_id_i` | Input | `log2(NUM_POLYS)` | Target polynomial slot for NTT/INTT/ADDSUB. Ignored by CWM. |
| `aux_poly_id_i` | Input | `log2(NUM_POLYS)` | Secondary operand slot. Used by ADDSUB for operand Y. Ignored by CWM. |
| `cwm_num_terms_i` | Input | `log2(NUM_POLYS)` | Number of accumulation terms k for CWM (e.g., 2, 3, 4). |
| `is_sub_i` | Input | 1 | ADDSUB polarity: `0` = addition (A+B), `1` = subtraction (A−B). Ignored for other modes. |
| `done_o` | Output | 1 | 1-cycle pulse when the current job has completed. The FSM returns to `S_IDLE` the following cycle. |

### 3.1 `op_type_i` Encoding (`pe_mode_e`)

| Symbol | Value | Operation |
| :--- | :--- | :--- |
| `PE_MODE_IDLE` | `4'b0000` | No operation / reset default |
| `PE_MODE_CWM` | `4'b1000` | Coordinate-Wise Multiplication |
| `PE_MODE_NTT` | `4'b1010` | Number Theoretic Transform (forward) |
| `PE_MODE_INTT` | `4'b1111` | Inverse NTT |
| `PE_MODE_ADDSUB` | `4'b0011` | Modular point-wise Add / Sub |
| `PE_MODE_COMP` | `4'b1100` | Compression (unsupported) |
| `PE_MODE_DECOMP` | `4'b0100` | Decompression (unsupported) |

> [!NOTE]
> The encoding bits are not arbitrary — internal PE MUXes use `ctrl_i[0..3]` directly to select data paths (e.g., `ctrl_i[0]` selects INTT vs NTT operand routing in PE0/PE3). Do not add new mode encodings without reviewing all PE routing tables.

## 4. Memory Subsystem — Primary PAU Port

The primary port is used for NTT/INTT in-place reads and writes, CWM A_hat and e_hat reads, and the final CWM t_hat writeback.

### 4.1 Primary Port — Outputs (PAU → Memory)

| Port | Width | Description |
| :--- | :--- | :--- |
| `pau_req_o` | 1 | Request strobe. High whenever the PAU has a read or write transaction pending. |
| `pau_rd_en_o` | 1 | Read enable. High when the current transaction includes a coefficient read. |
| `pau_rd_poly_id_o` | `log2(NUM_POLYS)` | Polynomial slot to read from. |
| `pau_rd_idx_o` | `4 × 8` | Coefficient indices for the four parallel read lanes [lane3:lane0]. |
| `pau_rd_lane_valid_o` | 4 | Per-lane validity mask for the read request. |
| `pau_wr_en_o` | 4 | Per-lane write enable. High for lanes carrying writeback data. |
| `pau_wr_poly_id_o` | `log2(NUM_POLYS)` | Polynomial slot to write to. |
| `pau_wr_idx_o` | `4 × 8` | Coefficient indices for the four writeback lanes. |
| `pau_wr_data_o` | `4 × 16` | Writeback data. Lower 12 bits of each lane are valid (coefficient); upper 4 bits are zero-padded. |

### 4.2 Primary Port — Inputs (Memory → PAU)

| Port | Width | Description |
| :--- | :--- | :--- |
| `pau_rd_valid_i` | 1 | Read response valid. Asserted exactly 1 cycle after the memory subsystem accepts a primary read request. |
| `pau_rd_poly_id_i` | `log2(NUM_POLYS)` | Polynomial slot echoed by memory on the response beat. |
| `pau_rd_idx_i` | `4 × 8` | Coefficient indices returned by memory. May be in any lane order; the CMI re-orders them. |
| `pau_rd_lane_valid_i` | 4 | Per-lane validity mask for the read response. |
| `pau_rd_data_i` | `4 × 16` | Coefficient data returned by memory. The CMI re-maps response lanes to the requested destination lanes. |
| `pau_stall_i` | 1 | Back-pressure from the memory arbiter. When high, the CMI blocks all new read accepts and drives `ready_o = 0` to the controller, pausing the issue counter. |

## 5. Memory Subsystem — Auxiliary PAU Port

The auxiliary port is **read-only** from the PAU's perspective. It is used to fetch the secondary operand simultaneously with the primary operand, allowing CWM to receive A_hat and s_hat in the same cycle, and ADDSUB to receive operand Y in the same cycle as X.

> [!NOTE]
> `pau_aux_wr_en_o`, `pau_aux_wr_poly_id_o`, `pau_aux_wr_idx_o`, and `pau_aux_wr_data_o` are permanently driven to zero by the CMI. The auxiliary port never writes.

### 5.1 Auxiliary Port — Outputs (PAU → Memory)

| Port | Width | Description |
| :--- | :--- | :--- |
| `pau_aux_req_o` | 1 | Auxiliary request strobe. |
| `pau_aux_rd_en_o` | 1 | Auxiliary read enable. |
| `pau_aux_rd_poly_id_o` | `log2(NUM_POLYS)` | Polynomial slot to read (s for CWM; Y operand for ADDSUB). |
| `pau_aux_rd_idx_o` | `4 × 8` | Coefficient indices for the four auxiliary read lanes. |
| `pau_aux_rd_lane_valid_o` | 4 | Per-lane validity mask. In CWM mode, only lanes 0 and 1 are asserted. |
| `pau_aux_wr_en_o` | 4 | Always `4'b0000`. |
| `pau_aux_wr_poly_id_o` | `log2(NUM_POLYS)` | Always `'0`. |
| `pau_aux_wr_idx_o` | `4 × 8` | Always `'0`. |
| `pau_aux_wr_data_o` | `4 × 16` | Always `'0`. |

### 5.2 Auxiliary Port — Inputs (Memory → PAU)

| Port | Width | Description |
| :--- | :--- | :--- |
| `pau_aux_rd_valid_i` | 1 | Auxiliary read response valid. Asserted 1 cycle after an accepted auxiliary read request. |
| `pau_aux_rd_poly_id_i` | `log2(NUM_POLYS)` | Polynomial slot echoed on the auxiliary response beat. |
| `pau_aux_rd_idx_i` | `4 × 8` | Coefficient indices returned. |
| `pau_aux_rd_lane_valid_i` | 4 | Per-lane validity mask for the auxiliary response. |
| `pau_aux_rd_data_i` | `4 × 16` | Auxiliary coefficient data. |

## 6. Handshaking & Protocol Rules

* **1-cycle read latency model**: The memory subsystem asserts `pau_rd_valid_i` exactly one clock cycle after the PAU's primary read request is accepted (i.e., one cycle after `pau_req_o & pau_rd_en_o & ~pau_stall_i`). The CMI latches the pending request on acceptance and uses the stored indices to re-map the response lanes to the correct destination positions.

* **Lane reordering**: The CMI does **not** assume that `pau_rd_idx_i[n]` corresponds to `pau_rd_idx_o[n]`. It performs an explicit index-matching loop on the response to assign coefficients to the correct lanes, tolerating any permutation from the memory bank.

* **INTT Radix-2 lane swap**: During INTT Pass 0 (radix-2), the controller reads coefficients in interleaved order `{0, 2, 1, 3}` across the four lanes so PE0 and PE2 can each perform a butterfly in parallel. The CMI detects this condition (`is_radix2_i & pass_idx_i == 0`) and re-orders writeback indices and data to restore natural order `{0, 1, 2, 3}` before writing back.

* **Writeback alignment**: The CMI maintains a configurable delay pipeline (up to 11 stages) for writeback indices and lane-valid signals. The controller selects the correct tap via `wb_latency_i`, which varies by mode and pass (range: 2–11 cycles). This ensures the write address arrives at memory in the same cycle as the PE result data.

* **Stall behavior**: `pau_stall_i` pauses the entire read pipeline. The controller's `issue_fire` is gated by `cmi_ready_i = ~pau_stall_i`. The FSM counters do not advance while stalled, so no beats are dropped.

* **Write-only cycles**: Writeback can occur without a concurrent read (e.g., during NTT/INTT drain phases and CWM t_hat writeback). `pau_req_o` will be asserted and `pau_rd_en_o` de-asserted in these cases.

* **CWM drain handshake**: During the drain phase the controller asserts `mac_drain_issue_o`. The row accumulator responds with `mac_drain_accept_o` when it is ready to emit the next pair. The CMI `cwm_drain_issue_i` gating ensures both the CMI and the accumulator agree on each pair.
