==========================
TinyEMU (RISC-V Emulator)
==========================

TinyEMU is a full system emulator and a virtual machine for
the RISC-V architecture which can boot and run an RISC-V Linux.
TinyEMU consists of various components such as a CPU emulator,
memory management unit, IO devices, Network, interrupt device support
like PLIC, CLINT etc. Memory management module in TinyEMU
handles all the physically allocated memory of the emulated guest
machine DRAM, memory mapped IO (MMIO), PCI address space
etc. It maps the guest CPU memory accesses to the correct physical
memory locations in the host system, checks the validity of the
CPU issued memory addresses and generates a page fault exception
in the emulated CPU on a page miss in the guest machine DRAM.

.. figure:: ../figures/TinyEMU.*
   :figwidth: 240 px
   :align: center

   TinyEMU Architecture

MARSS-RISCV Architecture
------------------------

MARSS-RISCV is basically a **simulation suite** consisting of a cycle accurate CPU, branch prediction unit, MMU, caches and DRAM, built on top of TinyEMU, working with the physical memory and the devices emulated by TinyEMU.

.. figure:: ../figures/marss-riscv-arch.*
   :figwidth: 500 px
   :align: center

   MARSS-RISCV Architecture