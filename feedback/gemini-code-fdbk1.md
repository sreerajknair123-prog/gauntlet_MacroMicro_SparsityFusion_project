# Architectural Co-Design Proposal: Fusing Macro Data Token Deduplication with Micro Bit-Serial Compute Sparsity

This document provides a comprehensive compilation of the system architecture design, literature critique, and microarchitectural specifications for a top-tier computer architecture conference paper (e.g., ISCA, ASPLOS, MICRO). 

---

## 1. Executive Summary & Core Thesis

Next-generation accelerators for Vision-Language Models (VLMs) handling long-form video inputs are severely bound by both computational and memory-bandwidth overheads. Prior state-of-the-art (SOTA) solutions approach efficiency from isolated layers:
1. **Macro Data Sparsity (Token Deduplication):** Compressing incoming temporal and spatial streams by computing unique vectors and replicating results to original sequence positions.
2. **Micro Arithmetic Sparsity (Value-Driven Pruning):** Dynamically skipping bit-level operations within a processing element (PE) fabric based on data importance.

### The Core Problem: The Replication Bottleneck
Naively stitching these two paradigms together into a multi-stage pipeline yields a severe system-level execution trap. While redundant math is minimized, the system introduces a heavy **Replication Bottleneck**. Storing compressed token outputs into a centralized Result Cache and employing a separate multi-ported controller or DMA engine to execute non-contiguous scatter-writes back to memory introduces high area/power overhead, severe DRAM page thrashing, and pipeline desynchronization.

### The Research Thesis
We can eliminate the centralized Result Cache and separate replication engine entirely by building a **Fused Metadata-Driven In-Fabric Forwarding Architecture**. By feeding high-level deduplication pointers directly into a localized, clustered processing fabric, PEs can dynamically coordinate 1-cycle peer-to-peer result forwarding over local register rings, delivering a pre-reconstructed, contiguous sequential write-back stream directly to memory.

---

## 2. Literature Review & Technical Foundation

The proposed architecture acts as a structural bridge between two foundational SOTA accelerator designs introduced at HPCA:

### A. FOCUS: A Streaming Concentration Architecture for Efficient VLMs
* **Core Principle:** Eliminates spatial-temporal visual token redundancy in long-context video inputs.
* **Key Mechanisms:**
  * **Streaming External Concentration (SEC):** Prunes low-relevance vision tokens using semantic context derived from textual prompt attention matrices.
  * **Spatial-Temporal Block Concentration:** Swaps global token sorting for highly localized 3D spatiotemporal sliding-window comparisons to maintain data locality.
  * **Streaming Internal Concentration (SIC):** Performs sub-token vector matching inside General Matrix Multiplication (GEMM) tiles to capture fine-grained pixel drifts.
* **Hardware Profile:** Implemented as a lean streaming unit integrated inside the local accumulation loops of GEMM tiles, preventing redundant token traffic from saturating off-chip DRAM.

### B. PADE: A Predictor-Free Sparse Attention Accelerator
* **Core Principle:** Eradicates the silicon area and latency penalties of separate hardware sparsity predictors in low-bit quantized attention mechanisms.
* **Key Mechanisms:**
  * **Bit-Serial Enabled Stage Fusion (BSF):** Merges the tracking of attention weight evaluation and runtime pruning into a unified hardware cycle.
  * **Bit-Wise Uncertainty Interval-Enabled Guarded Filtering (BUI-GF):** Computes matrix multiplication cycle-by-cycle starting from the Most Significant Bit (MSB). It tracks mathematical confidence intervals at each bitplane and prunes insignificant token pairs mid-execution.
  * **Bidirectional Out-of-Order Execution (BS-OOE):** Mitigates low hardware utilization and pipeline bubbles caused by highly irregular, value-dependent bit-serial execution times via out-of-order instruction scheduling.

---

## 3. The Strawman Baseline & The Replication Bottleneck

To establish a credible evaluation narrative, a multi-stage modular baseline architecture must be characterized and subsequently dismantled.

