This search provides a major technical breakthrough for your project implementation. One of the two core frameworks is fully open-sourced with a comprehensive, full-stack artifact repository that directly aligns with your planned workflow.

Below is the breakdown of available repositories, artifact documentations, and architectural clues that you can leverage immediately to build your integrated simulator.

---

### 1. The GitHub Artifact: **Focus** (HPCA 2026 Best Paper Candidate)

The entire full-stack implementation for the **Focus** paper is open-sourced on GitHub. It includes the exact baseline simulators you need and directly mirrors the 3-step evaluation workflow identified in your project's *Path to Paper*.

* **Repository URL:** [https://github.com/dubcyfor3/Focus](https://github.com/dubcyfor3/Focus) (or accessible via Zenodo Archive [10.5281/zenodo.17851347])
* **Significance to Your Project:** This repository completely removes the 9-month infrastructure risk from your timeline. Instead of building a custom simulator from scratch, you can clone this repo and inject your custom `PADE` PE blocks directly into its existing framework.

#### Repository Breakdown & Included Infrastructure:

The `Focus` repo incorporates multiple modular, industry-standard simulators already wired together:

1. **`algorithm/` (Trace Generation):** Built on top of `lmms-eval` and `LLaVA-NeXT`. It includes out-of-the-box scripts to evaluate Vision-Language Models (such as *LLaVA-Video*, *LLaVA-OneVision*, *MiniCPM-V*, and *Qwen2.5-VL*) on major benchmarks (*VideoMME*, *MVBench*, *MLVU*). Crucially, **it features built-in sparse-trace generation** that dumps VLM visual-token activity directly to files.
2. **`simulator/` (Cycle-Accurate Hardware Modeling):** Features an execution modeling backend built on **ScaleSim** (a popular cycle-accurate systolic array simulator), **CACTI** (for SRAM area/power modeling), and **DRAMsim3** (for cycle-accurate memory subsystem/DRAM traffic modeling).
3. **`rtl/` (Hardware Modules):** Includes physical Verilog/SystemVerilog implementations of the systolic array compute blocks and the macro concentration modules.

---

### 2. Implementation Strategy for **PADE** (HPCA 2026)

While the official standalone repository for **PADE** has not been broadly open-sourced under a unified public GitHub handle yet (portions of the underlying dynamic sparse attention algorithms are tracked through academic branches like the Tsinghua/Shanghai Jiao Tong ML frameworks), the paper's documentation reveals exactly how its microarchitectural pieces are mapped.

#### Bridging PADE into the Focus Simulator Framework:

To model PADE's *Bit-wise Uncertainty Interval-Enabled Guarded Filtering (BUI-GF)* and *Bidirectional Out-of-Order Execution (BS-OOE)* within the cloned `Focus` environment, modify the core `scalesim/` directory inside the Focus repository:

* **Modifying ScaleSim for Bit-Serial Latency:** In standard ScaleSim, a GEMM array assumes a fixed, uniform 1-cycle execution per bit-parallel MAC tile. You must override the compute-cycle calculation logic in ScaleSim's cycle tracker. Read the data matrix bit-by-bit from the `Focus` algorithmic traces. Implement a behavioral function representing PADE's BUI-GF confidence equation: if the numerical variance drops below the threshold, force that specific tile simulation row to terminate early (e.g., changing its latency from a rigid 16 cycles down to 4 or 8 cycles depending on the token value).
* **Integrating the Forwarding Logic:** ScaleSim tracks cycle-by-cycle reads from its virtual SRAM buffers. To evaluate the proposed fused project, intercept the memory address generation stage. If a token frame contains a `REUSE` pointer tag, force ScaleSim to bypass its standard SRAM read/write energy counting for that clock cycle, and instead log a fractional 1-cycle register shift cost.

---

### 3. Critical Peer Documentation & Technical Clues

Analyzing the technical specifications and review summaries of both papers yields critical design insights for the project:

#### Focus Implementation Clues

* **Hardware Footprint:** The authors document that the entire standalone Focus streaming hardware module occupies **only 2.7% of the total systolic array area**. This means your distributed baseline is starting with a highly efficient, compact macro-pruning footprint.
* **Sparsity Scaling:** Focus extracts an average of **80% to 83% computational sparsity** at the token/block level on long-context video workloads. This is a massive confirmation for your project's thesis: because Focus is filtering out over 80% of the raw visual stream, the leftover unique tokens represent high-density compute points, leaving the remaining 80% as prime candidates for your *In-Fabric 1-Cycle Forwarding* mechanism.
* **Data Layout:** Focus adopts a custom, **convolution-style layout and sliding-window sweep** (resembling a 3D CNN kernel movement) to process tokens out of the compute core. This means your intra-cluster routing should explicitly optimize for a 2D mesh sliding-window dataflow rather than a global arbitrary routing framework.

#### PADE Benchmark Metrics (For Your Evaluation Baseline)

* When evaluating your integrated design, your hardware baseline numbers must match or outperform the standalone metrics documented by the PADE authors. PADE demonstrates a **7.43× speedup and 31.1× higher energy efficiency over an Nvidia H100 GPU** on dynamic sparse attention tasks.
* When pitching your paper against other SOTA hardware accelerators, you should include **Sanger, DOTA, and SOFA** as comparative baseline bars in your evaluation plots, as PADE officially establishes its performance gains relative to these three specialized sparse transformers (achieving up to a 5.1× energy reduction against them).

### Actionable Next Steps for the Student

1. **Clone the Repo:** Git clone `https://github.com/dubcyfor3/Focus.git` and initialize its `scalesim` and `DRAMsim3` submodules.
2. **Generate Baseline Traces:** Run the provided `algorithm/run_eval` scripts using an open-source VLM checkpoint (like LLaVA-Video) on a small limit-set of a video dataset to verify that the trace logs generate properly.
3. **Inject the Kill-Shot Back-of-the-Envelope Script:** Use the generated trace logs to count how many non-contiguous scatter operations are requested versus contiguous operations. Run this footprint through the included `DRAMsim3` configuration file to immediately calculate your DRAM page-miss penalty. This provides the exact validation demanded in your first meeting checklist.