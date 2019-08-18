============
Introduction
============
MARSS-RISCV (Micro-Architectural System Simulator - RISCV), is a **open source, cycle-accurate single-core full system (Linux) micro-architectural simulator** for the RISC-V instruction set architecture (ISA) built upon RISCVEMU hypervisor, a system emulator for RISC-V. It consists of detailed cycle accurate models of an modern RISC-V in-order and out-of-order core, branch prediction unit and memory hierarchy. Being a true full system simulator, MARSS-RISCV can simulate all of the system in a cycle accurate fashion including OS code, libraries, interrupt handlers
etc.

Main Features 
-------------

1. **True cycle-accurate full system simulator**

   a. Not a functional simulator of the RISC-V ISA, pipeline hardware simulation is cycle-accurate, i.e. pipeline and memory simulation activities model the activities in the real system correctly for every clock cycle

   b. Simulates all of the system in a cycle accurate fashion including OS code, libraries, interrupt handlers etc

   c. Provides real-time IO simulations and cycle-accurate simulation statistics for all the CPU privilege modes (user, supervisor, hyper-visor and machine)

   d. Complete system simulation tool that can simulate chip-set, DRAM, network interface cards and peripheral devices along with the application, library, OS code that use the hardware

2. **Configurable RISC-V CPU**

   a. 5-stage and 6-stage in-order pipeline configurations

   b. 8-stage out-of-order pipeline configuration

   c. Multiple execution units with configurable latencies (can be pipelined)

   d. Supports RV32GC and RV64GC (user level ISA version 2.2 and privileged architecture version 1.10)

3. **Configurable memory hierarchy**

   a. 2-level cache hierarchy with various allocation and miss handling policies

   b. Simple DIMM based DRAM model simulating row-buffer hits and misses

4. **Configurable branch prediction unit**

   a. Bimodal and 2-level adaptive predictor (Gshare, Gselect, GAg, GAp, PAg, PAp) 

5. **Checkpoint based simulations**

   a. Provides simulation start and simulation stop markers with seamless switching between simulation and emulation mode transparent to the applications running on the guest VM
   
   b. Enables the users and researchers to fast-forward simulations to regions of interest where cycle accurate simulations are actually needed