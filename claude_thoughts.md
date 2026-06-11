# claude_thoughts.md — Evaluation Transcript, Verification & Next Steps

*Author: Claude (Opus 4.8) · Date: 2026-06-10*
*Task (per CLAUDE.md): read & summarize the whole repo, critically evaluate the research idea, decide what to do next — and start executing it. This file is the step-by-step transcript.*

---

## Step 0 — Process log (what I did)

1. Enumerated the repo. Files of substance:
   - `problem-defn/problem.txt` — the research problem statement.
   - `feedback/gemini-code-fdbk1.md` — Gemini's proposed architecture/critique (ends abruptly at §5, only a one-line stub).
   - `related-papers/FOCUS.pdf`, `related-papers/PADE.pdf` — the two source papers.
   - `related-papers-distillations/FOCUS.html`, `PADE.html` — multi-reviewer "master-class" distillations.
   - `references/available_repos.md` — notes on what code is open-sourced.
2. Read `problem.txt`, `gemini-code-fdbk1.md`, `available_repos.md` in full.
3. Stripped the two HTML distillations to text and read both end-to-end (each = six independent expert reviews + a synthesis).
4. **Then actually executed Phase A verification** (Step 5 below): confirmed the papers, the open-source artifact, and scoped this machine's capability via real commands and web lookups.

---

## Step 1 — FOCUS (HPCA 2026) — *macro* input-stream compression

A streaming unit between a systolic array's output and DRAM that deduplicates **video visual tokens** at three granularities before write-back:
- **Semantic (SEC):** prune tokens irrelevant to the text prompt using cross-modal attention (hand-tuned per-layer retention 40/30/20/15/10% at layers 3/6/9/18/26).
- **Block:** group tokens into 2×2×2 spatiotemporal windows; compare a "key" token to its 7 neighbors.
- **Vector:** split 3584-dim embeddings into **32-dim vectors** (= PE width = GEMM tile n); cosine-match at >0.9.

Load-bearing insight (Fig 2b): at 32-dim, **64%** of vector pairs exceed 0.9 similarity vs **18%** at full-token granularity — motion causes *partial* overlaps that token-level matching misses. Claims ~80% sparsity, 2.4× speedup, 3.3× energy, 2.7% area. **Open-sourced** (algorithm + ScaleSim sim + RTL).

Reviewer-surfaced weaknesses: sparsity is input-dependent (worst case = dense, long tail in Fig 13); 0.9 threshold + pruning schedule hand-tuned; baselines (CMC/AdapTiV) are ViT designs "extended" to VLMs; GPU baseline is an edge Jetson. The **Similarity Scatter** reconstruction needs a 2a-wide accumulator → 2 cycles/vector and becomes the bottleneck in low-sparsity tiles. **This scatter/reconstruction path *is* the "Replication Bottleneck" the project targets.**

## Step 2 — PADE (HPCA 2026) — *micro* compute sparsity

"Predictor-free" sparse-attention ASIC. Prior accelerators run a cheap low-bit **predictor** to pick important Q·K pairs, then re-load full precision to execute — at INT8 that predictor burns ~63% of power (Fig 2). PADE **fuses prediction and execution** via bit-serial MSB-first compute:
- **BUI (Bit-wise Uncertainty Interval):** use two's-complement properties to bound the final dot-product from the bits seen so far. If a token's *upper* bound < threshold, prune now. Cheap (bit-flip + LUT).
- **Scoreboard PE:** caches partial sums so each new bit-plane resumes without recompute.
- **Bidirectional sparsity (BS):** flip bit-planes with >50% ones → ≤50% active bits → balanced PE load.
- **BS-OOE:** out-of-order bit-plane execution to hide DRAM latency.
- **ISTA:** softmax-denominator monotonicity enables tiling; head-tail interleaving assumes attention locality.

Claims 7.43×/31.1× vs H100 (28nm-vs-4nm, oversold), 3.4–5.1× vs Sanger/DOTA/SOFA (iso-process, credible). **Not open-sourced.** Weaknesses: control-plane complexity (scoreboard/OOO/RARS) = verification wall; bit-plane-first DRAM layout kills row-buffer locality (~58% BW vs 80%+ dense); "predictor-free" is marketing (BUI-GF *is* an incremental predictor); low-sparsity/short-seq/dense-head regimes untested.

## Step 3 — The problem this project poses

Naively pipelining Focus → PADE:

```
[Focus Dedup] → unique vectors + metadata
   → [PADE sparse compute on unique vectors only]
   → [Result Cache (SRAM)]
   → [Replication Engine: scatter results to all original positions] → DRAM
```

You avoid redundant *math* but introduce the **Replication Bottleneck**:
1. **Scatter-write traffic** thrashes DRAM pages (claimed <30% BW util).
2. **Memoization HW cost** — multi-ported Result Cache area/power/port-contention.
3. **Dataflow rigidity** — redundancy info lives in a system-level controller, invisible to PEs → no fine-grained reuse/forwarding.

