# Macro/Micro Sparsity Fusion for VLM Accelerators

Research ideation and planning repository for a hardware-architecture project on
**fusing two orthogonal forms of sparsity** in next-generation Vision-Language Model
(VLM) accelerators.

## The problem

State-of-the-art VLM accelerators exploit sparsity in two disparate ways:

1. **Input-stream compression (macro)** — deduplicating semantically redundant input
   tokens (e.g. temporally similar video frames), à la **Focus**.
2. **Dynamic compute sparsity (micro)** — pruning insignificant arithmetic inside the
   compute kernels (e.g. bit-serial guarded filtering in attention), à la **PADE**.

Naively combining them forces a **"compute-once, replicate-result"** pipeline: compute
unique vectors, cache the results, then scatter them to their final output positions.
That scatter-and-cache stage introduces a **Replication Bottleneck** — irregular
scatter-write memory traffic, result-cache area/power overhead, and dataflow rigidity
that hides redundancy information from the compute PEs.

**Goal:** quantify the replication overhead of the memoization baseline, then design an
architecture that fuses redundancy metadata *directly into the compute micro-architecture*
— eliminating the explicit result-replication engine and capturing reuse within the
compute fabric itself. Figures of merit: area/power/traffic/latency of the replication
path, total bit-ops and off-chip movement, and end-to-end Tokens/Joule and Tokens/second
on video VLM workloads.

See [`problem-defn/problem.txt`](problem-defn/problem.txt) for the full problem statement.

## Repository layout

```
problem-defn/                 Full problem statement (context, key problem, figures of merit)
references/
  available_repos.md          Survey of reusable open-source artifacts (Focus, PADE, etc.)
related-papers/               Source papers (FOCUS.pdf, PADE.pdf)
related-papers-distillations/ Text/HTML distillations of those papers for quick reference
feedback/                     External review notes on the approach
CLAUDE.md                     Working prompt / context for AI-assisted ideation
claude_thoughts.md            Running ideation notes
```

## Key references

- **Focus** (HPCA 2026) — vector-granular similarity gather/scatter; open-source
  full-stack artifact (algorithm traces, ScaleSim/CACTI/DRAMsim3 simulator, RTL).
  See `references/available_repos.md` for how its simulator can host custom PADE PEs.
- **PADE** (HPCA 2026) — bit-wise uncertainty-interval guarded filtering; the target
  fine-grained compute micro-architecture, with scoreboard-assisted result-reuse PEs.

## Status

Early-stage: problem definition, literature distillation, and reuse strategy are in
place; next steps are baseline reproduction and architecture ideation toward an
integrated, replication-free simulator.
