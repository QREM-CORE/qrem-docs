# Transcoder Unit — Interfaces

## 1. Overview

The Transcoder Unit communicates with the rest of the system via four strictly decoupled interface groups: a sideband control bus for job dispatch and completion, AXI4-Stream for external host data ingress and egress, a 4-lane polynomial memory interface to the Poly-Mem Subsystem, a 64-bit seed/protocol store interface for 32-byte ML-KEM protocol objects, and a Hash Snoop output that multicasts specific incoming byte streams directly to the Keccak core during Decapsulation.

## 2. Clocks and Resets

| Port | Direction | Description |
| :--- | :--- | :--- |
| `clk` | Input | Positive-edge synchronous clock. All registers in the Transcoder are synchronous to this clock. |
| `rst` | Input | Synchronous, active-high reset. |

## 3. Control Interface (Sideband)

Driven by the top-level Core Control Unit.

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `ctrl_start` | Input | 1 | 1-cycle pulse to launch a new operation. `ctrl_opcode` and `ctrl_sec_level` are latched internally on this edge. Must only be asserted when the FSM is in `ST_IDLE`. |
| `ctrl_done` | Output | 1 | Asserted for exactly 1 cycle when the complete operation has finished (including all math, bypass, and hash phases). The FSM returns to `ST_IDLE` the following cycle. |
| `ctrl_sec_level` | Input | 2 | ML-KEM security level. `2'b00` = ML-KEM-512 (k=2), `2'b01` = ML-KEM-768 (k=3), `2'b10` = ML-KEM-1024 (k=4). Must be stable from `ctrl_start` until `ctrl_done`. |
| `ctrl_opcode` | Input | 5 | Operation opcode (`tr_opcode_t`). Selects one of 21 Transcoder micro-operations. Must be stable from `ctrl_start` until `ctrl_done`. See `README.md` for the full opcode table. |

## 4. AXI4-Stream Interfaces

### 4.1 Input Stream — Host to Transcoder (RX)

Used to ingest external data: seeds, messages, encapsulation keys, ciphertext.

| Port | Width | Description |
| :--- | :--- | :--- |
| `s_axis_tdata` | 64 | Inbound payload. Data is consumed LSB-first; the bit packing order matches the FIPS 203 byte serialization (little-endian coefficient packing). |
| `s_axis_tvalid` | 1 | High when the upstream producer has valid data on `tdata`. |
| `s_axis_tready` | 1 | Asserted by the Transcoder when ready to accept data. Driven combinationally by the active router mode. In dual-destination modes (`MATH_RX_SNOOP`), only asserted when **both** the Unpacker and Hash Snoop core are simultaneously ready. |
| `s_axis_tlast` | 1 | Received but not used internally — the FSM derives packet boundaries from the beat counter and microcode ROM parameters. May be left unconnected by the host. |

### 4.2 Output Stream — Transcoder to Host (TX)

Used to emit encoded/compressed polynomial data: packed keys, ciphertexts, shared secrets.

| Port | Width | Description |
| :--- | :--- | :--- |
| `m_axis_tdata` | 64 | Outbound payload. Data is emitted LSB-first, matching FIPS 203 byte serialization. |
| `m_axis_tvalid` | 1 | High when the Transcoder has valid data to emit. |
| `m_axis_tready` | 1 | Asserted by the downstream consumer. The Transcoder supports full backpressure; valid data is held until ready is asserted. |
| `m_axis_tkeep` | 8 | Lane byte enables. **Permanently driven to `8'hFF`** — all FIPS 203 artifacts are strictly byte-aligned and perfectly divisible by 8 bytes. |
| `m_axis_tlast` | 1 | Asserted on the final beat of the current operation's output. Generated combinationally by the FSM beat tracker, not by the Packer engine. |

## 5. Polynomial Memory Interface

The Transcoder interfaces with the external Poly-Mem Subsystem using a 4-lane wide bus (4×12 bits = 48 bits) capable of reading or writing 4 coefficients per cycle. Access is **mutually exclusive**: either the Packer (read) or the Unpacker (write) holds the bus for a given operation; never both simultaneously.

### 5.1 Global Control

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `poly_req_o` | Output | 1 | Global request strobe. Asserted whenever the Packer or Unpacker has an active transaction. `poly_req_o = packer_poly_req | unpacker_poly_req`. |
| `poly_stall_i` | Input | 1 | Backpressure from the memory arbiter. When high, the active math engine halts all new reads or writes and holds its state. |