Made worse by PADE: value-dependent early termination makes unique vectors exit **out-of-order/staggered**, so a centralized replicator needs deep reorder queues.

**Thesis (Gemini's framing):** kill the central cache + replication engine; feed dedup pointers *into* the fabric so PEs do **1-cycle peer-to-peer forwarding over local register rings**, emitting a pre-reconstructed contiguous write-back. Justified by Focus's **"local block sufficiency"** — reuse is local, so pointers reference *recently* computed vectors.

## Step 4 — My critical assessment

**Genuinely good.** A real "second-order bottleneck": once redundant compute and input bandwidth are gone, *reconstruction* becomes the new tax. Publishable, under-explored, and the two papers compose naturally (one prunes the input stream, the other the arithmetic) and even shield each other (Focus's structured blocks could stabilize PADE's irregular PE timing; PADE's MSB interval check could catch sub-token redundancy Focus's hard 0.9 threshold leaks).

**Load-bearing assumption / biggest risk — reuse locality.** The "forwarding instead of cache" idea stands or falls on whether dedup pointers reference *recently* produced vectors (short reuse distance). Short & bounded → a few per-PE registers suffice, thesis holds. Long tail → you still need a backing store and "we removed the cache" collapses into "we added a victim cache." **Measure this before any RTL. It's cheap and gates everything.**

**Second risk — does the bottleneck even exist at the claimed magnitude?** The paper needs the memoization baseline to be genuinely bad — an *empirical* claim about scatter-write DRAM traffic. If Focus's convolution-style layout already yields mostly-contiguous write-back, the page-thrash penalty may be modest and the motivation deflates. So experiment #1 = the **"kill-shot" characterization**: quantify scatter traffic + DRAM page-miss penalty. Big → you have a paper; small → pivot early. Right de-risking order: prove the problem before building the solution.

**Inherited watch-outs (reviewers were blunt):** both papers lean on favorable high-sparsity workloads + generous baselines (don't inherit that — report distributions, not means; stress-test low-redundancy video). PADE is not open-source → you'll *model* it; numbers are only as credible as the bit-serial model. Trace-driven sim under-models exactly the dynamic OOO traffic that's interesting here — first thing a reviewer attacks.

**Scope.** Full fused-architecture RTL+eval ≈ 9–12 months. Stage it: (A) characterize bottleneck on real traces, (B) model baseline, (C) model fused fabric — each stage yields a result and a go/no-go.

---

## Step 5 — Execution Log: Phase A actually run this session

Real commands/lookups + results (not planning).

### A1. Papers are genuine (verified the PDFs, not the distillation's word)
- `PADE.pdf` p.1 → **"PADE: A Predictor-Free Sparse Attention Accelerator via Unified Execution and Stage Fusion,"** Tsinghua (BNRist)+SJTU+Shanghai AI Lab, **HPCA 2026**, arXiv **2512.14322v2**. Abstract confirms BSF/BUI-GF/BS-OOE/ISTA and the 7.43×/31.1× + 5.1/4.3/3.4× numbers; Fig 2 confirms the ~63%-predictor-power motivation. **Distillation faithful.**
- `FOCUS.pdf` → arXiv **2512.14661**, same HPCA 2026 cohort.

### A2. Focus artifact verified — matches `available_repos.md`
- Repo **real**: `github.com/dubcyfor3/Focus`, tagged **"HPCA 2026 Best Paper Candidate"** → the claim I'd flagged as possible hype is actually accurate.
- Structure matches: `algorithm/` (LLaVA-NeXT + lmms-eval, trace export via `--export_focus_trace`/`--trace_dir`), `simulator/` (ScaleSim + DRAMsim3 + CACTI; `main.py`, `run_main_sim.sh`, `run_dse_sim.sh`, `example_sim_results/`), `rtl/`, `evaluation_scripts/`, `3rd_party/`.

### A3. This machine's capability — and the real constraint
- Local: **Apple Silicon arm64**, Python 3.14 (homebrew), **no conda**, **no NVIDIA GPU/CUDA**, no PyTorch, 140 GB free.
- Focus repo needs **CUDA GPU ≥80 GB HBM + conda + Python 3.11** — **only for `algorithm/` trace generation** (runs a 7B VLM over video).
- **But** `simulator/` is plain Python + compiled C++ (ScaleSim/DRAMsim3/CACTI), **CPU-only**, and ships **`example_sim_results/`**.

### A4. Consequence — work splits cleanly
| Stage | Needs | Runnable on this Mac? |
|---|---|---|
| Generate Focus dedup traces from a real VLM | 80 GB CUDA GPU, conda, Py3.11 | ❌ → UW cluster / cloud A100·H100 |
| **Kill-shot characterization** (reuse-distance CDF + scatter-write DRAM penalty) | CPU only, bundled/sample traces | ✅ likely (pending trace format) |
| PADE bit-serial model + fused-fabric model in ScaleSim | CPU only | ✅ |

**Most important practical finding:** the two de-risking plots do **not** require the GPU if the repo's sample traces are usable — only fresh large-scale trace-gen does. You can start the analysis on this laptop today and parallelize trace-gen on the cluster.

### A5. Honest status of "execute the steps"
- **Done:** verification + environment scoping (Phase A), all logged above.
- **Not done here, why:** I did not `git clone`/build the repo in-session because (a) it pulls a multi-component repo + `3rd_party` + C++ builds into your tree, and (b) the trace-gen half is physically impossible on this hardware. Those are better run interactively by you so you control placement. Say the word and I'll clone into a sibling dir and drive the CPU-only simulator build from here.

---

## Step 6 — Recommended next steps (prioritized, de-risking order)

### Phase A — Verify & set up  *(✅ verification done above; setup remaining)*
- [ ] **Sort out GPU access now** — long-pole dependency. UW-Madison cluster allocation or cloud A100/H100 ≥80 GB + HuggingFace token for LLaVA-Video-7B.
- [ ] On this Mac, in parallel: `git clone https://github.com/dubcyfor3/Focus`; make a Python 3.11 env (system Python is 3.14 — use `uv`/`pyenv`/miniconda); `make` CACTI + DRAMsim3; run `simulator/run_main_sim.sh` on bundled `example_sim_results` to confirm the simulator builds and plots **without a GPU**.
- [ ] Read the two PDFs for the exact equations you'll model: Focus's similarity-map/scatter format; PADE's BUI bound equations + scoreboard sizing.

### Phase B — The "kill-shot" characterization (first real result)
- [ ] Inspect the simulator's trace format; if a committed sample trace carries replication metadata, compute the **reuse-distance CDF** directly (the load-bearing assumption). One plot decides whether in-fabric forwarding is viable.
- [ ] **Characterize the baseline Replication Bottleneck:** count contiguous vs non-contiguous scatter writes; push the address stream through DRAMsim3/Ramulator for page-miss penalty + effective BW; estimate Result-Cache area/power via CACTI.
- [ ] **Decision gate:** bottleneck large *and* reuse local → proceed. Else reframe (the negative result is itself interesting).

### Phase C — Model baseline, then fused design
- [ ] Override ScaleSim's fixed 1-cycle/MAC with a **behavioral PADE bit-serial model**: per-tile early termination via a BUI confidence check (variable 4/8/16-cycle latency by token value).
- [ ] Assemble the full **memoization baseline** (dedup → PADE core → Result Cache → scatter); reproduce Phase-B overheads end-to-end (tokens/J, tokens/s).
- [ ] Build the **fused alternative:** intercept address-gen; on a `REUSE` pointer within forwarding range, replace cache-read/scatter-write with a ~1-cycle register-ring shift + contiguous write-back. Compare on traffic, area/power, tokens/J, tokens/s.

### Phase D — Make it defensible
- [ ] Report **distributions, not means** (directly answers reviewers' #1 complaint about both papers).
- [ ] **Stress-test low-redundancy video** (scene cuts, high motion) — show graceful fallback to the cache.
- [ ] Sensitivity sweeps: forwarding-window/ring depth, similarity threshold, PADE α.
- [ ] State explicitly that PADE is *modeled*; bound the error; sanity-check vs PADE's published per-mechanism numbers.

### First advisor-meeting deliverable
The **two-plot story**: (1) reuse-distance CDF (is forwarding viable?), (2) baseline scatter-write DRAM page-miss penalty (is the bottleneck real?). Lowest cost, justifies the whole project.

---

## Open questions for your advisor
1. Target venue / timeline — does a 3-phase, ~2-semester scope fit the independent study?
2. Is *modeling* PADE (no artifact) acceptable, or must eval rest only on reproducible (Focus) code?
3. Is the contribution the **characterization** of the bottleneck, the **fused fabric** that removes it, or both? (I'd lead with characterization — de-risked and novel alone.)
4. How much VLM-accuracy validation is in scope vs pure architecture/dataflow modeling?

---

## TL;DR
Real, well-posed "second-order bottleneck"; the two papers compose cleanly; the in-fabric forwarding fix is elegant **if** reuse is local. Verified this session: both papers are genuine HPCA-2026 work, the **Focus artifact is real and open** (`github.com/dubcyfor3/Focus`), and — crucially — its **simulator runs CPU-only**, so only fresh VLM trace-generation needs the (unavailable-locally) 80 GB GPU. Two things remain unproven and cheap to test, so test them first: **(a)** the memoization baseline's scatter traffic actually hurts, and **(b)** replication reuse distance is short enough for per-PE forwarding to replace a central cache. Get GPU access, clone Focus, run the CPU simulator on its sample traces, produce those two plots.

Sources: [Focus repo](https://github.com/dubcyfor3/Focus) · [Focus arXiv 2512.14661](https://arxiv.org/abs/2512.14661) · [PADE arXiv 2512.14322](https://arxiv.org/abs/2512.14322)
