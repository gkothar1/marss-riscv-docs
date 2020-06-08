==========================
TinyEMU (RISC-V Emulator)
==========================

﻿TinyEMU is a full system emulator and a virtual machine for the RISC-V architecture, which can boot and run a RISC-V Linux. It maintains the entire state of the RISC-V CPU in software. It consists of the program counter, CPU privilege mode, integer, and floating-point register files and control and status registers (CSR). TinyEMU has a CPU emulator that supports user, supervisor, and machine privilege modes and uses the binary interpretation technique to perform the emulation of the RISC-V system. TinyEMU also emulates a hardware memory management unit (MMU) that manages emulated guest memory and provides support for *sv32*, *sv39*, and *sv48* paged virtual memory schemes.  It translates the virtual memory addresses issued by the emulated CPU to the correct physical memory locations in the host system using three distinct direct-mapped address translation buffers (or TLB) for code, loads, and stores. It also checks the validity of the CPU issued memory addresses and generates a page fault exception in the emulated CPU on a page miss in the emulated memory. Emulated devices include standard platform-level interrupt controller (PLIC), core local interrupter (CLINT), real-time clock device (RTC), universal asynchronous receiver/transmitter (UART), VirtIO and Network block device. MARSS-RISCV is a cycle-accurate CPU simulator built on top of TinyEMU. It utilizes RISC-V CPU state, physical memory, MMU, and all the devices emulated by TinyEMU. Figure \ref{fig:marss-arch} show the high-level architecture of MARSS-RISCV.

.. figure:: ../figures/marss-riscv-arch.*
   :figwidth: 500 px
   :align: center

   MARSS-RISCV Architecture