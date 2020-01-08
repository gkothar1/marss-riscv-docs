============
Introduction
============
MARSS-RISCV (Micro-ARchitectural System Simulator - RISCV) is a open source, cycle-accurate single core full-system (Linux) micro-architectural simulator for the RISC-V ISA built upon TinyEMU emulator (https://bellard.org/tinyemu) by Fabrice Bellard and uses its code for all the device emulation and configuration. It consists of detailed cycle accurate models of a modern RISC-V In-order and Out-of-order processor with branch prediction unit and a complete memory hierarchy including TLBs, caches and DRAM. It comes integrated with DRAMSim2 (https://github.com/umd-memsys/DRAMSim2), a cycle accurate memory system simulator. It is currently being developed and maintained by CAPS (Computer Architecture and Power Aware Systems Research Group) at the State University of New York at Binghamton. Being a true full system simulator, MARSS-RISCV can simulate all of the system in a cycle accurate fashion including OS code, libraries, interrupt handlers etc.

.. note::
   Currently, our simulator is in alpha status as we are validating the cycle accuracy using various development boards. The simulated in-order core is tested and operational, however, the simulated out-of-order core is in microarchitectural testing phase.

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

   b. Choose from 2 DRAM memory models: Simple DIMM based basic DRAM model that simulates row-buffer (open-page) hits and DRAMSim2

4. **Configurable branch prediction unit**

   a. Bimodal and 2-level adaptive predictor (Gshare, Gselect, GAg, GAp, PAg, PAp) 

5. **Checkpoint based simulations**

   a. Provides simulation start and simulation stop markers with seamless switching between simulation and emulation mode transparent to the applications running on the guest VM
   
   b. Enables the users and researchers to fast-forward simulations to regions of interest where cycle accurate simulations are actually needed