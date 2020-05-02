===================================
Out-of-Order Core Microarchitecture
===================================

MARSS-RISCV includes a highly configurable cycle accurate model of an out-of-order scalar core. It can be configured to simulate both 32-bit and 64-bit versions of RISC-V instruction set. The simulated design does not have distinct physical register files, but the **re-order buffer slots are used as physical registers**.

.. figure:: ../figures/oopipeline.*
   :figwidth: 740 px
   :align: center

   Simulated Out-of-Order Pipeline

Overview of Pipeline Stages
===========================

The simulated out-of-order core has the 7 generic pipeline stages: **fetch**, **decode**, **dispatch**, **issue**, **execute**, **memory**, **commit**.

Fetch
--------------------------
Fetch stage probes the code TLB, BPU and instruction cache in parallel. On BPU hit, the predicted address is set as the next fetch PC. TLB and cache probe functions return latency for cache and TLB lookup and generate DRAM accesses if required (*e.g. TLB miss, LLC miss, victim
eviction*). Stage stalls until the look-up latency is simulated and all generated DRAM
accesses are completed. 

.. note::
   On a TLB miss, DRAM accesses required to read/write page table entries are simulated. Page-fault exception is raised, if page table entry is invalid.

Decode
---------------------------
Decode stages decodes the instruction and if illegal, raises an exception. Entries in the branch predictor structures are created for branches if not present.

.. note::
   For simplicity, SYSTEM major opcode instructions (*csr{r/w}, ecall, sret etc.*) are executed in a **single cycle** due to their complex logic. For this purpose, TinyEMU complex opcode helper functions are used.

Dispatch
-----------------------------
Dispatch stage performs register renaming and allocates the required entries in IQ, LSQ and ROB. 
The stage stalls if the any of the required resources are not available. Valid operands are read from the architectural register file.

Issue
---------------------
In Issue stage, remaining operands are read from the indicated ROB slot if they are avaliable or from the architectural register file. The stage stalls if the any of the required resources are not available. There is a single global issue queue for all the instructions, in which the entries are checked sequentially every cycle for issuing.

Execute
---------------------------
Execution unit consists of Int-ALU, Int-MUL, Int-DIV, FPU-ALU and FPU-FMA. All the execution units can be configured to run in a pipelined fashion, where the latency (in CPU cycles) of each pipelined stage for an execution unit can be configured separately. Last stage for an execution unit writes the result produced in the corresponding ROB entry and marks it as ready to commit. 

.. note::
      Integer ALU performs memory address calculation for loads, stores, atomic instructions, branch target calculation and branch condition evaluation.

Memory
------
All the data memory accesses are performed by the Load Store Unit (also know as the memory stage) through the Load Store Queue (LSQ). 
For loads, stores and atomic operations, LSU probes the TLB and data cache in parallel. 
TLB and cache probe functions return latency for cache and TLB lookup and also generate DRAM accesses if required 
(e.g. TLB miss, LLC miss, victim eviction). 
LSU stalls until the look-up latency is simulated and all generated DRAM accesses are completed. Data read by loads and atomic instructions is written in the corresponding ROB entry and is marked ready to commit.

.. note::
   Current version supports a simple unified LSQ design, in which loads and stores are processed sequentially, in program order.
.. note::
   On a TLB miss, DRAM accesses required to read/write page table entries are simulated. Page-fault exception is raised, if page table entry is invalid.

Commit
------
Retirement logic commits the instructions from the head of ROB and also updates the rename tables. Results are written to the architectural register file from the corresponding ROB entry. Number of commit ports on ROB can be configured.

Stage-wise activities based on instruction type
===============================================

This section describes stage by stage pipeline activities for various instruction types from moment they enter the pipeline 
until they are committed.

Operate Instructions
--------------------

Consider the following operate instruction in logical format: **OPCODE RD, RS1, RS2**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction

* **Dispatch (Rename-Dispatch)**

   * Stall, if ROB or IQ is full

   * Read the rename table to get the latest physical register mappings (or ROB indexes) for ``RS1`` and ``RS2``, say ``PRS1`` and ``PRS2`` respectively

   * If the value of ``PRS1`` or ``PRS2`` is ``-1``, this indicates that ``RS1`` or ``RS2`` can be safely read from architectural register file

   * Update ``RD`` mapping in the rename table to ``PRD`` (or the ROB index of the current instruction), and save old mapping for the destination ``RD`` as ``PREV_PRD``

   * After renaming, the instruction becomes: ``OPCODE PRD, PRS1, PRS2``

* **Issue**

   * If the value of ``PRS1`` or ``PRS2`` is not ``-1``, then the values of ``RS1`` or ``RS2`` are read from ROB slots ``PRS1`` or ``PRS2`` respectively, if they are ready

   * If the required execution unit is free, issue the instruction to appropriate execution unit and remove IQ entry
   
   * Instruction is not issued until all the source operands are read and the required execution unit is available

* **Execute**

   * Calculate the result, write it to the corresponding ROB entry and mark the ROB entry as ready to commit

