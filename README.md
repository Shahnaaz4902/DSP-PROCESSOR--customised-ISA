# DSP Processor — Program-Driven RTL DSP Core

A **program-driven DSP processor** implemented in synthesizable Verilog RTL. The design combines a compact instruction set, scalar/complex/vector arithmetic, accumulator hardware, address-generation units, configurable DSP sequencers, and fixed-point saturation/rounding logic into a single DSP-oriented processor core.

The processor executes one instruction per cycle for ordinary arithmetic/core instructions, while DSP sequence operations use a dedicated multi-cycle sequencer.

---

## Features

- Program-driven DSP processor architecture
- 32-bit instruction word
- 32-bit general-purpose DSP register file
- Dedicated accumulator register file
- Complex and vector arithmetic support
- Multiply-accumulate (MAC) operations
- Saturation, rounding and scaling
- Dedicated address-generation units (AGUs)
- Configurable multi-cycle DSP sequencer
- FIR filtering
- IIR filtering
- Convolution
- Correlation
- Decimation
- Interpolation
- Polyphase processing
- LMS adaptive filtering
- Circular-buffer addressing
- Linear addressing
- Phase-based/polyphase addressing
- CSR-based DSP configuration
- Self-checking RTL testbenches
- Python golden/reference models
- Intel Quartus Prime project

---

# Architecture

```text
                         +------------------+
                         |  Instruction     |
                         |     Memory       |
                         +--------+---------+
                                  |
                                  v
                         +------------------+
                         |    DSP CORE      |
                         |                  |
                         | Instruction      |
                         | Decoder          |
                         | Register File    |
                         | CSR File         |
                         +--------+---------+
                                  |
              +-------------------+-------------------+
              |                   |                   |
              v                   v                   v
      +---------------+   +---------------+   +---------------+
      | DSP Arithmetic|   | Accumulator   |   | DSP Sequencer |
      |     Unit      |   | Register File |   |               |
      +-------+-------+   +---------------+   | FIR / IIR     |
              |                               | CONV / CORR   |
              |                               | DECIM / INTERP|
              |                               | POLY / LMS    |
              |                               +-------+-------+
              |                                       |
              |                               +-------+-------+
              |                               |     AGUs      |
              |                               +-------+-------+
              |                                       |
              v                                       v
       +-------------+                    +----------------------+
       | Sat/Round/  |                    | Coeff / X / Y / D   |
       |    Scale    |                    |      Memories       |
       +-------------+                    +----------------------+
```

The architecture is intentionally divided into:

1. **Core instruction execution**
2. **Arithmetic/accumulator datapath**
3. **DSP sequence engine**
4. **Address-generation hardware**
5. **DSP memories**

---

# Processor Execution Model

The core is currently **single-cycle per ordinary instruction**.

DSP sequence instructions are different: operations such as FIR, IIR, interpolation and LMS require multiple cycles and are executed by the dedicated `DspSequencer`.

```text
Normal instruction:

FETCH → DECODE → EXECUTE → NEXT INSTRUCTION


SEQ instruction:

FETCH → START SEQUENCER
             |
             v
        MULTI-CYCLE FSM
             |
             v
           DONE
             |
             v
       NEXT INSTRUCTION
```

The current design is **not pipelined**. Pipelining the mixture of single-cycle arithmetic instructions and variable-latency DSP sequence operations would require explicit sequencer-busy hazard/stall handling.

---

# Instruction Format

The processor uses a 32-bit instruction word.

The top three bits select the instruction class:

```text
31    29
+-------+
| CLASS |
+-------+
```

## Instruction Classes

| Class | Binary | Operation |
|---|---|---|
| ARITH | `000` | DSP arithmetic |
| SEQ | `001` | Multi-cycle DSP sequence |
| LD | `010` | Load |
| ST | `011` | Store |
| LI | `100` | Load immediate |
| PACK | `101` | Pack two 16-bit values |
| CSRW | `110` | CSR write |
| SPECIAL | `111` | Accumulator read / HALT |

---

# ARITH Instructions

The arithmetic instruction class supports operations decoded by `DspArithCtrl`.

Supported operations include:

```text
CADD
CSUB
VADD
VSUB
CONJ
MAG2
CMUL
VMUL
MAC
MSAC
CMAC
CMSAC
VMAC
VMSAC
```

