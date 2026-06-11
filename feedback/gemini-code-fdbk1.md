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