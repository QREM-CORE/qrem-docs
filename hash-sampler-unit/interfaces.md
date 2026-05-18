# Hash Sampler Unit (HSU) Interfaces

## 1. Overview
The Hash Sampler Unit (HSU) orchestrates cryptographic hashing and polynomial sampling. It communicates via highly specialized memory interfaces (Poly Memory Writer, Poly Memory Reader, and Seed Memory Port) rather than generic AXI-Stream for its main datapaths, minimizing latency. It also features a Raw AXI4-Stream sink for direct Keccak bypass operations.

## 2. Clocks and Resets
| Port Name | Direction | Domain / Freq | Description |
| :--- | :--- | :--- | :--- |
| `clk` | Input | Core Clock | Primary synchronous clock. |
| `rst` | Input | Async / Active-High | Global asynchronous reset, synchronized internally. |

## 3. Control & Status Signals (Sideband)
| Port Name | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `start_i` | Input | 1 | 1-cycle pulse to start a new hashing/sampling operation. |
| `hsu_mode_i` | Input | Enum | Defines operation mode (e.g., NTT, CBD, SHA3, SHAKE256). |
| `is_eta3_i` | Input | 1 | Configures CBD sampler (`1` = ML-KEM-768/1024, `0` = ML-KEM-512). |
| `xof_len_i` | Input | `XOF_LEN_WIDTH`| Total output bytes for SHAKE modes. |
| `input_sel_i` | Input | 2 | Selects Keccak source (0 = Seed Mem, 1 = Poly Mem Reader, 2 = Raw AXI-S). |
| `absorb_poly_i` | Input | 1 | Pulse to absorb one polynomial via packer. Wait for `packer_done_o`. |
| `absorb_last_i` | Input | 1 | High on the LAST segment before Keccak squeezes. Gates `t_last`. |
| `hsu_done_o` | Output | 1 | Sticky completion signal. Latches high when operation finishes. |
| `packer_done_o` | Output | 1 | Asserts when Poly Mem Reader packer finishes gearbox drain. |
| Metadata Inputs | Input | Various | `poly_id_i`, `seed_id_i`, `row_i`, `col_i`, `cbd_n_i` |

## 4. Memory Interfaces

### 4.1 Poly Memory Writer (Outputs from Samplers)
Used during `MODE_SAMPLE_NTT` and `MODE_SAMPLE_CBD` to write generated coefficients.
| Port Name | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `hsu_req_o` | Output | 1 | Active when making a memory request. |
| `hsu_wr_en_o` | Output | 4 | Byte/Lane write enables. |
| `hsu_wr_poly_id_o` | Output | `log2(NUM_POLYS)` | Target polynomial ID. |
| `hsu_wr_idx_o` | Output | 4x `log2(NCOEFF)`| Indices for 4 simultaneous coefficient writes. |
| `hsu_wr_data_o` | Output | 4x `COEFF_W` | 4 generated coefficients. |

### 4.2 Poly Memory Reader (Input to Packer)
Used during `MODE_ABSORB_POLY` to read coefficients for hashing (e.g., H(t_hat)).
| Port Name | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `hsu_rd_poly_id_o` | Output | `log2(NUM_POLYS)` | Target polynomial ID to read. |
| `hsu_rd_idx_o` | Output | 4x `log2(NCOEFF)`| Indices to read. |
| `hsu_rd_lane_valid_o`| Output | 4 | Valid lanes requested. |
| `hsu_rd_valid_i` | Input | 1 | Indicates `hsu_rd_data_i` is valid. |
| `hsu_rd_data_i` | Input | 4x `COEFF_W` | Returned coefficient data. |

### 4.3 Seed Memory Port
| Port Name | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `hsu_seed_req_o` | Output | 1 | Request to read/write seed memory. |
| `hsu_seed_we_o` | Output | 1 | Write enable (1=Write, 0=Read). |
| `hsu_seed_idx_o` | Output | `log2(SEED_BEATS)`| Beat index for 32B/64B seeds. |
| `hsu_seed_wdata_o` | Output | `SEED_W` | Data to write (from Keccak bypass). |
| `hsu_seed_rdata_i` | Input | `SEED_W` | Data read (for Keccak absorption). |
| `hsu_seed_ready_i` | Input | 1 | Backpressure from seed memory. |
| `hsu_seed_rvalid_i`| Input | 1 | Read data valid flag. |

## 5. Raw AXI4-Stream Input (Bypass)
Used exclusively when `input_sel_i = 2` for direct Keccak absorption from external sources.
| Port Name | Width | Description |
| :--- | :--- | :--- |
| `axis_t_valid_i` | 1 | Valid data present. |
| `axis_t_ready_o` | 1 | HSU ready to accept data (driven by `keccak_t_ready_o`). |
| `axis_t_data_i` | `SEED_W` | Payload data. |
| `axis_t_keep_i` | `SEED_W/8` | Byte enables. |
| `axis_t_last_i` | 1 | End of stream. |

## 6. Handshaking & Protocol Rules
* **Seed Backpressure:** Operations writing to Seed Memory respect `hsu_seed_ready_i`. Seed injection pauses until ready.
* **Packer Read Throttling:** `coeff_to_axis_packer` uses a `rd_pending_q` register to ensure memory read requests (`hsu_req_o` / `hsu_rd_en_o`) are issued exactly once per index set, preventing pipelined memory read duplication.
* **Multi-Phase Absorption:** The controller must wait for `packer_done_o` to assert before pulsing `absorb_poly_i` again or changing `input_sel_i`. `absorb_last_i` must only be asserted on the final absorption chunk.