These operations allow the same arithmetic hardware to support scalar, complex and vector-oriented DSP computations.

The arithmetic path is:

```text
Register File
     |
     v
DspArithCtrl
     |
     v
DspArithUnit
     |
     +----> Result
     |
     +----> Accumulator Term
```

---

# Accumulator Architecture

The processor contains a dedicated accumulator register file.

The accumulator datapath supports:

- Multiple accumulator selections
- Accumulate
- Subtract/accumulate
- Clear
- Complex real/imaginary accumulation
- Saturation
- Rounding
- Scaling

Conceptually:

```text
                +----------------+
                | DspArithUnit  |
                +-------+--------+
                        |
                  term_re / term_im
                        |
                        v
                +---------------+
                | Accumulator   |
                | Register File |
                +-------+-------+
                        |
                        v
                +---------------+
                | Sat/Round/     |
                | Scale          |
                +-------+-------+
                        |
                        v
                     Result
```

---

# DSP Sequencer

`DspSequencer.v` is the main multi-cycle DSP execution engine.

It reuses the arithmetic and accumulator hardware rather than implementing separate multiplier/accumulator datapaths for every DSP algorithm.

The supported sequence modes are:

```text
MODE_FIR
MODE_IIR
MODE_CONV
MODE_CORR
MODE_DECIM
MODE_INTERP
MODE_POLY
MODE_LMS
```

---

# FIR Filtering

The FIR mode performs a multiply-accumulate operation across a configurable number of taps.

Conceptually:

```text
y[n] = Σ h[k] · x[n-k]
```

The sequencer:

1. Walks through the tap count
2. Generates coefficient addresses
3. Generates input-history addresses
4. Multiplies coefficient and sample
5. Accumulates the result
6. Applies saturation/rounding/scaling
7. Produces the output sample

The x-history is maintained using a circular buffer.

---

# IIR Filtering

The IIR mode extends the FIR datapath with feedback-history processing.

Conceptually:

```text
y[n] = feedforward terms + feedback terms
```

The sequencer uses:

- Feedforward coefficient memory
- Input history
- Output history
- Separate feedback coefficient memory
- Independent tap counts
- Independent circular-buffer lengths

The second phase walks through the feedback taps and accumulates the feedback contribution using the same arithmetic/accumulator hardware.

---

# Convolution

The convolution mode operates on two linearly addressed input sequences.

```text
y[n] = x[n] * h[n]
```

The AGUs generate the required coefficient/data addresses while the shared arithmetic unit performs the multiply-accumulate operation.

---

# Correlation

Correlation reuses the same MAC-oriented datapath but uses the appropriate input addressing to generate correlation results.

This demonstrates how the processor can support multiple DSP kernels through programmable addressing rather than requiring a dedicated hardware block for every operation.

---

# Decimation

The DECIM mode continues to perform the filtering/MAC operation for every input sample but only commits an output periodically.

The output commit interval is controlled by:

```text
rate
```

Conceptually:

```text
Input samples:

x0  x1  x2  x3  x4  x5  ...

Filter:

y0  y1  y2  y3  y4  y5  ...

Output:

y0        y2        y4
 |         |         |
 v         v         v
committed every RATE samples
```

This separates the filtering operation from output-rate control.

---

# Interpolation and Polyphase Processing

The processor supports phase-based coefficient addressing for:

```text
INTERP
POLY
```

The coefficient memory is organized into multiple sub-filters.

The `phase` input selects the required sub-filter.

```text
Coefficient Memory

+-------------------+
| Phase 0 taps      |
+-------------------+
| Phase 1 taps      |
+-------------------+
| Phase 2 taps      |
+-------------------+
| ...               |
+-------------------+
```

The input-history window is kept fixed across the phases belonging to the same input sample.

The x-history pointer advances only after the final phase has been processed.

This allows the same MAC datapath to implement polyphase filtering without duplicating multiplier hardware.

---

# LMS Adaptive Filtering

The LMS mode implements adaptive FIR filtering.

The processing sequence includes:

```text
1. FIR output calculation
2. Error calculation
3. Step-size multiplication
4. Coefficient update
```

The error is:

```text
e[n] = d[n] - y[n]
```

The coefficient update is conceptually:

```text
h[k] = h[k] + μ · e[n] · x[n-k]
```

where:

- `d[n]` = desired signal
- `y[n]` = filter output
- `e[n]` = error
- `μ` = LMS step size

