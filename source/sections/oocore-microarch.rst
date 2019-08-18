===================================
Out-of-Order Core Microarchitecture
===================================

MARSS-RISCV includes a highly configurable cycle accurate model of an out-of-order scalar core. It can be configured to simulate both 32-bit and 64-bit versions of RISC-V instruction set. 
The simulated out-of-order core is roughly based on RISC-V BOOM processor and follows a **unified physical register file design**.

.. figure:: ../figures/oopipeline.*
   :figwidth: 620 px
   :align: center

   Simulated Out-of-Order Pipeline

Overview of Pipeline Stages
===========================

The simulated out-of-order core has the 8 generic pipeline stages: **fetch**, **decode**, **dispatch**, **issue**, **execute**, **memory**, **writeback**, **commit**.

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
Decode stages decodes the instruction and if illegal, raises an exception. Entries in the branch predictor structures are created for branches if not present.

.. note::
   For simplicity, SYSTEM major opcode instructions (*csr{r/w}, ecall, sret etc.*) are executed in a **single cycle** due to their complex logic. For this purpose, RISCVEMU complex opcode helper functions are used.

Dispatch
-----------------------------
Dispatch stage performs register renaming and allocation of a free physical register to the destination register. Also, the required entries are created in IQ, LSQ and ROB. 
The stage stalls if the any of the required resources are not available.

Issue
---------------------
In Issue stage, sources operands are read from the physical register file or the data forwarding buses and the instruction is then issued to the appropriate execution unit.
The stage stalls if the any of the required resources are not available. Three issues queues are present with configurable number of issue ports: for integer instructions, floating-point instructions and memory access instructions.

.. note::
   Current version has unordered instruction issue logic, meaning issue queue entries are checked sequentially for issuing. Age ordered issue logic will be implemented in the future versions.

Execute
---------------------------
Execution unit consists of Int-ALU, Int-MUL, Int-DIV, FPU-ALU and FPU-FMA. All the execution units can be configured to run in a pipelined fashion, wherein the latency (in CPU cycles) of each pipelined stage for an execution unit can be configured separately. All the execution units have dedicated data forwarding bus. Last stage for an execution unit puts data on the forwarding bus on which data **remains valid exactly for one cycle**.

.. note::
      Integer ALU performs memory address calculation for loads, stores, atomic instructions, branch target calculation and branch condition evaluation.

Memory
------
All the data memory accesses are performed by the Load Store Unit (also know as the memory stage) through the Load Store Queue (LSQ). 
For loads, stores and atomic operations, LSU probes the TLB and data cache in parallel. 
TLB and cache probe functions return latency for cache and TLB lookup and also generate DRAM accesses if required 
(e.g. TLB miss, LLC miss, victim eviction). 
LSU stalls until the look-up latency is simulated and all generated DRAM accesses are completed. Data read by loads and atomic instructions is forwarded to the waiting instructions in the issue queues.

.. note::
   Current version supports a simple unified LSQ design, in which loads and stores are processed sequentially, in program order. Support for processing loads and store out of program order will be added in future versions.
   
.. note::
   On a TLB miss, DRAM accesses required to read/write page table entries are simulated. Page-fault exception is raised, if page table entry is invalid.

Write-back
----------
Writes to physical register files are performed in write-back stage. There are two distinct physical register files, one for the integer data and one for the floating point data. Number of write ports to each of those can be configured.

Commit
------
Instructions are committed from head of the ROB and retirement (commit) rename tables are updated here. Number of commit ports on ROB can be configured.

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

   * Read the front-end rename table to get the latest physical register mappings for ``RS1`` and ``RS2``, say ``PRS1`` and ``PRS2`` respectively

   * Allocate a new free physical destination for ``RD``, say ``PRD``

   * Update ``RD`` mapping in the front-end rename table to ``PRD``, and save old mapping for the destination ``RD`` as ``PREV_PRD``. After renaming, the instruction becomes: ``OPCODE PRD, PRS1, PRS2``

   * Allocate entry for the instruction in ROB and the corresponding IQ (based on instruction type: Int, FP or Mem)
   
   * The stage stalls if any of the required resources are not available

* **Issue**

   * Read operands from physical register file if valid, or from data-bypass network

   * If the required execution unit is free, issue the instruction to appropriate execution unit and remove IQ entry
   
   * Instruction is not issued until all the source operands are read and the required execution unit is available

