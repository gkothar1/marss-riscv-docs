=====================
Simulating Benchmarks
=====================
This section describes in detail how to configure the simulator as per the desired configuration and simulate benchmarks in full system mode. 

For this tutorial, we shall be configuring MARSS-RISCV to simulate a simple 5-stage 32-bit in-order RISC-V processor and run `CoreMark <https://github.com/eembc/coremark>`_, an industry-standard benchmark that measures the performance of central processing units (CPU) and embedded microcrontrollers (MCU).

System Requirements and Dependencies
====================================
Check the system requirements and install the required dependencies.

System requirements
-------------------
* 32-bit or 64-bit Linux machine
* Libcurl, OpenSSL and SDL Libraries
* Standard C compiler

Installing the dependencies
---------------------------
Make sure that you have all the dependencies (`ssl`, `sdl`, and `curl` libraries) installed on the system. For Debian-based (including Ubuntu) systems, the packages are: ``build-essential``, ``libssl-dev``, ``libsdl1.2-dev``, ``libcurl4-openssl-dev``. 

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

	$ export MARSS_RISCV_TOP=$(pwd)
	$ cd $MARSS_RISCV_TOP
	$ git clone https://github.com/bucaps/marss-riscv

Then, ``cd`` into the simulator source directory:

.. code-block:: console

	$ cd $MARSS_RISCV_TOP/marss-riscv/src/

Modify ``CONFIG_XLEN`` and ``CONFIG_FLEN`` options in the Makefile. Set ``CONFIG_XLEN`` to ``32`` and ``CONFIG_FLEN`` to ``64``. Then, compile the simulator using:

.. code-block:: console

	$ make

Preparing the bootloader, kernel and user-land image
===================================================

Using pre-built bootloader, kernel and user-land images is the easiest way to start. The pre-built images are located in the `marss-riscv-images <https://github.com/bucaps/marss-riscv-images>`_ repository. To clone it, make sure you have `Git LFS <https://git-lfs.github.com/>`_ installed on your system, and type:

.. code-block:: console

	$ cd $MARSS_RISCV_TOP
	$ git clone https://github.com/bucaps/marss-riscv-images

The user-land image needs to be decompressed first:

.. code-block:: console

	$ cd $MARSS_RISCV_TOP/marss-riscv-images/riscv32-unknown-linux-gnu/
	$ xz -d -k -T 0 riscv32.img.xz

Note that the file system on the disk image has almost no space. Hence we need to resize it to the desired size.

Grow the image file to the desired size (``8GB`` for this tutorial):

.. code-block:: console

	$ truncate --size 8G riscv32.img

Attach the disk image to a loopback device: 

.. code-block:: console

	$ losetup /dev/loop0 riscv32.img

Run fsck before growing the file system:

.. code-block:: console

	$ e2fsck -f /dev/loop0

Grow the file system to its maximum size:

.. code-block:: console

	$ resize2fs /dev/loop0

Run fsck post resize:

.. code-block:: console

	$ e2fsck -f /dev/loop0

Detach the loopback device:

.. code-block:: console

	$ losetup -d /dev/loop0

At this point, you should have a 32-bit RISC-V Linux image of size 8GB ready to use.

Configuring the Simulator
=========================
Simulation parameters can be configured using ``riscvemu.cfg``, located in ``$MARSS_RISCV_TOP/marss-riscv-images/riscv32-unknown-linux-gnu/``. We will now configure MARSS-RISCV to simulate a single core 32-bit RISC-V machine with the following configuration:

* 32 bit in-order core with 5-stage pipeline
* 32-entry instruction and data TLB
* 32-entry 2-way branch target buffer with a simple bimodal predictor
* single stage integer ALU with 1 cycle delay
* 2-stage pipelined integer multiplier with 2 cycle delay per stage
* single stage integer divider with 8 cycles delay
* single stage floating point ALU with 2 cycles delay
* 2-stage pipelined floating point fused multiply add unit with 2 cycle delay per stage
* 32KB 8-way L1-instruction and L1-data write-back caches with 1 cycle probe delay and LRU eviction
* 2MB 16-way L2-shared write-back cache with 2 cycle probe delay and LRU eviction
* 32-byte cache line size
* 1024MB DRAM with tCL-tRCD-tRP (17-17-17) cycles respectively