### 5.2 Read Channel (Packer → Poly-Mem)

Used during Compress/Encode operations to fetch coefficients from memory.

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `poly_rd_en_o` | Output | 1 | Read enable strobe. Asserted when the Packer has a read request and is not stalled by in-flight bit overflow. |
| `poly_rd_poly_id_o` | Output | `POLY_ID_WIDTH` (5) | Target polynomial slot to read from. Driven from `cfg_poly_base_id + k_counter`. |
| `poly_rd_idx_o` | Output | 4×8 | Coefficient indices for the four parallel read lanes: `{rd_counter, 2'b00}` through `{rd_counter, 2'b11}`. Always reads 4 contiguous coefficients. |
| `poly_rd_lane_valid_o` | Output | 4 | Per-lane validity mask. Always `4'b1111` — the Packer always reads all 4 lanes simultaneously. |

### 5.3 Read Response Channel (Poly-Mem → Packer)

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `poly_rd_valid_i` | Input | 1 | Read response valid. Asserted by memory when data is returning. The Packer uses a 1-cycle registered delay (`poly_rd_valid_q`) to align the response with the compress pipeline output. |
| `poly_rd_poly_id_i` | Input | `POLY_ID_WIDTH` (5) | Polynomial slot echoed by memory on the response beat. |
| `poly_rd_idx_i` | Input | 4×8 | Coefficient indices returned by memory. |
| `poly_rd_lane_valid_i` | Input | 4 | Per-lane validity mask for the read response. |
| `poly_rd_data_i` | Input | 4×`COEFF_WIDTH` (4×12) | Coefficient data returned from memory. Fed directly into the `compress` module. |

### 5.4 Write Channel (Unpacker → Poly-Mem)

Used during Decode/Decompress operations to write coefficients to memory.

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `poly_wr_en_o` | Output | 4 | Per-lane write enable. `4'b1111` when a write fires, `4'b0000` otherwise. Always writes all 4 lanes simultaneously. |
| `poly_wr_poly_id_o` | Output | `POLY_ID_WIDTH` (5) | Target polynomial slot to write into. |
| `poly_wr_idx_o` | Output | 4×8 | Coefficient indices for the four write lanes: `{wr_counter, 2'b00}` through `{wr_counter, 2'b11}`. |
| `poly_wr_data_o` | Output | 4×`COEFF_WIDTH` (4×12) | Decompressed coefficient data to write. Output of the `decompress` module. |

## 6. Seed / Protocol Store Interface

The SeedBank holds 32-byte (256-bit) protocol objects identified by a `seed_id_e` enum. It is accessed in 64-bit beats (`SEED_BEATS = 4` beats per object). The FSM drives addressing and control; the router drives write data.

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `seed_req_o` | Output | 1 | Request strobe. Asserted during `ST_BYPASS` and during internal math phases (`ST_MATH_WAIT` with `cfg_internal_math_en`). |
| `seed_we_o` | Output | 1 | Write enable. High when writing to SeedBank (AXI-RX bypass or Packer→SeedBank internal crossbar). Low for reads. |
| `seed_id_o` | Output | 4 (`seed_id_e`) | Target protocol object identifier. See enum table below. |
| `seed_idx_o` | Output | `$clog2(SEED_BEATS)` (2) | Beat index within the 256-bit object (0–3). Driven from `beat_counter[1:0]`. |
| `seed_wdata_o` | Output | `SEED_W` (64) | Write data. Sourced from the router (`seed_wdata_o` of `tr_router`). |
| `seed_ready_i` | Input | 1 | SeedBank ready for a write transaction. |
| `seed_rvalid_i` | Input | 1 | SeedBank read data valid. Asserted when read data is available on `seed_rdata_i`. |
| `seed_rdata_i` | Input | `SEED_W` (64) | Read data from the SeedBank. Routed via the router to AXI-TX or the Unpacker. |

### 6.1 `seed_id_e` Encoding

