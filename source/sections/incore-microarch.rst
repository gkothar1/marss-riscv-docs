===============================
In-Order Core Microarchitecture
===============================

MARSS-RISCV includes a highly configurable cycle-level model of a generic pipelined in-order scalar core. It consists of six pipeline stages: pcgen, fetch, decode, execute, memory, commit. For 5-stage pipeline configuration, pcgen and fetch unit operate as a single stage. The pipeline is shared by the integer and floating-point instructions and supports data forwarding (bypassing)—the figure below a high-level overview of the simulated in-order pipeline.

.. figure:: ../figures/pipeline.*
   :figwidth: 720 px
   :align: center

   Simulated In-order Pipeline

Overview of Pipeline Stages
===========================

The simulated in-order core has the 6 generic pipeline stages: **pcgen**, **fetch**, **decode**, **execute**, **memory**, **commit**. For 5-stage pipeline, pcgen and fetch stages are unified.

PC Generator (pcgen)
----------------------
By default, pcgen increments PC to point to the next instruction in the sequence. Branch resolution overrides the address in pcgen upon misprediction signal from the memory stage. Branch target buffer (BTB) delivers the target address to pcgen for branches on a BPU hit.

Fetch
--------------------------
The fetch stage probes the instruction TLB, BPU, and instruction cache in parallel using the pcgen stage generated address. On BPU hit, the predicted address is sent to the pcgen unit. The fetch stage will stall until the latency for cache-lookup and memory access (if required) is simulated.

.. note::
   Memory access latency can include latency for reading the cache line containing instruction and hardware page-table walk if required.

Decode
---------------------------
The decode stage decodes the instruction and, if illegal, raises an exception. The return address is pushed onto RAS if a function call is detected. Operands are read from forwarding buses or the register files if
valid. Stage stalls until all the data dependencies (RAW and WAW) are satisfied and the required (functional unit) FU is free. 

.. note::
   For simplicity, SYSTEM major opcode instructions (*csr{r/w}, ecall, sret, etc.) are handled as software exceptions due to their complex logic. For this purpose, TinyEMU complex opcode helper functions are used.

Execute
-----------------------------
The execution pipeline stage is sub-divided into five distinct functional units. They are Integer ALU, Integer Multiplier, Integer Divider, Floating-Point ALU, and Floating-Point Fused-Multiply-Add unit. FPU-ALU is strictly non-pipelined, whereas the rest can be configured to run iteratively or in a pipelined fashion. All the execution units have dedicated data forwarding buses. After simulating the execution latency, the FU writes the result on the forwarding bus on which it **remains valid exactly for one cycle**. 

.. note::
   Integer ALU performs memory address calculation for loads, stores, atomic instructions, branch target calculation, and branch condition evaluation.

Memory
---------------------
Non-memory instructions (arithmetic and branches) pass through the memory stage in a single cycle and push their results, which were calculated during execution on the memory stage's forwarding bus. 

For loads, stores, and atomic operations, the memory stage probes the data TLB and data cache in parallel. The stage will stall until the latency for cache-lookup and memory access (if required) is simulated. After simulating the delay, data read by load and atomic instruction is written on the dedicated forwarding bus for the memory stage, which **remains valid exactly for one cycle**. 

The entries for branches in the branch predictor structures are created for branches if not present and updated with the branch's new outcome. The target address is forwarded to pcgen, followed by a flushing of preceding stages on misprediction. Entries 

.. note::
   Memory access latency can include latency for reading/writing the cache line containing data and hardware page-table walk if required.

Commit
---------------------------
The results are written to the register files (Int and FP) in the commit stage.