### Baseline Dataflow Pipeline
1. **Compression Stage:** A front-end `Focus`-like deduplicator separates the incoming video token stream into a dense *Payload Stream* (unique vectors) and an isolated *Metadata Control Stream* (mapping pointers).
2. **Execution Stage:** The dense payload stream is fed into a `PADE`-like core. PEs execute attention layers utilizing value-dependent BUI-GF bit-pruning.
3. **Memoization Stage:** Computed outputs for unique vectors are written sequentially into a centralized, heavily multi-ported SRAM **Result Cache**.
4. **Reconstruction Stage:** A standalone **Replication Engine** reads the metadata tracking queues, queries the Result Cache for unique vector outputs, and issues non-contiguous **scatter-writes** to global memory to rebuild the original dense tensor layout.

### Why the Baseline Fails under Microarchitectural Analysis
* **Asynchronous Desynchronization:** `PADE` introduces highly variable, value-dependent cycle runtimes per token due to its early-termination bitplanes. Unique vectors exit the compute array in a staggered, out-of-order wave. For a centralized replication controller to map these back to the front-end metadata, the hardware requires deep, power-hungry reorder queues and complex tracking scoreboards.
* **Memory and Port Contention:** The centralized Result Cache experiences severe structural hazards as compute PEs try to write new outputs while the replication engine simultaneously issues read requests.
* **DRAM Page Thrashing:** The non-contiguous scatter-writes to global memory disrupt page locality, forcing continuous row-buffer conflicts and dropping effective memory bandwidth utilization below 30%.

---

## 4. Symbiotic Synergy: Cross-Layer Optimization

The integrated architecture moves beyond an arbitrary coupling of two platforms, instead establishing a balanced symbiosis where each layer directly shields the hardware vulnerabilities of the other:

| Layer / Mechanism | Native Hardware Pitfall | The Antidote Provided by Fused Co-Design |
| :--- | :--- | :--- |
| **PADE Micro-Compute** | Extreme cycle-time non-determinism, workload imbalance, and high scheduling overhead (BS-OOE). | **Focus** enforces structural consistency by grouping incoming streams into localized spatiotemporal blocks, ensuring adjacent PEs compute mathematically correlated value ranges, stabilizing PE execution timelines. |
| **PADE Storage Capacity** | Long video context sequences threaten to thrash row-wise intermediate attention tiles (ISTA/RARS). | **Focus** applies aggressive macro-volume token pruning up front, shrinking the absolute working set size so intermediate states safely fit inside the local PE accumulator tiles. |
| **Focus Macro-Filters** | Rigid token/vector boundaries cause an "all-or-nothing" filtering constraint. Minor lighting changes or pixel noise cause checks to fail, letting redundant tokens leak downstream. | **PADE** operates cycle-by-cycle from the MSB. Because nearly identical tokens share identical high-order bitplanes, PADE’s BUI-GF interval tracker dynamically catches and prunes this residual sub-token redundancy mid-execution. |
| **Focus Power Profile** | Primarily optimizes memory bandwidth; hands unique vectors to standard bit-parallel units that waste dynamic switching energy on lower-order zero bits. | **PADE** replaces the backend with a mathematically sparse bit-serial core, scaling arithmetic execution power dynamically based on the exact bitplane significance of preserved data. |

---

## 5. Proposed Fused In-Fabric Architecture & Dataflow