Based on the above configuration, the ``riscvemu.cfg`` will look like below. You can modify your copy of ``riscvemu.cfg`` accordingly or just paste the contents below in your copy.

.. code-block:: json

	/* VM configuration file */
	{
		version: 1,
		machine: "riscv32",
		bios: "bbl32.bin",
		cmdline: "console=hvc0 root=/dev/vda rw",
		drive0: { file: "riscv32.img" },
		eth0: { driver: "user" },

		core_name: "test-riscv-core",           /* Name of the simulated RISC-V CPU */
		core_type: "incore",                    /* Core type: In-order (incore) or Out-of-order(oocore) */

		/** In-order core parameters **/
		num_cpu_stages: 5,                      /* Number of pipeline stages: 5, 6 */

		/** Out-of-order core parameters **/
		prf_int_size: 64,                      /* Number of integer physical registers */
		prf_fp_size: 64,                       /* Number of floating-point physical registers */
		iq_int_size: 16,                       /* Number of integer issue-queue entries */
		iq_fp_size: 16,                        /* Number of floating-point issue-queue entries */
		iq_mem_size: 16,                       /* Number of memory issue-queue entries */
		iq_int_issue_ports: 3,		       /* Number of issue ports on integer issue-queue */
		iq_fp_issue_ports: 2,		       /* Number of issue ports on floating-point issue-queue */
		iq_mem_issue_ports: 2,		       /* Number of issue ports on memory issue-queue */	
		prf_int_write_ports: 3,                /* Number owrite ports to integer PRF */ 
		prf_fp_write_ports: 2,                 /* Number of write ports to floating-point PRF */ 
		rob_size: 64,                          /* Number of ROB entries */
		rob_commit_ports:4,		       /* Number of ROB commit ports */
		lsq_size: 16,                          /* Number of LSQ entries */ 

		/** Tracing and Logging Parameters **/
		sim_stats_file: "simstats.txt",         /* Path to the file to save simulation stats */
		sim_trace_file: "simtrace.txt",         /* Path to the file to save commit trace, must compile MARSS-RISCV with CONFIG_SIM_TRACE CFLAG */

		/** Execution Unit Parameters **/
		num_alu_stages: 1,                      /* Number of stages for integer ALU unit */
		alu_stage_latency: "1",                 /* Latencies for all integer ALU stages, comma separated. */			

		num_mul_stages: 2,                      /* Number of stages for integer MUL unit */			
		mul_stage_latency: "2,2",                 /* Latencies for all integer MUL stages, comma separated. eg. "1,2,1" */

		num_div_stages: 1,                      /* Number of stages for integer DIV unit */
		div_stage_latency: "8",                 /* Latencies for all integer DIV stages, comma separated. eg. "1,2,1" */

		num_fpu_alu_stages: 1,                  /* Number of stages for floating point ALU unit */
		fpu_alu_stage_latency: "2",             /* Latencies for all floating point ALU stages, comma separated. eg. "1,2,1" */

		num_fpu_fma_stages: 2,                  /* Number of stages for floating point fused multiply add unit */
		fpu_fma_stage_latency: "2,2",             /* Latencies for all floating point fused multiply add stages, comma separated. eg. "1,2,1" */

		/** BPU Parameters **/
		enable_bpu: "true",                     /* Enable branch prediction unit: true, false */
		btb_size: 32,                          /* Number of entries in BTB, must be power of 2 */
		btb_ways: 2,                            /* Number of BTB ways */
		btb_eviction_policy: "lru",             /* Eviction policy for BTB: lru, random */
		bpu_type: "bimodal",                   /* Type of predictor: bimodal, adaptive */
		bpu_ght_size: 1,                        /* Number of entries in Global history table, must be power of 2 */
		bpu_pht_size: 1,                        /* Number of entries in Pattern history table, must be power of 2 */
		bpu_history_bits: 2,                    /* Number of bits in history register */
		bpu_aliasing_func_type: "xor",          /* Aliasing function for adaptive predictor: and, xor, none
	                                                   This can be used to construct Gshare(xor) and Gselect(and) predictors
	                                                   when bpu_ght_size and bpu_pht_size are set to 1 */

		/** DRAM Parameters **/
		tlb_size: 32,                           /* Number of entries in instruction TLB, load TLB and store TLB */
		memory_size: 1024,                      /* Size of DRAM in MB */
		dram_burst_size: 32,                    /* DRAM burst size in bytes (Cache line size if caches are enabled) */
		mem_bus_access_rtt_latency: 0,		/* Round trip delay of memory bus in CPU cycles */
		tCL: 17,					/* Number of CPU cycles to read the data from a active DRAM row and drive it on the memory bus */
		tRCD: 17,				/* Number of CPU cycles required between opening a row of memory and accessing columns within it */
		tRP: 17,					/* Number of CPU cycles required between issuing the precharge command and opening the next row */
		row_buffer_write_latency: 17,		/* Number of CPU cycles required to write the data in the already active row */

		/** Cache Parameters **/
		enable_l1_caches: "true",               /* Enable L1 caches: true, false */

		icache: {
			size: 32,                       /* Size of icache in KB */
			ways: 8,                        /* Number of ways in icache */
			probe_latency: 1,               /* Probe latency for icache in CPU cycles */
			eviction: "lru",             /* Eviction policy for icache: lru, random */
		},

		dcache: {
			size: 32,                       /* Size of dcache in KB */
			ways: 8,                        /* Number of ways in dcache */
			probe_latency: 1,               /* Probe latency for dcache in CPU cycles */
			eviction: "lru",             /* Eviction policy for dcache: lru, random */
		},

		enable_l2_cache: "true",               /* Enable l2_shared_cache: true, false */

		l2_shared_cache: {
			size: 2048,                      /* Size of l2_shared_cache in KB */
			ways: 16,                        /* Number of ways in l2_shared_cache */
			probe_latency: 2,               /* Probe latency for l2_shared_cache in CPU cycles */
			eviction: "lru",             /* Eviction policy for l2_shared_cache: lru, random */
		},

		cache_allocate_on_write_miss: "true",   /* Allocate entry in cache on write miss: true, false  */
		cache_write_policy: "writeback",        /* Cache write policy: writeback, writethrough */
		words_per_cache_line: 8,                /* Number of words in each cache line
		                                           (used to determine cache-line-size = words_per_cache_line * 'XLEN' bits) */
	}

