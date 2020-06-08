==========================
Simulation of memory access delay
==========================

This section briefly describes the simulation of the memory access delay and the simulated components of the memory hierarchy. The memory hierarchy includes three main components:

1. Memory controller
2. Translation look-aside buffers (TLB)
3. Caches

.. note::
   We do not model the actual data in the caches and memory controller for simplicity, but just the addresses for simulating the delays.

Memory controller
=================
The memory controller includes a single FIFO queue known as ``mem_req_queue`` comprising all the pending memory access requests. Requests are processed sequentially, one at a time, from the head of ``mem_req_queue``. Every memory access request consists of:

1.  Starting physical address 
2. Type of access: ``read``/``write``
3. Access size (cache-line size)
4. Pipeline stage which generated access: ``fetch``/``memory``
5. A bit indicating whether this request is for a page table entry

Memory access requests are added to ``mem_req_queue`` by cache lookup functions on a cache miss or by TLB lookup functions on a TLB miss (for modifying page table entries). 

.. note::
   We don't stall the CPU pipeline stage for the ``write`` requests to complete.  However, the delay for a ``write`` request is nevertheless simulated asynchronously through the memory controller.

Base DRAM model
-------------------
When MARSS-RISCV is configured to run with the ``base`` DRAM model, requests are processed sequentially, from the head of ``mem_req_queue``. Processing solely involves simulating a fixed configurable latency in CPU cycles, known as ``mem_access_latency``. After simulating the latency, the stall on the waiting CPU pipeline stage is removed, and the current entry is dequeued from ``mem_req_queue``.  

In ``base`` mode, the memory controller keeps track of the physical page number of the latest request processed. Any subsequent accesses to the same physical page occupies a lower delay, which is roughly 60 percent of the fixed ``mem_access_latency``.

DRAMSIM2
------------
When ``dramsim2`` is enabled, requests are sent to DRAMSIM2 from the head of ``mem_req_queue``. When the callback from DRAMSIM2 is received, the stall on the waiting CPU pipeline stage is removed. The current entry is dequeued from ``mem_req_queue``.  

Translation look-aside buffers (TLB)
====================================
TinyEMU emulates a hardware memory management unit (MMU) that manages emulated guest memory and provides support for *sv32*,  *sv39*, and *sv48* paged virtual memory schemes.  It translates the virtual memory addresses issued by the emulated CPU to the correct physical memory locations in the host system using three distinct direct-mapped address translation buffers (or TLB) for code, loads, and stores. It checks the validity of the CPU issued memory addresses, performs page table walk in case of a TLB miss, and generates a page fault exception in the emulated CPU on a page miss in the emulated memory.

MARSS-RISCV uses the MMU emulated by TinyEMU to fetch the instructions and data into the simulated CPU pipeline. TLB lookup and address translation on a TLB hit is modeled to complete in a **single cycle** and happens in parallel with the cache lookup. On a TLB miss, the simulator creates a memory request per page table entry containing the physical address of the entry and appends it to the memory controller queue.

Caches
======
Cache hierarchy model consists of two levels of physically indexed, physically tagged blocking caches. Level-1 caches include a separate instruction and data cache, whereas Level-2 consists of an optional unified cache. 

The cache accesses are non-pipelined and follow a non-inclusive non-exclusive design, meaning the contents of the lower level cache are neither strictly inclusive nor exclusive of the higher-level cache.  L2 cache can be accessed in parallel by split L1 caches. 

The size, associativity, read-hit latency, write-hit latency, line size, eviction policy (``random`` or ``pseudo-LRU``), write policy (``write-through`` or ``write-back``), and write-miss policy (``write-allocate`` or ``write-no-allocate``) is configurable. A miss in the last level cache (LLC) or eviction creates a memory request which is appended to the memory controller queue.

.. figure:: ../figures/mem-hy.*
   :figwidth: 500 px
   :align: center

   Simulated Memory Hierarchy