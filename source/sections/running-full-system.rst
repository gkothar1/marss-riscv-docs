=====================
Simulating Benchmarks
=====================
This section describes how to configure the simulator as per the desired configuration and simulate benchmarks in full system mode. 

For this tutorial, we shall be configuring MARSS-RISCV to simulate a simple 5-stage 32-bit in-order RISC-V system on a chip (SoC) and run `CoreMark <https://github.com/eembc/coremark>`_. Coremark is an industry-standard benchmark that measures the performance of central processing units (CPU) and embedded microcontrollers (MCU).

System Requirements and Dependencies
====================================
Check the system requirements and install the required dependencies.

System requirements
-------------------
* 32-bit or 64-bit Linux machine
* Libcurl, OpenSSL and SDL Libraries
* Standard C and C++ compiler

Installing the dependencies
---------------------------
Ensure all the dependencies (``ssl``, ``sdl``, and ``curl`` libraries) are installed on the system. For Debian-based (including Ubuntu) systems, the packages are: ``build-essential``, ``libssl-dev``, ``libsdl1.2-dev``, ``libcurl4-openssl-dev``.

.. code-block:: console

    $ sudo apt-get update
    $ sudo apt-get install build-essential
    $ sudo apt-get install libssl-dev
    $ sudo apt-get install libsdl1.2-dev
    $ sudo apt-get install libcurl4-openssl-dev

Compiling the simulator
=======================
First, clone the simulator repository:

.. code-block:: console

    $ git clone https://github.com/bucaps/marss-riscv

Then, ``cd`` into the simulator source directory:

.. code-block:: console

    $ cd marss-riscv/src/

Open Makefile and set ``CONFIG_XLEN`` to ``32``. Then, compile the simulator using:

.. code-block:: console

    $ make

Preparing the bootloader, kernel, and userland image
===================================================

The simplest way to start is by using a pre-built bootloader, kernel, and userland image. The pre-built 32-bit and 64-bit RISC-V userland, bootloader, and kernel are available here: `marss-riscv-images
<http://cs.binghamton.edu/~marss-riscv/marss-riscv-images.tar.gz>`_

.. code-block:: console

    $ wget http://cs.binghamton.edu/~marss-riscv/marss-riscv-images.tar.gz
    $ tar -xvzf marss-riscv-images.tar.gz

The userland image needs to be decompressed first:

.. code-block:: console

    $ cd marss-riscv-images/riscv32-unknown-linux-gnu/
    $ xz -d -k -T 0 riscv32.img.xz

Note that the file system on the disk image has almost no space. Hence we need to resize it to the desired size.

Grow the image file to the desired size (``8GB`` for this tutorial):

.. code-block:: console

    $ truncate --size 8G riscv32.img

.. note::
   Below steps may require ``sudo`` access.

Find the first available ``losetup`` device. On my system, the below command returned: ``/dev/loop8``

.. code-block:: console

    $ sudo losetup -f

Attach the disk image to the given loopback device:

.. code-block:: console

    $ losetup /dev/loop8 riscv32.img

Run fsck before growing the file system:

.. code-block:: console

    $ e2fsck -f /dev/loop8

.. note::
   You may require ``e2fsck`` version ``1.43.1`` or greater.

Grow the file system to its maximum size:

.. code-block:: console

    $ resize2fs /dev/loop8

Run fsck post resize:

.. code-block:: console

    $ e2fsck -f /dev/loop8

Detach the loopback device:

.. code-block:: console

    $ losetup -d /dev/loop8

At this point, you should have a 32-bit RISC-V Linux image of size 8GB ready to use.

Configuring the Simulator
=========================
Simulation and TinyEMU SoC parameters are configured using the TinyEMU JSON configuration file provided in the configs directory.  We will now configure MARSS-RISCV to simulate a single-core 32-bit RISC-V SoC with the following configuration:

* 32-bit in-order core with a 5-stage pipeline, 1GHz clock
* 32-entry instruction and data TLBs
* 32-entry 2-way branch target buffer with a simple bimodal predictor, with 256-entry history table
* 4-entry return address stack
* single-stage integer ALU with one cycle delay
* 2-stage pipelined integer multiplier with one-cycle delay per stage
* single-stage integer divider with eight cycles delay
* All the instructions in FPU ALU with a latency of 2 cycles
* 3-stage pipelined floating-point fused multiply-add unit with one-cycle delay per stage
* 32KB 8-way L1-instruction and L1-data caches with one cycle latency and LRU eviction
* 2MB 16-way L2-shared cache with 12 cycle latency and LRU eviction
* 64-byte cache line size with write-back and write-allocate caches
* 1024MB DRAM with base DRAM model with 75 cycles for  main memory access

Based on the above configuration, the ``configuration`` will look like the below. 

