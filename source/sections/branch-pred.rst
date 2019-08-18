======================
Branch Prediction Unit
======================

Branch prediction unit (BPU) can be configured to run a simple bimodal
predictor or a complex 2-level adaptive predictors like GShare, GSelect, GAg, GAp, PAg or
PAp. This section describes the major hardware structures used in BPU
and their interaction with the CPU pipeline.

Branch Target Buffer
--------------------

Branch Target Buffer (BTB) is modeled as a set-associative data structure. Supported
eviction policies are random eviction and LRU using hot-cold bits.

.. note::
   For a simple bi-modal predictor, we keep the prediction bits as a part of BTB entry itself, rather than keeping them in a separate Branch History Table (BHT).

Two-Level Adaptive Predictor
----------------------------

Level 1 of adaptive predictor consists of a Global History Table (GHT) and
level 2 consists of a Pattern History Table (PHT). Each GHT entry consist
of a PC and History Register (HR) of ``N`` bits. Each PHT entry consist of a
PC and an array of :math:`2^N` 2-bit saturating up-down counters. Based on the number of entries in GHT and PHT, 4 different prediction schemes are possible shown in the following table.

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

CPU pipeline interacts with the Branch Prediction Unit in mainly 4 ways as
follows:

* **Probing the the branch predictor**

   * In Fetch stage, branch predictor is probed using the virtual address of the instruction

   * If the address is present in the BTB, then probe returns hit

   * In case of conditional branches, for two-level adaptive predictor, address must also be present in GHT (For PAg and PAp based schemes) and PHT (For GAp and PAp based schemes)

* **Acquiring the target address**

   * On branch prediction unit hit, fetch stage requests the target address from the BPU

   * For unconditional branches, the target is immediately returned from the BTB entry

   * For conditional branches, only if the prediction is taken, the target is returned from the BTB entry

* **Adding entries to branch predictor structures**

   * If the instruction missed in the BPU during fetch stage and is decoded as a branch, BTB entry is created for this instruction in decode stage

   * In case of two-level adaptive predictor, required entries are also created in GHT and PHT based on the prediction scheme, for conditional branches

* **Updating branch predictor entries**

   * Branch Prediction unit is probed again in memory stage (or ``Int-ALU`` in the out of order pipeline) after the branch resolves

   * If probe misses, then the updates are skipped

   * If probe results in hit, BTB entry of the branch is updated with new target address

   * For the conditional branches, prediction counters are updated based on the latest branch outcome

Timings
-------

* Probe and target address retrieval: **one cycle** (overlaps with the TLB and cache probes)

* Add and update entry: **one cycle each** (overlaps with relevant stage processing)