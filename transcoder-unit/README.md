# Transcoder Unit (`transcoder_unit`)

## Overview

The Transcoder Unit is the **Stateless Formatting Engine and Macro-Sequencing Coprocessor** for the QREM ML-KEM core. It bridges the internal 12-bit polynomial coefficient domain with byte-aligned external interfaces by performing structural formatting (ByteEncode/ByteDecode) and mathematical scaling (Compress/Decompress) as defined by FIPS 203. It operates via a 21-instruction ISA driven by the top-level Core Control Unit, decoupling the full ML-KEM protocol control flow from the hardware execution primitives.

## Directory Map

| File | Contents |
| :--- | :--- |
| `interfaces.md` | All port definitions (control, AXI4-Stream, PolyMem, SeedBank, Hash Snoop), handshaking rules, and backpressure semantics |
| `fsm-states.md` | `tr_fsm` micro-sequencer states, Packer/Unpacker sub-FSM states, transitions, and control outputs |
| `datapath.md` | Gearbox pipeline stages, compress/decompress math, bit packing/slicing, router crossbar modes, microcode ROM opcode table, and critical path |

## Operation Modes

The Transcoder is driven by asserting `ctrl_start` for one cycle with `ctrl_opcode` and `ctrl_sec_level` held stable. `ctrl_done` pulses for exactly one cycle when the complete operation (including any math phase, bypass phase, and hash snooping) is fully finished.

### Security Level Resolution

| `ctrl_sec_level` | Variant | k (vector dim) | du | dv |
| :--- | :--- | :--- | :--- | :--- |
| `2'b00` | ML-KEM-512  | 2 | 10 | 4 |
| `2'b01` | ML-KEM-768  | 3 | 10 | 4 |
| `2'b10` | ML-KEM-1024 | 4 | 11 | 5 |

### Instruction Set (`tr_opcode_t`)

| Opcode | Value | Data Flow | Notes |
| :--- | :--- | :--- | :--- |
| `TR_OP_IDLE` | 5'd0 | — | FSM at rest |
| **KeyGen** | | | |
| `TR_OP_KG_INGEST_D` | 5'd1 | AXI-RX → SeedBank(d) | Bypass only, 4 beats |
| `TR_OP_KG_EXPORT_DK_PKE` | 5'd2 | PolyMem(s) → Encode₁₂ → AXI-TX | k-loop, vector |
| `TR_OP_KG_EXPORT_EK_PKE_1` | 5'd3 | PolyMem(t) → Encode₁₂ → AXI-TX | k-loop, vector |
| `TR_OP_KG_EXPORT_EK_PKE_2` | 5'd4 | SeedBank(ρ) → AXI-TX | Bypass only |
| `TR_OP_KG_EXPORT_HEK` | 5'd5 | SeedBank(H(ek)) → AXI-TX | Bypass only |
| **Encap** | | | |
| `TR_OP_EN_INGEST_M` | 5'd6 | AXI-RX → SeedBank(m) | Bypass only |
| `TR_OP_EN_INGEST_EK_1` | 5'd7 | AXI-RX → Decode₁₂ → PolyMem(t) | k-loop, vector |
| `TR_OP_EN_INGEST_EK_2` | 5'd8 | AXI-RX → SeedBank(ρ) | Bypass only |
| `TR_OP_EN_MSG_DEC` | 5'd9 | SeedBank(m) → Decode₁/Decompress₁ → PolyMem(μ) | Internal crossbar, scalar |
| `TR_OP_EN_EXPORT_CT_1` | 5'd10 | PolyMem(u) → Compress_du/Encode_du → AXI-TX | k-loop, vector, security-level d |
| `TR_OP_EN_EXPORT_CT_2` | 5'd11 | PolyMem(v) → Compress_dv/Encode_dv → AXI-TX | Scalar, security-level d |
| `TR_OP_EN_EXPORT_K` | 5'd12 | SeedBank(K) → AXI-TX | Bypass only |
| **Decap** | | | |
| `TR_OP_DC_INGEST_DK_PKE` | 5'd13 | AXI-RX → Decode₁₂ → PolyMem(s) | k-loop, vector |
| `TR_OP_DC_INGEST_C1` | 5'd14 | AXI-RX → Decode_du/Decompress_du → PolyMem(u') + Hash Snoop | k-loop, vector, snooping |
| `TR_OP_DC_INGEST_C2` | 5'd15 | AXI-RX → Decode_dv/Decompress_dv → PolyMem(v') + Hash Snoop | Scalar, snooping |
| `TR_OP_DC_INGEST_Z` | 5'd16 | AXI-RX → SeedBank(z) | Bypass only |
| `TR_OP_DC_MSG_ENC` | 5'd17 | PolyMem(w) → Compress₁/Encode₁ → SeedBank(m') | Internal crossbar, scalar |
| `TR_OP_DC_EXPORT_K` | 5'd18 | SeedBank(K) → AXI-TX | Bypass only |
| `TR_OP_DC_EXPORT_K_BAR` | 5'd19 | SeedBank(K̄) → AXI-TX | Bypass only |
| `TR_OP_DC_EXPORT_R` | 5'd20 | SeedBank(r) → AXI-TX | Bypass only |

## Integration Notes

> [!IMPORTANT]
> `ctrl_opcode` and `ctrl_sec_level` **must be held stable** from the cycle `ctrl_start` is asserted until `ctrl_done` pulses. The FSM latches them on the rising edge of `ctrl_start` and feeds them to the combinational microcode ROM throughout execution.

> [!IMPORTANT]
> PolyMem access is **mutually exclusive** between the Packer and Unpacker. The FSM enforces this via opcode routing — no opcode triggers both simultaneously. Never issue an opcode that requires both a read and a write operation on the same polynomial in the same `ctrl_start` invocation.

> [!IMPORTANT]
> `TR_OP_DC_INGEST_C1` and `TR_OP_DC_INGEST_C2` multicast incoming ciphertext to both the Unpacker and the **Hash Snoop interface** simultaneously. The router enforces strict AND-gating: `s_axis_tready` is only asserted when both `unpacker_tready` and `hash_snoop_ready_i` are high. The upstream producer must be able to handle backpressure from either consumer.

> [!NOTE]
> `TR_OP_EN_MSG_DEC` and `TR_OP_DC_MSG_ENC` use the **internal SeedBank crossbar** — the math engines read from or write to the SeedBank directly, bypassing the external AXI bus entirely. This is transparent to the host; no AXI transaction occurs for these opcodes.

> [!NOTE]
> The Transcoder does **not** implement the Fujisaki-Okamoto transform, implicit rejection, constant-time comparison (`c == c'`), or random bit generation for `z`. These are handled by the top-level Core Control Unit, which calls Transcoder opcodes as hardware primitives.