| Symbol | Value | Protocol Object |
| :--- | :--- | :--- |
| `SEED_ID_D` | 4'd0 | KeyGen seed d |
| `SEED_ID_RHO` | 4'd1 | Public seed ρ (part of encapsulation key) |
| `SEED_ID_M` | 4'd2 | Message m / recovered message m' |
| `SEED_ID_K` | 4'd3 | Derived shared secret K |
| `SEED_ID_Z` | 4'd4 | Implicit rejection seed z |
| `SEED_ID_K_BAR` | 4'd5 | Implicit rejection secret K̄ |
| `SEED_ID_R` | 4'd6 | Re-encryption randomness r |
| `SEED_ID_HEK` | 4'd7 | Hash of encapsulation key H(ek) |
| `SEED_ID_SIGMA` | 4'd8 | PRF seed σ |
| `SEED_ID_SS` | 4'd9 | Shared secret (intermediate) |
| `SEED_ID_TMP` | 4'd10 | Temporary scratch |

## 7. Hash Snoop Interface

A dedicated AXI4-Stream output that multicasts specific incoming byte streams to the Keccak core. Used exclusively during Decapsulation ciphertext ingestion (`TR_OP_DC_INGEST_C1`, `TR_OP_DC_INGEST_C2`) to compute the Fujisaki-Okamoto re-encryption validation hash without requiring the ciphertext to be re-read from memory.

| Port | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `hash_snoop_data_o` | Output | 64 | Snooped data word. Directly mirrors `s_axis_tdata` in snoop-active modes. |
| `hash_snoop_keep_o` | Output | 8 | Lane byte enables. **Permanently `8'hFF`** — same alignment guarantee as the main AXI-TX bus. |
| `hash_snoop_valid_o` | Output | 1 | Valid strobe. Mirrors `s_axis_tvalid` in snoop-active modes; `0` otherwise. |
| `hash_snoop_ready_i` | Input | 1 | Backpressure from the Keccak core. AND-gated with `unpacker_tready` to form `s_axis_tready` during snoop modes. |
| `hash_snoop_last_o` | Output | 1 | Asserted on the final snooped beat. Generated by the FSM beat tracker. |

## 8. Handshaking & Protocol Rules

* **`ctrl_start` timing**: Assert for exactly 1 cycle. `ctrl_opcode` and `ctrl_sec_level` must be valid on the same rising edge. Do not re-assert `ctrl_start` until `ctrl_done` has been observed.

* **AXI backpressure (single destination)**: In `MATH_TX`, `MATH_RX`, `BYPASS_TX`, and `BYPASS_RX` modes, the Transcoder implements standard AXI4-Stream handshaking. Data transfers occur only on cycles where both `tvalid` and `tready` are asserted simultaneously. No data is dropped.

* **AXI backpressure (dual destination — SNOOP modes)**: In `MATH_TX_SNOOP` and `MATH_RX_SNOOP` modes, `s_axis_tready` is the logical AND of `unpacker_tready` and `hash_snoop_ready_i`. A beat fires only when **both** consumers are simultaneously ready. The upstream producer must tolerate multi-cycle stalls.

* **`tkeep` is always `8'hFF`**: All FIPS 203 formatted artifacts (keys, ciphertexts, messages, seeds) are byte-aligned and sized as exact multiples of 8 bytes. No partial-word beats are ever emitted or expected.

* **`tlast` generation**: The outbound `m_axis_tlast` and `hash_snoop_last_o` signals are generated combinationally by the FSM beat tracker, **not** by the Packer engine. The FSM counts beats using the handshake snoops (`axi_tx_vld_rdy`) and asserts `tlast` on the final beat as computed from the microcode ROM parameters (`cfg_bypass_beats` or `cfg_d_param × k_limit × 4 beats`).

* **SeedBank read latency shielding**: In `BYPASS_TX` mode (SeedBank → AXI-TX), the router is forced to `TR_ROUTER_IDLE` until `seed_rvalid_i` goes high. This prevents the AXI-TX bus from stalling due to SRAM read latency being exposed to the downstream consumer.

* **PolyMem mutual exclusion**: `poly_req_o` is the OR of `packer_poly_req` and `unpacker_poly_req`. Since no single opcode activates both the Packer and Unpacker simultaneously, only one source ever asserts `poly_req_o` at a time. This is a protocol-level guarantee, not a hardware arbiter.

* **Stall behavior**: `poly_stall_i` backpressures the active math engine (Packer or Unpacker). The FSM itself has no stall input — it waits for `packer_done` or `unpacker_done`, which are only asserted after all in-flight memory transactions complete. A stall during a math phase extends the operation duration but does not cause data loss.