Run the simulator
=================

By default, the simulator will boot in "snapshot" mode, meaning it will **not** retain the file system changes after it is shut down. In order to persist the changes, pass ``-rw`` command line argument to the simulator. By default, guest boots in emulation mode. To start in simulation mode run with ``-simstart`` command line option. But for now, we will let it start in emulation mode and switch into simulation mode just before running the benchmark.

.. code-block:: console

	$ cd $MARSS_RISCV_TOP/marss-riscv/src
	$ ./marss-riscv -rw -ctrlc $MARSS_RISCV_TOP/marss-riscv-images/riscv32-unknown-linux-gnu/riscvemu.cfg

Once the guest boots, we need to initialize the environment. Normally, this should happen automatically but due to an unresolved bug it needs to done explicitly. So, once you have access to the guest machine terminal, type:

.. code-block:: console

	$ export PYTHONPATH=/usr/lib64/python2.7/site-packages/
	$ env-update

The system is ready for use. It has a working GCC compiler, ssh, git, and `more <https://github.com/bucaps/marss-riscv-images/blob/master/riscv32-unknown-linux-gnu/PACKAGES>`_.

Load the benchmark and the simulation utility programs inside the guest VM
==========================================================================
Now we will load the CoreMark benchmark and MARSS-RISCV simulation utility programs inside the guest VM. For that, on the guest VM terminal, type:

.. code-block:: console

	$ git clone https://github.com/eembc/coremark.git
	$ git clone https://github.com/bucaps/marss-riscv-utils.git

To install the simulation utility programs, type:

.. code-block:: console

	$ cd marss-riscv-utils
	$ make

This basically installs the following commands (programs): ``simstart``, ``simstop`` and ``simulate`` which will help us to enable and disable simulation mode, before and after running CoreMark respectively. 

At this point we are pretty much ready to run CoreMark.

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

After every ``simstop`` command, the summary of the performance stats is printed on the console and respective stats file is generated based on the timestamp.