* **Execute**

   * Calculate the result and broadcast it across data-bypass network for the given execution unit

* **Write-back**

   * Write the result to physical register file

   * Mark the corresponding ROB entry as ready to commit

* **Commit**

   * Once this instruction comes to ROB top and no exception has occurred, update the commit (retirement) rename table

   * Add the ``PREV_PRD`` back to the free list of physical registers
   
   * Deallocate the ROB entry


Loads
-----

Consider the following load instruction in logical format: **LOAD RD, RS1, IMM**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction
  
* **Dispatch (Rename-Dispatch)**

   * Read the front-end rename table to get the latest physical register mappings for ``RS1``, say ``PRS1``

   * Allocate a new free physical destination for ``RD``, say ``PRD``
  
   * Update ``RD`` mapping in the front-end rename table to ``PRD``, and save old mapping for the destination ``RD`` as ``PREV_PRD``. After renaming, the instruction becomes: ``LOAD PRD, PRS1, IMM``
     
   * Allocate entry for the instruction in ROB, LSQ and the corresponding IQ (Mem)
   
   * The stage stalls if any of the required resources are not available

* **Issue**

   * Read operands from physical register file if valid, or from data-bypass network

   * If the ``Int-ALU`` is free, issue the instruction to ``Int-ALU`` and remove IQ entry
   
   * Instruction is not issued until all the source operands are read and the ``Int-ALU`` is available

* **Execute**

   * Calculate the memory address and write it to the LSQ entry and the LSQ entry is marked as valid

* **Memory**

   * Once the LSQ entry for this load reaches to LSQ top and is valid, issue the load to memory

   * Remove LSQ entry after memory access completes and send the result to write-back stage and data-bypass bus

* **Write-back**

   * Write the result to physical register file

   * Mark the corresponding ROB entry as ready to commit

* **Commit**

   * Once this instruction comes to ROB top and no exception has occurred, update the commit (retirement) rename table

   * Add the ``PREV_PRD`` back to the free list of physical registers
   
   * Deallocate the ROB entry

Stores
------

Consider the following store instruction in logical format: **STORE RS1, RS2, IMM**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction
  
* **Dispatch (Rename-Dispatch)**

   * Read the front-end rename table to get the latest physical register mappings for ``RS1`` and ``RS2``, say ``PRS1`` and ``PRS2``
  
   * After renaming, the instruction becomes: ``STORE PRS1, PRS2, IMM``
     
   * Allocate entry for the instruction in ROB, LSQ and the corresponding IQ (Mem)
   
   * The stage stalls if any of the required resources are not available

* **Issue**

   * Read operands from physical register file if valid, or from data-bypass network

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

Consider the following load instruction in logical format: **BRANCH RS1, RS2, IMM**

* **Fetch**

   * Read instruction from memory

* **Decode**

   * Decode the instruction

* **Dispatch (Rename-Dispatch)**

   * Read the front-end rename table to get the latest physical register mappings for ``RS1`` and ``RS2``, say ``PRS1`` and ``PRS2``

   * After renaming, the instruction becomes: ``BRANCH PRS1, PRS2, IMM``
 
   * Allocate entry for the instruction in ROB and the corresponding IQ (Int)

   * The stage stalls if any of the required resources are not available

* **Issue**

   * Read operands from physical register file if valid, or from data-bypass network

   * If the ``Int-ALU`` is free, issue the instruction to ``Int-ALU`` and remove IQ entry

   * Instruction is not issued until all the source operands are read and the ``Int-ALU`` is available

* **Execute**

   * Evaluate the branch condition and calculate the target address

   * On a misprediction,

      * Stop fetch, decode and dispatch stages

      * Set the ``exception`` bit in the corresponding ROB entry

   * Mark ROB entry of the branch as ready to commit

* **Commit**

   * Once this branch comes to ROB top and is ready to commit, check for misprediction exception

   * On a misprediction,

      * Flush the entire pipeline (LSQ, ROB, IQs etc)

      * Restore the front-end rename tables using commit rename tables

      * Send the new target address to fetch and restart fetch stage

   * Deallocate the ROB entry

.. note::
   Current version of the out of order pipeline treats any branch misprediction as an exception which is handled once the branch comes to ROB top. However, speculative execution and branch tag based rollback mechanism will be implemented in the future versions.
   
.. note::
   Currently out of order core has no branch prediction unit support. It will be added in the future versions.