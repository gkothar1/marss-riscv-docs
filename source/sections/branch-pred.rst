======================
Branch Prediction Unit
======================

Branch prediction unit (BPU) is configurable and can run a simple bimodal
predictor or complex 2-level adaptive predictors like GShare, GSelect, GAg, GAp, PAg, or PAp. This section describes the major structures used in BPU and their interaction with the CPU pipeline.

.. figure:: ../figures/bpu.*
   :figwidth: 400 px
   :align: center

   BPU High-level Overview

Branch Target Buffer
---------------------

Branch Target Buffer (BTB) is modeled as a set-associative data structure. Supported eviction policies are random eviction and bit PLRU.

.. note::
   For a simple bi-modal predictor, we keep the prediction bits in a separate Branch History Table (BHT).

Return Address Stack
----------------------
BPU consists of an optional return address stack (RAS), which keeps track of the last ``N`` function return addresses, where ``N`` is the number of entries in RAS. The return address is pushed onto RAS from the decode pipeline stage if the decoded instruction becomes a function call. Similarly, if the decoded instruction turns out to be a function return, the address is popped from RAS and forwarded to the fetch stage.

Two-Level Adaptive Predictor
----------------------------

Level 1 of the adaptive predictor consists of a Global History Table (GHT), and level 2 consists of a Pattern History Table (PHT). Each GHT entry consists of a PC and History Register (HR) of ``N`` bits. Each PHT entry consists of a PC and an array of :math:`2^N` 2-bit saturating up-down counters. Based on the number of GHT and PHT entries, four different prediction schemes are possible, shown in the following table.

================ ================== ===============
**GHT entries**    **PHT Entries**   **Predictor**
================ ================== ===============
= 1              = 1                GAg
= 1              > 1                GAp
> 1              = 1                PAg
> 1              > 1                PAp
================ ================== ===============

Interaction with the CPU Pipeline
---------------------------------

CPU pipeline interacts with the Branch Prediction Unit in mainly four ways as follows:

* **Probing the branch predictor**

   * In the Fetch stage, the branch predictor is probed using the virtual address of the instruction

   * If the address is present in the BTB, then probe returns hit

   * In case of conditional branches, for two-level adaptive predictor, address must also be present in GHT (For PAg and PAp based schemes) and PHT (For GAp and PAp based schemes)

* **Acquiring the target address**

   * On branch prediction unit hit, fetch stage requests the target address from the BPU

   * For unconditional branches, the target is immediately returned from the BTB entry

   * For conditional branches, only if the prediction is taken, the target is returned from the BTB entry

* **Adding entries to branch predictor structures**

   *  A BTB entry is created for a branch instruction missed in the BTB when the branch is resolved.

   * In case of two-level adaptive predictor, required entries are also created in GHT and PHT based on the prediction scheme, for conditional branches

* **Updating branch predictor entries**

   * Branch Prediction unit is probed again in memory stage (or ``Int-ALU`` in the out of order pipeline) after the branch resolves

   * If probe misses, then the updates are skipped

   * If probe results in a hit, BTB entry of the branch is updated with the current target address

   * For the conditional branches, prediction counters are updated based on the latest branch outcome

Timings
-------

* Probe and target address retrieval: **one cycle** (overlaps with the TLB and cache probes)

* Add and update entry: **one cycle each** (overlaps with relevant stage processing)