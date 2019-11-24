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