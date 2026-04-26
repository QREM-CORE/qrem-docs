# [Module Name] Interfaces

## 1. Overview
*Provide a 1-2 sentence summary of how this module communicates with the rest of the system.*
**Example:** The Transcoder Unit uses an AXI4-Stream interface to receive 12-bit polynomial coefficients from the Poly-Mem Subsystem and outputs compressed 8-bit byte streams to the Core Control Unit.

## 2. Clocks and Resets
| Port Name | Direction | Domain / Freq | Description |
| :--- | :--- | :--- | :--- |
| `clk_i` | Input | Core Clock (e.g., 220MHz) | Primary synchronous clock. |
| `rst_ni` | Input | Async / Active-Low | Global asynchronous reset, synchronized internally. |

## 3. AXI4-Stream Data Interfaces
*Define the streaming datapaths. If you use a custom protocol instead of AXI-Stream, define it clearly here.*

### 3.1 Input Stream (RX)
| Port Name | Width | Description |
| :--- | :--- | :--- |
| `s_axis_tvalid` | 1 | High when valid data is present on `tdata`. |
| `s_axis_tready` | 1 | Asserted by this module when ready to accept data. |
| `s_axis_tdata` | [Width] | Payload data (e.g., 64-bit polynomial chunk). |
| `s_axis_tlast` | 1 | Indicates the last beat of a 256-coefficient polynomial. |

### 3.2 Output Stream (TX)
| Port Name | Width | Description |
| :--- | :--- | :--- |
| `m_axis_tvalid` | 1 | High when valid data is present on `tdata`. |
| `m_axis_tready` | 1 | Asserted by the downstream consumer. |
| `m_axis_tdata` | [Width] | Payload data. |
| `m_axis_tlast` | 1 | Indicates the end of the packet/polynomial. |

## 4. Control & Status Signals (Sideband)
*List any discrete control signals, flags, or configuration wires that fall outside the main streaming/memory protocols.*

| Port Name | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `start_i` | Input | 1 | 1-cycle pulse to trigger the module operation. |
| `done_o` | Output | 1 | Asserted when the module has completed its active task. |
| `mode_i` | Input | 2 | Execution mode (e.g., `00` = Encapsulate, `01` = Decapsulate). |

## 5. Memory Interfaces (If Applicable)
*If this module acts as a memory master to the Poly-Mem Subsystem, define the read/write ports here.*

| Port Name | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `mem_rd_req_o` | Output | 1 | Read request to the memory arbiter. |
| `mem_rd_addr_o` | Output | `ADDR_W` | Physical row address. |
| `mem_rd_data_i` | Input | 64 | Data returned from the Poly-Mem Subsystem. |

## 6. Handshaking & Protocol Rules
*Define the behavioral rules of the interface. This is critical for preventing deadlocks.*

* **Backpressure:** [e.g., This module supports full `tready` backpressure. It will not drop data if the downstream consumer stalls.]
* **Latency:** [e.g., There is a 2-cycle pipeline latency between `s_axis_tvalid` asserting and the first `m_axis_tvalid` asserting.]
* **Combinational Paths:** [e.g., `s_axis_tready` is registered and does not depend combinationally on `m_axis_tready`.]