The proposed architecture transitions to a **Fused Metadata-Driven In-Fabric Forwarding Dataflow**.
[Input Stream] ---> [Focus Deduplicator]
|
v
Fused Data + Metadata Control Token
|
v
=============== PE ARRAY =============
|  [Cluster 1]       [Cluster 2]     |
|  +--------+        +--------+      |
|  | PE A   |=======>| PE B   |      |  <-- Intra-Cluster Local Ring Bus
|  | (Comp) | Local  | (Reuse)|      |      (Bypasses Global Result Cache)
|  +--------+ Bypass +--------+      |
===============|======================
|
v
Contiguous Coalesced Stream
|
v
[Sequential DRAM Write]
### Step-by-Step Dataflow Execution
* **Step 1: Token Augmentation:** The front-end deduplicator appends an 8-bit routing tag directly to the data payload vector, creating an *Execution Directing Token* that preserves the macro-redundancy mapping.
* **Step 2: Clustered Distribution:** Tokens are dispatched to localized $4\times4$ PE clusters. Because video temporal redundancy exhibits high "local block sufficiency," duplicate tokens are naturally routed to physically adjacent PEs within the same cluster.
* **Step 3: Asynchronous Execution & Local Buffering:** When a PE receives a token marked `UNIQUE`, it executes bit-serial MAC operations. Upon completion, the result vector is latched into a small, local *Result Reuse Register File ($R^3F$)* embedded inside the local PE routing boundary.
* **Step 4: In-Fabric Forwarding (Eliminating Cache/Scatter):** When an adjacent PE encounters a matching token marked `REUSE`, its local execution controller instantly clock-gates its entire arithmetic multiplier array. It extracts the spatial offset vector from the token's metadata tag and opens an immediate point-to-point neighbor bypass port. The computed output vector is copied over the local cluster ring bus into the reuse PE's output registers in exactly **1 clock cycle**, completely bypassing global SRAM or DRAM.
* **Step 5: Coalesced Streaming Write-Back:** Because duplicate values are populated directly into their correct spatial layouts within the edge registers of the compute fabric, the output matrix emerges fully reconstructed. The array streams out a dense, sequential block that maximizes DRAM page hits.

---

## 6. Detailed Microarchitectural Specifications

### A. Token Frame Bit-Level Format
The databus is expanded to support a unified 1032-bit frame consisting of a 1024-bit vector payload and an 8-bit Metadata Control Tag:
* **Valid Bit ($V$ - 1 bit):** Identifies valid payload or control data.
* **Token Type ($T$ - 1 bit):** `0` = UNIQUE Token (execute full bit-serial math); `1` = REUSE Token (bypass math, trigger local register copy).
* **Spatial Coordinate Offset Vector ($X\text{-}OFF, Y\text{-}OFF$ - 6 bits):** Signed 2's complement integers pinpointing the relative coordinate distance to the donor PE inside the localized cluster (maximum reach of $\pm 3$ PEs).

### B. Augmented PE Block Blueprint
Every PE combines `PADE`'s bit-serial BUI-GF MAC engines with custom local routing extensions:
1. **Input Decoupled Queue (IDQ):** A 16-entry deep, 1032-bit wide FIFO that cushions downstream execution jitter and prevents dynamic bit-serial runtime variances from locking up the global dispatch bus.
2. **Metadata-Driven Execution Controller (MDEC):** Combinational decoding logic ($\approx 400$ gates) that intercepts the head of the IDQ. It determines whether to engage the bit-serial multipliers or to stall arithmetic logic and latch neighbor data.
3. **Result Reuse Register File ($R^3F$):** A dual-ported local storage structure buffering $2 \times 1024\text{-bit}$ vectors (current computed frame + previous frame), acting as the fully decentralized replacement for the global Result Cache.
4. **Modified Dependency Scoreboard:** Extended with a 1-bit `Forwarding_Pending` flag and a 4-bit `Target_Cluster_Mask` to guarantee that a source PE does not overwrite its local register file until all out-of-order neighbor PEs have successfully drained the duplicate values.

### C. Pipeline Phase Resolution
* **Stage 1 (IF/TD - Fetch & Tag Decode):** The MDEC parses the 8-bit tag from the head of the IDQ, isolating token type and physical offset instructions.
* **Stage 2 (DR/IS - Dependency Resolution & Issue):** If `REUSE`, the scoreboard checks the target neighbor PE's execution flag. If the neighbor's data is stable, the instruction issues. If the neighbor is still processing its bitplanes, the local instruction slot stalls independently, allowing other out-of-order independent operations to bypass it.
* **Stage 3 (BS-IC / FWD - Compute or Forwarding Wait):** `UNIQUE` paths cycle through value-dependent bitplanes (4 to 16 cycles). `REUSE` paths clock-gate all arithmetic logic and complete a 1-cycle data transfer from the neighbor port.
* **Stage 4 (LW/IPF - Local Writeback & Inter-PE Forward):** Registers are populated, and credit-return signals are routed back to the donor PE to free up buffer locks.

