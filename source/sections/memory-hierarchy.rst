==========================
Simulation of memory access delay
==========================

This section briefly describes the memory access delay simulation and the simulated components of the memory hierarchy. The memory hierarchy includes four main components:

1. Memory controller
2. Translation look-aside buffers (TLB)
3. Caches
4. DRAM

.. note::
   We do not model the actual data in the memory hierarchy for simplicity, but just the addresses (or cache tag arrays) for simulating the delays.

Translation look-aside buffers (TLB)
====================================
TinyEMU emulates a hardware memory management unit (MMU) that manages emulated guest memory and supports *sv32*,  *sv39*, and *sv48* paged virtual memory schemes.  It translates the virtual memory addresses issued by the emulated CPU to the host system's correct physical memory locations using three distinct direct-mapped address translation buffers (or TLB) for code, loads, and stores. It checks the validity of the CPU issued memory addresses, performs a page table walk in case of a TLB miss, and generates a page fault exception in the emulated CPU on a page miss in the emulated memory.

MARSS-RISCV uses the MMU emulated by TinyEMU to fetch the instructions and data into the simulated CPU pipeline. TLB lookup and address translation on a TLB hit are modeled to complete in a **single cycle** and in parallel with the cache lookup. On a TLB miss, the simulator creates a memory request per page table entry containing the entry's physical address and appends it to the memory controller queue.

Caches
======
The cache hierarchy model consists of two levels of physically indexed, physically tagged blocking caches. Level-1 caches include a separate instruction and data cache, whereas Level-2 consists of an optional unified cache. The cache accesses are non-pipelined L2 cache can be accessed in parallel by split L1 caches. A miss in the last level cache (LLC) or eviction creates a memory request appended to the memory controller queue.

Memory controller
=================
The memory controller includes a single FIFO queue known as ``mem_req_queue`` comprising all the pending memory access requests. Requests are processed sequentially, one at a time, from the head of ``mem_req_queue``. 

Memory access requests are added to ``mem_req_queue`` by cache lookup functions on a cache miss or TLB lookup functions on a TLB miss (for modifying page table entries). 

.. note::
   We do not stall the CPU pipeline stage for the ``write`` requests to complete.  However, the delay for a ``write`` request is nevertheless simulated asynchronously through the memory controller.

DRAM
=================
MARSS-RISCV supports three DRAM model: ``Base Model``, ``Ramulator`` and ``DRAMSim3``.

Base DRAM model
-------------------
When MARSS-RISCV is configured to run with the ``base`` DRAM model, requests are processed sequentially, from the head of ``mem_req_queue``. Processing solely involves simulating a fixed configurable latency in CPU cycles, known as ``mem_access_latency``. After simulating the latency, the stall on the waiting CPU pipeline stage is removed, and the current entry is dequeued from ``mem_req_queue``.  

The ``Base DRAM`` model keeps track of the physical page number of the latest request processed. Any subsequent access to the same physical page occupies a lower delay, roughly 60 percent of the fixed ``mem_access_latency``.

.. figure:: ../figures/mem-hy.*
   :figwidth: 500 px
   :align: center

   Simulated Memory Hierarchy