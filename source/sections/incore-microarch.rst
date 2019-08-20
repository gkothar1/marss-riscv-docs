===============================
In-Order Core Microarchitecture
===============================

MARSS-RISCV includes a highly configurable cycle accurate model of an in-order scalar core (*In-Order Issue and In-Order Write-back*) based on RISC-V Rocket Core. It can be configured to simulate both 32-bit and 64-bit versions of RISC-V instruction set with 5 and 6 stage pipeline configurations. The pipeline is shared by integer and floating-point instructions and uses data forwarding. 

.. figure:: ../figures/pipeline.*
   :figwidth: 620 px
   :align: center

   Simulated In-order Pipeline

Overview of Pipeline Stages
===========================

The simulated in-order core has the 6 generic pipeline stages: **pcgen**, **fetch**, **decode**, **execute**, **memory**, **commit**. For 5-stage pipeline, pcgen and fetch stages are unified.

PC Generation (pcgen)
----------------------
By default, pcgen increments PC to point to the next instruction in the sequence. Branch resolution overrides the address in pcgen upon misprediction from memory stage. Branch target buffer (BTB) forwards the target address to pcgen for branches.

Fetch
--------------------------
Fetch stage probes the code TLB, BPU and instruction cache in parallel using the
address provided by pcgen stage. On BPU hit, the predicted address is forwarded to pcgen unit. TLB and cache probe functions return latency for cache and TLB lookup and generate DRAM accesses if required (*e.g. TLB miss, LLC miss, victim
eviction*). Stage stalls until the look-up latency is simulated and all generated DRAM
accesses are completed. 

.. note::
   On a TLB miss, DRAM accesses required to read/write page table entries are simulated. Page-fault exception is raised, if page table entry is invalid.

Decode
---------------------------
Decode stages decodes the instruction and if illegal, raises an exception.
Entries in the branch predictor structures are created for branches if not
present. Operands are read from forwarding buses or the register files if
valid. Stage stalls until all the data dependencies are satisfied and the
required FU is available.

.. note::
   For simplicity, SYSTEM major opcode instructions (*csr{r/w}, ecall, sret etc.*) are executed in a **single cycle** due to their complex logic. For this purpose, TinyEMU complex opcode helper functions are used.

Execute
-----------------------------
Execution unit consists of Int-ALU, Int-MUL, Int-DIV, FPU-ALU and FPU-FMA. All the execution units can be configured to run in a pipelined fashion, wherein the latency (in CPU cycles) of each pipelined stage for an execution unit can be configured separately. All the execution units have dedicated data forwarding bus. Last stage for an execution unit puts data on the forwarding bus on which data **remains valid exactly for one cycle**.

.. note::
   Integer ALU performs memory address calculation for loads, stores, atomic instructions, branch target calculation and branch condition evaluation.

Memory
---------------------
For loads, stores and atomic operations, memory stage probes the TLB and data cache in parallel. TLB and cache probe functions return latency for cache and TLB lookup
and also generate DRAM accesses if required (*e.g. TLB miss, LLC miss, victim
eviction*). Data read by loads and atomic instructions is pushed on the forwarding bus for this stage, which **remains valid exactly for one cycle**.
For branches, branch prediction structures are updated and the target address is forwarded to pcgen, followed by flushing of preceding stages on misprediction.
Stage stalls until the look-up latency is simulated and all generated DRAM
accesses are completed. 

.. note::
   On a TLB miss, DRAM accesses required to read/write page table entries are simulated. Page-fault exception is raised, if page table entry is invalid.

Commit
---------------------------
Writes to register files are performed in commit stage.