The sequencer reuses the shared arithmetic unit for the required subtract and multiply operations.

---

# Address Generation Units

`DspAGU.v` provides programmable address generation for DSP memory streams.

Supported address modes include:

```text
LINEAR
CIRCULAR
PHASE
```

The AGU accepts parameters such as:

```text
base
wptr
buflen
k
phase
stride_tap
stride_phase
```

This allows the same DSP sequencer to support:

- Linear coefficient streams
- Circular sample histories
- Polyphase coefficient banks
- Sliding windows
- Feedback histories

Architecture:

```text
                    +----------------+
base  ------------->|                |
wptr  ------------->|                |
buflen-------------->|     DspAGU     |----> address
k ----------------->|                |
phase--------------->|                |
stride_tap --------->|                |
stride_phase ------->|                |
                    +----------------+
```

---

# Register File

`DspRegFile.v` provides the general-purpose DSP register file used by the program-driven core.

The register file supplies:

```text
rs1
rs2
rd
```

and supports synchronous writes.

The registers are used for:

- Arithmetic operands
- Load/store addresses
- Immediate results
- Packed data
- Accumulator results

---

# CSR File

`DspCsrFile.v` stores configuration parameters for the DSP sequencer.

The configuration registers include:

```text
P0
P1
P2
P3
LC
LC2
BUFLEN
BUFLEN2
MU
RATE
PHASE
```

These parameters control:

- Coefficient bases
- Input/output history bases
- Feedback coefficient base
- Tap counts
- Circular-buffer lengths
- LMS step size
- Decimation/interpolation rate
- Polyphase selection

A typical program therefore configures the sequencer using `CSRW` instructions and then launches a `SEQ` instruction.

---

# Saturation, Rounding and Scaling

`SatRoundScale.v` performs the final conversion of the wider accumulator result into the required output format.

The block supports:

- Arithmetic shifting
- Rounding
- Saturation

This is important for fixed-point DSP because intermediate MAC results can be wider than the final sample width.

```text
Wide Accumulator
       |
       v
  Scale / Shift
       |
       v
    Rounding
       |
       v
  Saturation
       |
       v
  Output Sample
```

---

# Memory Organization

The design contains separate memories for the processor and DSP sequencer.

```text
Instruction Memory
       |
       +---- imem


Data Memory
       |
       +---- dmem


DSP Memories
       |
       +---- coeff_mem
       +---- x_mem
       +---- y_mem
       +---- d_mem
```

The testbenches preload these memories using `$readmemh` files.

---

# Project Structure

```text
dsp_processor/
│
├── rtl/
│   ├── DspCore.v
│   ├── DspArithCtrl.v
│   ├── DspArithUnit.v
│   ├── DspRegFile.v
│   ├── DspCsrFile.v
│   ├── DspSequencer.v
│   ├── DspAGU.v
│   ├── AccRegFile.v
│   ├── SatRoundScale.v
│   │
│   ├── tb_dsp_core.v
│   ├── tb_dsp_arith.v
│   ├── tb_dsp_ctrl.v
│   ├── tb_dsp_agu.v
│   ├── tb_acc_regfile.v
│   ├── tb_satround.v
│   ├── tb_dsp_seq_fir.v
│   ├── tb_dsp_seq_iir.v
│   ├── tb_dsp_seq_conv.v
│   ├── tb_dsp_seq_decim.v
│   ├── tb_dsp_seq_interp.v
│   └── tb_dsp_seq_lms.v
│
├── verify/
│   ├── dsp_arith_golden.py
│   ├── gen_vectors.py
│   ├── gen_core_program.py
│   ├── gen_agu_vectors.py
│   ├── gen_acc_vectors.py
│   ├── gen_ctrl_vectors.py
│   ├── gen_satround_vectors.py
│   ├── gen_seq_fir_vectors.py
│   ├── gen_seq_iir_vectors.py
│   ├── gen_seq_conv_vectors.py
│   ├── gen_seq_decim_vectors.py
│   ├── gen_seq_interp_vectors.py
│   └── gen_seq_lms_vectors.py
│
├── docs/
│   ├── DspProcessor_Architecture.pdf
│   └── build_pdf.py
│
├── DspProcessor.qpf
├── DspProcessor.qsf
└── *.hex / *.txt test vectors
```

---

# Verification