.. code-block:: json

    /* VM configuration file */
    {
        version: 1,
        machine: "riscv32", /* riscv32, riscv64 */
        memory_size: 1024, /* MB */
        bios: "riscv32-unknown-linux-gnu/bbl32.bin",
        kernel: "riscv32-unknown-linux-gnu/kernel-riscv32.bin",
        cmdline: "console=hvc0 root=/dev/vda rw",
        drive0: { file: "riscv32-unknown-linux-gnu/riscv32.img" },
        eth0: { driver: "user" },

        core: {
            name: "32-bit inorder riscv CPU",
            type: "incore", /* incore, oocore */
            cpu_freq_mhz: 1000,
            rtc_freq_mhz: 10,

            incore : {
                num_cpu_stages: 5, /* 5, 6 */
            },

            oocore: {
                iq_size: 16,
                iq_issue_ports: 3,
                rob_size: 64,
                rob_commit_ports:4,
                lsq_size: 16,
            },

            /* Note: Latencies for functional units, caches and memory are specified in CPU cycles */
            functional_units: {
                num_alu_stages: 1,
                alu_stage_latency: "1",
        
                num_mul_stages: 2,
                mul_stage_latency: "1,1",
        
                num_div_stages: 1,
                div_stage_latency: "8",
        
                /* Note: This will create a pipelined FP-FMA unit with 4 stages with a
                 * latency of 1 CPU cycle(s) per stage */
                num_fpu_fma_stages: 2,
                fpu_fma_stage_latency: "1,1",
        
                /* Note: FP-ALU is non-pipelined */
                fpu_alu_stage_latency: {
                    fadd: 2,
                    fsub: 2,
                    fmul: 2,
                    fdiv: 2,
                    fsqrt: 2,
                    fsgnj: 2,
                    fmin: 2,
                    fmax: 2,
                    feq: 2,
                    flt: 2,
                    fle: 2,
                    cvt: 2,
                    fcvt: 2,
                    fmv: 2,
                    fclass: 2,
                },

                /* Latency for RISC-V SYSTEM opcode instructions (includes CSR and privileged instructions)*/
                system_insn_latency: 3,
            },
        
            bpu: {
                enable: "true", /* true, false */
                flush_on_context_switch: "false", /* true, false */

                btb: {
                    size: 32,
                    ways: 2,
                    eviction_policy: "lru", /* lru, random */
                },
        
                bpu_type: "bimodal", /* bimodal, adaptive */
        
                bimodal: {
                    bht_size: 256,
                },
        
                adaptive: {
                    ght_size: 1,
                    pht_size: 1,
                    history_bits: 2,
                    aliasing_func_type: "xor", /* xor, and, none */

                    /* Given config for adaptive predictor will create a Gshare predictor:
                    *   1) global history table consisting of one entry, entry includes a 2-bit history register
                    *   2) pattern history table consisting of one entry, entry includes an array of 4 saturating counters
                    *   3) value of history register will be `xor` ed with branch PC to index into the array of saturating counters
                    */
                },
        
                ras_size: 4, /* value 0 disables RAS */
            },
        
            caches: {
                enable_l1_caches: "true", /* true, false */
                allocate_on_write_miss: "true", /* true, false */
                write_policy: "writeback", /* writeback, writethrough */
                line_size: 64, /* Bytes */
        
                icache: {
                    size: 32, /* KB */
                    ways: 8,
                    latency: 1,
                    eviction: "lru", /* lru, random */
                },
        
                dcache: {
                    size: 32, /* KB */
                    ways: 8,
                    latency: 1,
                    eviction: "lru", /* lru, random */
                },
        
                l2_shared_cache: {
                    enable: "true",
                    size: 2048, /* KB */
                    ways: 16,
                    latency: 12,
                    eviction: "lru", /* lru, random */
                },
            },
        },

        memory: {
            tlb_size: 32,

            /* Memory controller burst-length in bytes */ 
            /* Note: This is automatically set to cache line size if caches are enabled */
            burst_length: 64, /* Bytes */

            base_dram_model: {
                mem_access_latency: 75,
            },

            dramsim3: {
                config_file: "DRAMsim3/configs/DDR4_4Gb_x16_2400.ini",
            },

            ramulator: {
                config_file: "ramulator/configs/DDR4-config.cfg",
            },
        },
    }


Run the simulator
=================

.. code-block:: console

    $ ./marss-riscv -rw -ctrlc -sim-mem-model base <path-to-config-file>

Load the benchmark and the simulation utility programs inside the guest VM
==========================================================================
Now we will load the CoreMark benchmark and MARSS-RISCV simulation utility programs using ``git clone`` inside the guest VM. Before that, you may want to set the time to the current time in the VM manually. So in the guest terminal, type:

.. code-block:: console

    $ date --set="9 Dec 2019 10:00:00"

To clone the repos, type:

.. code-block:: console

    $ git clone https://github.com/eembc/coremark.git
    $ git clone https://github.com/bucaps/marss-riscv-utils.git

To install the simulation utility programs, type:

.. code-block:: console

    $ cd marss-riscv-utils
    $ make

This installs the following commands (programs): ``simstart``, ``simstop`` and ``simulate``, which will help us to enable and disable simulation mode, before and after running CoreMark, respectively. 

At this point, we are pretty much ready to run CoreMark.

Run Benchmark
=============

Switch to CoreMark directory inside the guest VM and compile the benchmark:

.. code-block:: console

    $ cd ../coremark
    $ make compile

This will generate the coremark executable: ``coremark.exe``. It has 3 set of inputs and the command lines are as follows (based on Makefile):

* ``./coremark.exe  0x0 0x0 0x66 0 7 1 2000 > ./run1.log``
* ``./coremark.exe  0x3415 0x3415 0x66 0 7 1 2000  > ./run2.log``
* ``./coremark.exe 8 8 8 0 7 1 2000 > ./run3.log``

Then, to simulate the benchmark inside the guest VM, type:

.. code-block:: console

    $ simstart; ./coremark.exe  0x0 0x0 0x66 0 7 1 2000 > ./run1.log; simstop;
    $ simstart; ./coremark.exe  0x3415 0x3415 0x66 0 7 1 2000  > ./run2.log; simstop;
    $ simstart; ./coremark.exe 8 8 8 0 7 1 2000 > ./run3.log; simstop;

After every ``simstop`` command, the summary of the performance stats is printed on the console, and the stats file for every run is generated based on the current time-stamp.