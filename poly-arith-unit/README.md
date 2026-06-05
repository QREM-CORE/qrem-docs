# Polynomial Arithmetic Unit (PAU)

## Overview

The PAU is a mixed-radix-4/2 polynomial arithmetic accelerator implementing the core number-theoretic operations required by FIPS 203 (ML-KEM / Kyber). It is based on the Unified Polynomial Arithmetic Module (UniPAM) architecture from Inha University (IEEE OJCAS 2025). The PAU supports in-place NTT and INTT transformations, point-wise ADDSUB, and Coordinate-Wise Multiplication (CWM) with a scratchpad-backed row accumulator, operating over the ring Z_q[X] / (X^256 + 1) with q = 3329.

## Directory Map

* `interfaces.md` — Port definitions: sideband control, primary and auxiliary memory ports, handshaking and stall protocol.
* `fsm-states.md` — Controller FSM state definitions, transitions, and control signal outputs for all seven states.
* `datapath.md` — PE pipeline topology, modular arithmetic primitives, twiddle ROM, scratchpad accumulator, and critical-path analysis.

## Operation Modes

The PAU is driven by asserting `start_i` for one cycle with the mode and poly-ID fields configured. `done_o` pulses for one cycle when the operation completes.

### 1. NTT / INTT — In-place transformation
- Write the target polynomial to a memory slot.
- Set `op_type_i = PE_MODE_NTT` or `PE_MODE_INTT`, `primary_poly_id_i = <slot>`.
- Pulse `start_i`. The PAU runs four passes (3× Radix-4 + 1× Radix-2 for NTT; 1× Radix-2 + 3× Radix-4 for INTT) in-place. Wait for `done_o`.
- Result overwrites the source slot.

### 2. ADDSUB — Point-wise addition or subtraction
- Write operand A to `primary_poly_id_i`, operand B to `aux_poly_id_i`.
- Set `op_type_i = PE_MODE_ADDSUB`, `is_sub_i = 0` (A + B) or `1` (A - B).
- Pulse `start_i`. Wait for `done_o`.
- Result overwrites operand A's slot.

### 3. CWM — Coordinate-Wise Multiplication (t = A·s + e)
CWM uses **fixed memory slots** regardless of `primary_poly_id_i` / `aux_poly_id_i`:

| Polynomial | Fixed Slot(s) |
| :--- | :--- |
| s (secret key) | 0 … k-1 (`POLY_ID_S0 = 0`) |
| e (error poly) | 4 (`POLY_ID_EI = 4`) |
| A (matrix) | 5 … 5+k-1 (`POLY_ID_A0 = 5`) |
| t (output) | 9 (`POLY_ID_T0 = 9`) |

- Set `op_type_i = PE_MODE_CWM`, `cwm_num_terms_i = k` (e.g., 2, 3, or 4).
- Pulse `start_i`. The PAU accumulates A[0]*s[0] + … + A[k-1]*s[k-1] in the internal scratchpad, then fuses the error polynomial e and writes t to slot 9. Wait for `done_o`.

### 4. COMP / DECOMP
Currently **unsupported**. RTL paths exist but are not exercised.

## Integration Notes

> [!IMPORTANT]
> The PAU requires **two independent memory ports** (primary and auxiliary). Both must be connected to the Poly-Mem Subsystem. The auxiliary port is read-only; CWM uses it to fetch s_hat in the same cycle as A_hat on the primary port.

> [!IMPORTANT]
> `ctrl_i` (op mode) is combinational to all internal PE MUXes. The PAU controller holds the mode steady for an entire job and inserts pipeline-flush drain cycles between passes. Do **not** change `op_type_i` while a job is in progress.

> [!WARNING]
> CWM does **not** observe `primary_poly_id_i` or `aux_poly_id_i`. It always uses the fixed slot map above. Ensure the Poly-Mem Subsystem has s, e, and A populated in those slots before asserting `start_i` for CWM.

> [!NOTE]
> `pau_stall_i` back-pressures the entire PAU read pipeline. The CMI converts this to a `ready_o` signal that gates the controller's `issue_fire`. A stall during a CWM drain will correctly pause the drain handshake.