* **Commit**

   * Once this instruction comes to ROB top and no exception has occurred and entry is ready to commit, write the results from ROB entry to the architectural register file

   * Update the rename table mapping for ``RD`` to ``-1``, this indicates that ``RD`` can be now read directly from the architectural register file

   * Deallocate the ROB entry


Loads
-----

Consider the following load instruction in logical format: **LOAD RD, RS1, IMM**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction
  
* **Dispatch (Rename-Dispatch)**

   * Stall, if ROB, IQ or LSQ is full

   * Read the rename table to get the latest physical register mappings (or ROB indexes) for ``RS1``, say ``PRS1``

   * If the value of ``PRS1`` is ``-1``, this indicates that ``RS1`` can be safely read from architectural register file

   * Update ``RD`` mapping in the rename table to ``PRD`` (or the ROB index of the current instruction), and save old mapping for the destination ``RD`` as ``PREV_PRD``

   * After renaming, the instruction becomes: ``LOAD PRD, PRS1, IMM``

* **Issue**

   * If the value of ``PRS1`` is not ``-1``, then the value of ``RS1`` is read from ROB slot ``PRS1``, if ready

   * If the ``Int-ALU`` is free, issue the instruction to ``Int-ALU`` and remove IQ entry
   
   * Instruction is not issued until all the source operands are read and the ``Int-ALU`` is available

* **Execute**

   * Calculate the memory address and write it to the LSQ entry and the LSQ entry is marked as valid

* **Memory**

   * Once the LSQ entry for this load reaches to LSQ top and is valid, issue the load to memory

   * Remove LSQ entry after memory access completes and write the result to corresponding ROB entry and mark the ROB entry as ready to commit

* **Commit**

   * Once this instruction comes to ROB top and no exception has occurred and entry is ready to commit, write the results from ROB entry to the architectural register file

   * Update the rename table mapping for ``RD`` to ``-1``, this indicates that ``RD`` can be now read directly from the architectural register file

   * Deallocate the ROB entry

Stores
------

Consider the following store instruction in logical format: **STORE RS1, RS2, IMM**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction
  
* **Dispatch (Rename-Dispatch)**

   * Stall, if ROB, LSQ or IQ is full

   * Read the rename table to get the latest physical register mappings (or ROB indexes) for ``RS1`` and ``RS2``, say ``PRS1`` and ``PRS2`` respectively

   * If the value of ``PRS1`` or ``PRS2`` is ``-1``, this indicates that ``RS1`` or ``RS2`` can be safely read from architectural register file

   * After renaming, the instruction becomes: ``STORE PRS1, PRS2, IMM``

* **Issue**

   * If the value of ``PRS1`` or ``PRS2`` is not ``-1``, then the values of ``RS1`` or ``RS2`` are read from ROB slots ``PRS1`` or ``PRS2`` respectively, if they are ready

   * If the ``Int-ALU`` is free, issue the instruction to ``Int-ALU`` and remove IQ entry
   
   * Instruction is not issued until all the source operands are read and the ``Int-ALU`` is available

* **Execute**

   * Calculate the memory address and write it to the LSQ entry and the LSQ entry is marked as valid

* **Commit**

   * Once this store comes to ROB top and LSQ top (LSQ entry must be valid), the store is issued to the memory

   * Deallocate the ROB and LSQ entry once the store completes its memory access

Atomics
-------
Atomic instructions are handled similar to the store instructions. They are dispatched only when all the prior instructions in the ROB have been committed (or only if the ROB is empty) and the memory access is initiated once the ROB entry for atomics reaches the head of the ROB.

Branches
--------

Consider the following branch instruction in logical format: **BRANCH RS1, RS2, IMM**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction

* **Dispatch (Rename-Dispatch)**

   * Stall, if ROB or IQ is full

   * Read the rename table to get the latest physical register mappings (or ROB indexes) for ``RS1`` and ``RS2``, say ``PRS1`` and ``PRS2`` respectively

   * If the value of ``PRS1`` or ``PRS2`` is ``-1``, this indicates that ``RS1`` or ``RS2`` can be safely read from architectural register file

   * After renaming, the instruction becomes: ``BRANCH PRS1, PRS2, IMM``


* **Issue**

   * If the value of ``PRS1`` or ``PRS2`` is not ``-1``, then the values of ``RS1`` or ``RS2`` are read from ROB slots ``PRS1`` or ``PRS2`` respectively, if they are ready

   * If the ``Int-ALU`` is free, issue the instruction to ``Int-ALU`` and remove IQ entry
   
   * Instruction is not issued until all the source operands are read and the ``Int-ALU`` is available

* **Execute**

   * Evaluate the branch condition and calculate the target address

   * On a misprediction,

      * Flush fetch, decode and dispatch stages

      * Set the fetch PC to the target address

      * Flush all the instructions from ROB, LSQ, LSU and IQ on the mis-speculated path, which were followed by this branch

      * Revert all the rename table mappings, to the point of the dispatch of this branch instruction

   * Mark ROB entry of the branch as ready to commit

* **Commit**

   * Once this branch comes to ROB top and is ready to commit, deallocate the ROB entry