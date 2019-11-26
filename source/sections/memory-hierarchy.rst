================
Memory Hierarchy
================

This section briefly explains the memory access hierarchy of MARSS-RISCV.

Overview
--------
Any memory access generated for instruction and data, follows the path: **TLB >> MMU >> Caches >> Memory controller >> DRAM**

.. figure:: ../figures/memory-hy.*
   :figwidth: 620 px
   :align: center

   Memory Hierarchy Overview

Memory Management Unit (MMU)
----------------------------
High-level overview of activities performed by MMU:

* Performs TLB lookup in single cycle using the CPU issued virtual address, to get the corresponding physical address

* Does a hardware page table walk on a TLB miss and raises a page fault exception if page walk fails

* If paging is disabled (no translation) or the CPU is running in machine mode, TLBs are by-passed, in such a case virtual address is same as the physical address

* Sends the physical address to L1 caches

.. note::
   During page table walk, DRAM accesses generated to read/write PTEs are routed through L1 data cache if present.

Cache Hierarchy
---------------

* Set-associative virtually indexed and physically tagged caches

* Supports two levels

   * Level 1: separate instruction and data cache

   * Level 2: single shared cache (optional)

* Policies for handling writes:

   * Write-back (WB), Write-through (WT)

* Policies for handling write miss:

   * Write allocate (WA), Write no allocate (WNA)

* Memory access requests are generated and sent to simulated DRAM on Last Level Cache (LLC) miss

.. note::
   Caches are only used for simulating cache access latency and do not store actual data.

DRAM Organization
-----------------

* Simple DRAM model **simulating row buffer hits and misses** with configuration (``src/riscvsim/dram.h``):

   * ``DIMM``: 1
   * ``Ranks``: 2
   * ``Chips``: 8 per rank
   * ``Bank``: 8 per chip
   * ``Column size``: 8-bit
   * ``Memory bus width``: 64-bit

* Physical address to DRAM mapping is as follows:

.. figure:: ../figures/dram-addr-map.*
   :figwidth: 700 px
   :align: center

* DRAM access latencies in CPU cycles are calculated as follows (These parameters can be configured via simulator's JSON configuration file):

   * Row buffer read hit: ``mem_bus_access_rtt + tCL``

   * Row buffer read miss: ``mem_bus_access_rtt_latency + tRP + tRCD + tCL``

   * Row buffer write hit: ``mem_bus_access_rtt_latency + row_buffer_write_latency``

   * Row buffer write miss: ``mem_bus_access_rtt_latency + tRP + tRCD + row_buffer_write_latency``

Memory Access Latency Calculation
---------------------------------
This section describes in detail how memory access latency is calculated and all the actions involved in the process using an example.

Let us assume that the instruction fetch unit is attempting to read instruction at virtual address ``0x80000000`` for the first time.

Following sequence of actions are involved:

1. Virtual address ``0x80000000`` is sent to the instruction TLB to acquire the corresponding physical address ``0x80000000``. Since the virtual address ``0x80000000`` used in this example belongs to kernel space prior to the start of paging, the TLB lookup is bypassed, meaning, the virtual and physical addresses are identical.

2. Both virtual address (``0x80000000``) and physical address (``0x80000000``) are sent accross the cache hierarchy. Since the physical address ``0x80000000`` is being accessed for the first time, it missess in L1 and L2 caches. 

3. L2 cache on a miss sends request to memory controller to read ``64`` bytes starting from physical address ``0x80000000``. ``64`` bytes here denote the cache line size, since all the transfer between caches and the memory controller are done at the granularity of cache line size.

4. Memory controller creates a request of following format: (``starting physical address``, ``bytes to access``, ``Operation (Read/Write)``). In this case, following request is created: (``0x80000000``, ``64 bytes``, ``Read``) and added to frontend memory queue and DRAM dispatch queue.

.. note::
   Internally, for simulating the memory accesses, the simulator maintains 3 queues: Frontend memory queue (which holds the pending accesses to instruction memory), backend memory queue (which holds the pending accesses to data memory) and dram dispatch queue (which holds the pending memory accesses to both instruction and data memory, but in sequence in which they are generated). Frontend and backend memory queues are only used to stalling fetch and memory pipeline stages respectively. Actual accesses happen from the top of dram dispatch queue.

5. Until the frontend memory queue is empty, fetch unit stalls. 

6. At some point, when the request (``0x80000000``, ``64``, ``Read``) reaches to the top of DRAM dispatch queue and the memory bus is free, memory controller breaks above request on granularity of memory bus width. In this case, memory bus width is configured to be of 64 bits, so it will generate 8 sub-requests each of 64-bits, starting from address ``0x80000000`` and get the dram access latency for each of this sub-requests as follows:

+------------+------------+---------+
|   Address  | Bits       | Latency |
+------------+------------+---------+
| 0x80000000 | 64         | 51      |
+------------+------------+---------+
| 0x80000008 | 64         | 17      |
+------------+------------+---------+
| 0x80000010 | 64         | 17      |
+------------+------------+---------+
| 0x80000018 | 64         | 17      |
+------------+------------+---------+
| 0x80000020 | 64         | 17      |
+------------+------------+---------+
| 0x80000028 | 64         | 17      |
+------------+------------+---------+
| 0x80000030 | 64         | 17      |
+------------+------------+---------+
| 0x80000038 | 64         | 17      |
+------------+------------+---------+
| Total      | 512 bits   | 170     |
|            | (64 Bytes) |         |
+------------+------------+---------+


7. Hence the total memory access latency for this instruction is: ``L1_icache_probe_latency (1) + L2_cache_probe_latency (2) + 170 cycles for DRAM access``.

8. Memory bus is marked as busy and after a total of 173 cycles, the memory access is marked as complete, the addresses read are installed in the caches and the request is removed from frontend memory queue and DRAM dispatch queue. The stall on fetch unit is removed and the instruction continues to the next pipeline stage.

.. note::
   Latency calculations for data memory and hardware page walk in case of TLB misses follows the same exact steps discussed above.