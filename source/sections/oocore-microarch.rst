===================================
Out-of-Order Core Microarchitecture
===================================
MARSS-RISCV includes a highly configurable cycle-accurate model of a generic pipelined out-of-order scalar core. The simulated design does not have separate physical register files. However, it uses the re-order buffer (ROB) slots as the physical registers and separate rename tables for integer and floating-point data. The simulated out-of-order core has seven generic stages: fetch, decode, rename/dispatch, issue, execute, memory, commit. Speculative execution is supported, and instructions are executed on a speculated path as long as required resources are available for dispatch. Figure 2 shows a high-level overview of the simulated out of order core model.

.. figure:: ../figures/oopipeline.*
   :figwidth: 740 px
   :align: center

   Simulated Out-of-Order Pipeline

Overview of Pipeline Stages
===========================

The simulated out-of-order core has the 7 generic pipeline stages: **fetch**, **decode**, **dispatch**, **issue**, **execute**, **memory**, **commit**.

Fetch
--------------------------
Fetch stage probes the instruction TLB, BPU, and instruction cache in parallel using the address provided by the pcgen stage. On BPU hit, the predicted address is sent to the pcgen unit. Fetch stage will stall until the latency for cache-lookup and memory access (if required) is simulated. 

.. note::
   Memory access latency can include latency for reading the cache line containing instruction and reading/writing page table entries on a TLB miss.

Decode
---------------------------
Decode stage decodes the instruction and, if illegal, raises an exception.
Entries in the branch predictor structures are created for branches if not
present. The return address is pushed onto RAS if a function call is detected.

.. note::
   For simplicity, SYSTEM major opcode instructions (*csr{r/w}, ecall, sret etc.*) are executed in a **single cycle** due to their complex logic. For this purpose, TinyEMU complex opcode helper functions are used.

Dispatch
-----------------------------
The dispatch stage performs register renaming and allocates the required entries in IQ, LSQ, and ROB. The stage stalls if any of the required resources are not available. Valid operands are read from the architectural register file.

Issue
---------------------
There is a single global issue queue (IQ) for all the instructions, in which the entries are checked sequentially every cycle for issuing to one of the five execution units.  When an IQ entry is processed, remaining operands are read from the designated ROB slot if they are ready. IQ entry is issued once all the operands are available, and the target execution unit is vacant.

Execute
---------------------------
The execution pipeline stage is sub-divided into five distinct functional units. They are Integer ALU, Integer Multiplier, Integer Divider, Floating-Point ALU, and Floating-Point Fused-Multiply-Add unit. Integer ALU and FPU ALU are strictly non-pipelined, whereas the rest can be configured to run iteratively or in a pipelined fashion. In out of order core design, all of these five functional units strictly operate in parallel. After simulating the execution latency, the FU writes the result produced in the corresponding ROB entry and marks it as ready to commit.

.. note::
      Integer ALU performs memory address calculation for loads, stores, atomic instructions, branch target calculation, and branch condition evaluation.

Memory
--------
A single unified load-store queue (LSQ) handles all the memory traffic of loads, stores, and atomic instructions in the program sequence from where they are issued to (load-store unit) LSU from the head of LSQ. Though loads are speculatively issued, stores and atomics are issued to memory only if they reach ROB top. 

LSU accepts a single memory request from the head of LSQ and probes the data TLB and data cache in parallel. The LSU will stall until the latency for cache-lookup and memory access (if required) is simulated. After simulating the delay, data read by load and atomic instruction is written to the corresponding ROB entry and is marked ready to commit.

.. note::
   Memory access latency can include latency for reading the cache line containing data and reading/writing page table entries on a TLB miss.

Commit
---------
Retirement logic works asynchronously and commits the instructions from the head of Reorder buffer (ROB) when the entry is ready to commit. Results are written to the architectural register file from the corresponding ROB entry. The number of commit ports on ROB is configurable.

Stage-wise activities based on the instruction type
===============================================

This section describes stage by stage pipeline activities for various instruction types from the instant they enter the pipeline 
until they commit.

Operate Instructions
--------------------

Consider the following operate instruction in the logical format: **OPCODE RD, RS1, RS2**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction

* **Dispatch (Rename-Dispatch)**

   * Stall, if ROB or IQ is full

   * Read the rename table to get the latest physical register mappings (or ROB indexes) for ``RS1`` and ``RS2``, say ``PRS1`` and ``PRS2`` respectively

   * If the value of ``PRS1`` or ``PRS2`` is ``-1``, this indicates that ``RS1`` or ``RS2`` can be safely read from the architectural register file

   * Update ``RD`` mapping in the rename table to ``PRD`` (or the ROB index of the current instruction), and save old mapping for the destination ``RD`` as ``PREV_PRD``

   * After renaming, the instruction becomes: ``OPCODE PRD, PRS1, PRS2``

* **Issue**

   * If the value of ``PRS1`` or ``PRS2`` is not ``-1``, then the values of ``RS1`` or ``RS2`` are read from ROB slots ``PRS1`` or ``PRS2`` respectively if they are ready

   * If the required execution unit is free, issue the instruction to the appropriate execution unit and remove IQ entry
   
   * Instruction is not issued until all the source operands are read, and the required execution unit is available