The project contains dedicated testbenches for the major hardware blocks and DSP algorithms.

### Arithmetic Unit

```text
tb_dsp_arith.v
```

Verifies arithmetic operations including scalar/complex/vector functionality.

### Arithmetic Controller

```text
tb_dsp_ctrl.v
```

Verifies ISA-operation decoding and control generation.

### Accumulator

```text
tb_acc_regfile.v
```

Verifies accumulator selection, accumulation, subtraction and clearing.

### AGU

```text
tb_dsp_agu.v
```

Verifies:

- Linear addressing
- Circular addressing
- Phase addressing
- Pointer/stride behavior

### Saturation / Rounding

```text
tb_satround.v
```

Verifies scaling, rounding and saturation behavior.

### DSP Sequencer

Dedicated testbenches verify:

```text
FIR
IIR
Convolution
Decimation
Interpolation
LMS
```

---

# Software Verification

The project includes Python reference/golden-model scripts.

Examples:

```text
verify/dsp_arith_golden.py
verify/gen_vectors.py
verify/gen_seq_fir_vectors.py
verify/gen_seq_iir_vectors.py
verify/gen_seq_conv_vectors.py
verify/gen_seq_decim_vectors.py
verify/gen_seq_interp_vectors.py
verify/gen_seq_lms_vectors.py
```

These scripts generate:

- Input vectors
- Coefficients
- Expected outputs
- Configuration vectors
- Memory initialization files

The generated `.hex` and `.txt` files are then consumed by the Verilog testbenches.

This creates a hardware/software co-verification flow:

```text
Python Golden Model
        |
        v
Expected Vectors
        |
        v
Verilog RTL
        |
        v
Testbench
        |
        v
PASS / FAIL
```

---

# Example DSP Processing Flow

A typical FIR operation follows:

```text
1. Configure CSR registers

   P0      -> coefficient base
   P1      -> input buffer base
   LC      -> number of taps
   BUFLEN  -> circular-buffer length
   RATE    -> output rate if required

2. Store input sample in x_mem

3. Issue SEQ/FIR instruction

4. Sequencer starts

5. AGUs generate coefficient/sample addresses

6. Arithmetic unit performs multiply

7. Accumulator collects MAC result

8. SatRoundScale converts the result

9. Output is written to y_mem

10. Sequencer asserts done
```

---

# Key RTL Concepts Demonstrated

- DSP processor architecture
- Custom instruction-set design
- Program-driven hardware
- Fixed-point arithmetic
- Complex arithmetic
- Vector arithmetic
- Multiply-accumulate datapaths
- Accumulator architecture
- Multi-cycle FSM design
- Address-generation hardware
- Circular buffering
- Polyphase addressing
- Adaptive filtering
- FIR/IIR hardware
- Convolution/correlation
- Decimation/interpolation
- Saturation and rounding
- CSR-controlled accelerators
- Hardware/software co-verification
- Self-checking RTL verification

---

# Current Scope

The current `DspCore` is intentionally a **single-cycle-per-instruction, non-pipelined processor** for the core instruction path.

The DSP `SEQ` operations are multi-cycle and occupy the sequencer until completion.

A natural future architecture improvement would be:

```text
Current:

Instruction
    |
    v
Single-cycle DSP Core
    |
    +---- Multi-cycle Sequencer


Future:

IF → ID → EX → MEM → WB
              |
              +---- DSP Sequencer
                    |
                    +---- busy/stall
                    +---- hazard control
```

This would allow higher instruction throughput while safely integrating variable-latency DSP operations.

---

# Tools Used

- **Verilog HDL**
- **Intel Quartus Prime**
- **Questa / ModelSim**
- **Python**
- Fixed-point DSP reference models

---

# Possible Extensions

- Pipeline the processor core
- Add sequencer-busy hazard/stall handling
- Add instruction/data caches
- Add DMA support
- Add AXI/Avalon interfaces
- Add hardware loop instructions
- Add more DSP-specific instructions
- Add configurable SIMD width
- Add wider/multiple MAC units
- Add compiler/assembler support for the DSP ISA
- Add formal verification
- Add FPGA resource/timing benchmarking
- Map DSP memories to FPGA BRAMs
- Add streaming input/output interfaces

---

## Author

**Shahnaaz Parveen**

M.Tech — Electrical Engineering  
Digital Design | RTL | DSP | Computer Architecture