---

## 7. Evaluation Plan & Key Performance Indicators (KPIs)

To validate the architecture for a top-tier venue, the evaluation must map the proposed in-fabric forwarding mechanisms directly against the challenges of the multi-stage baseline:

| Key Performance Indicator | Baseline Challenge Addressed | Expected Baseline Behavior | Expected Proposed Behavior | Microarchitectural Mechanism |
| :--- | :--- | :--- | :--- | :--- |
| **Memory Stall Cycles & DRAM Efficiency** | **The Replication Bottleneck** (Non-contiguous scatter-writes) | Random, disjointed memory writes causing severe DRAM page thrashing and high bus back-pressure. Bandwidth efficiency $<30\%$. | Complete on-fabric tensor reconstruction. Output streams as a clean, sequential block. Bandwidth efficiency $>90\%$. | **In-Fabric Spatial Re-alignment** eliminates post-compute scattering logic. |
| **Energy Efficiency (Tokens/Joule)** | **Memoization Hardware Overhead** (Power and Caching Costs) | High dynamic power consumption due to continuous multi-port reads/writes across global SRAM banks. | Massive energy reduction ($2\times - 3\times$ improvement) by containing data movement within the PE fabric. | **Localized Register Forwarding Chains** restrict data movement to short, low-capacitance wires. |
| **Silicon Area Footprint ($mm^2$)** | **Memoization Hardware Overhead** (Silicon Area Cost) | Substantial footprint consumed by global multi-ported cache bit-cells, crossbar decoders, and DMA. | Net reduction or parity. Global cache footprint is reclaimed and small fractions are redistributed to PEs. | **Decentralized Scoreboard Controllers** trade heavy centralized SRAM for lean, distributed register files. |
| **PE Utilization & Load Balance** | **Dataflow Rigidity & Asynchronous Desynchronization** | PEs frequently freeze due to global stall signals when variable-latency bit-serial outputs overflow cache queues. | Steady, uninterrupted pipeline flow ($>85\%$ utilization). Local clusters absorb timing variances. | **Input Decoupled Queues (IDQ)** buffer value-dependent bit-pruning execution variances locally. |
| **On-On-Chip Network (NoC) Congestion** | **The Bottleneck-Shifting Risk** (Reviewer 2 Pitfall) | Minimal intra-PE traffic but heavy contention at global centralized memory interface hubs. | No global NoC congestion. Interconnect traffic is strictly confined to immediate cluster sub-networks. | **Local Block Sufficiency Exploitation** maps spatiotemporally adjacent tokens to physically adjacent PEs. |

### Rigorous Methodology Roadmap
1. **Phase 0 (Sanity Check):** Develop a trace-driven Python script within 3 weeks mapping VLM token reuse statistics (from models like Video-LLaVA or LLaVA-NeXT) against simulated memory bus bandwidth to provide the "back-of-the-envelope" proof that the replication stage creates a bottleneck.
2. **Microarchitecture Synthesis:** Model the augmented PE, MDEC, and $R^3F$ configurations in Chisel or SystemVerilog. Synthesize via Synopsys Design Compiler using a commercial node (e.g., TSMC 7nm) to extract precise area ($mm^2$) and power data.
3. **Cycle-Accurate Co-Simulation:** Embed these synthesized hardware metrics into an advanced, cycle-accurate simulator (e.g., Gem5 or a specialized systolic framework) coupled with a detailed memory simulator like Ramulator to generate the final stacked-bar plots demonstrating the complete elimination of replication overhead.