* **Execute**

   * Calculate the result, write it to the corresponding ROB entry and mark the ROB entry as ready to commit

* **Commit**

   * Once this instruction comes to ROB top and no exception has occurred, and entry is ready to commit, write the results from ROB entry to the architectural register file

   * Update the rename table mapping for ``RD`` to ``-1``, which indicates that ``RD`` can now be read directly from the architectural register file

   * Deallocate the ROB entry


Loads
-----

Consider the following load instruction in the logical format: **LOAD RD, RS1, IMM**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction
  
* **Dispatch (Rename-Dispatch)**

   * Stall, if ROB, IQ or LSQ is full

   * Read the rename table to get the latest physical register mappings (or ROB indexes) for ``RS1``, say ``PRS1``

   * If the value of ``PRS1`` is ``-1``, this indicates that ``RS1`` can be safely read from the architectural register file

   * Update ``RD`` mapping in the rename table to ``PRD`` (or the ROB index of the current instruction), and save old mapping for the destination ``RD`` as ``PREV_PRD``

   * After renaming, the instruction becomes: ``LOAD PRD, PRS1, IMM``

* **Issue**

   * If the value of ``PRS1`` is not ``-1``, then the value of ``RS1`` is read from ROB slot ``PRS1``, if ready

   * If the ``Int-ALU`` is free, issue the instruction to ``Int-ALU`` and remove IQ entry
   
   * Instruction is not issued until all the source operands are read, and the ``Int-ALU`` is available

* **Execute**

   * Calculate the memory address, write it to the LSQ entry and mark it as ready

* **Memory**

   * Once the LSQ entry for this load reaches to LSQ top and is valid, issue the load to memory

   * Remove LSQ entry after memory access completes and write the result to corresponding ROB entry and mark the ROB entry as ready to commit

* **Commit**

   * Once this instruction comes to ROB top and no exception has occurred, and entry is ready to commit, write the results from ROB entry to the architectural register file

   * Update the rename table mapping for ``RD`` to ``-1``, which indicates that ``RD`` can now be read directly from the architectural register file

   * Deallocate the ROB entry

Stores
------

Consider the following store instruction in the logical format: **STORE RS1, RS2, IMM**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction
  
* **Dispatch (Rename-Dispatch)**

   * Stall, if ROB, LSQ or IQ is full

   * Read the rename table to get the latest physical register mappings (or ROB indexes) for ``RS1`` and ``RS2``, say ``PRS1`` and ``PRS2`` respectively

   * If the value of ``PRS1`` or ``PRS2`` is ``-1``, this indicates that ``RS1`` or ``RS2`` can be safely read from the architectural register file

   * After renaming, the instruction becomes ``STORE PRS1, PRS2, IMM``

* **Issue**

   * If the value of ``PRS1`` or ``PRS2`` is not ``-1``, then the values of ``RS1`` or ``RS2`` are read from ROB slots ``PRS1`` or ``PRS2`` respectively if they are ready

   * If the ``Int-ALU`` is free, issue the instruction to ``Int-ALU`` and remove IQ entry
   
   * Instruction is not issued until all the source operands are read, and the ``Int-ALU`` is available

* **Execute**

   * Calculate the memory address, write it to the LSQ entry, mark it as valid

* **Commit**

   * Once this store comes to ROB top and LSQ top (LSQ entry must be valid), the store is issued to the memory

   * Deallocate the ROB and LSQ entry once the store completes its memory access

Atomics
--------
Atomic instructions are handled similarly to the store instructions are dispatched only when all the prior instructions in the ROB are committed (ROB is empty). The memory access initiates once the ROB entry for atomics reaches the head of the ROB and LSQ.

Branches
--------

Consider the following branch instruction in the logical format: **BRANCH RS1, RS2, IMM**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction

* **Dispatch (Rename-Dispatch)**

   * Stall, if ROB or IQ is full

   * Read the rename table to get the latest physical register mappings (or ROB indexes) for ``RS1`` and ``RS2``, say ``PRS1`` and ``PRS2`` respectively

   * If the value of ``PRS1`` or ``PRS2`` is ``-1``, this indicates that ``RS1`` or ``RS2`` can be safely read from the architectural register file

   * After renaming, the instruction becomes: ``BRANCH PRS1, PRS2, IMM``


* **Issue**

   * If the value of ``PRS1`` or ``PRS2`` is not ``-1``, then the values of ``RS1`` or ``RS2`` are read from ROB slots ``PRS1`` or ``PRS2`` respectively if they are ready

   * If the ``Int-ALU`` is free, issue the instruction to ``Int-ALU`` and remove IQ entry
   
   * Instruction is not issued until all the source operands are read, and the ``Int-ALU`` is available

* **Execute**

   * Evaluate the branch condition and calculate the target address

   * On a misprediction,

      * Flush fetch, decode and dispatch stages

      * Set the fetch PC to the target address

      * Flush all the instructions from ROB, LSQ, LSU, and IQ on the miss speculated path, followed by this branch

      * Revert all the rename table mappings, to the point of the dispatch of this branch instruction

   * Mark ROB entry of the branch as ready to commit

* **Commit**

   * Once this branch comes to ROB top and is ready to commit, deallocate the ROB entry