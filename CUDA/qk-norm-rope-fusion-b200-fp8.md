# PyTorch compile + CUDA learnings (from vLLM issue #33295)

## Date

- 2026-02-06

## Context

- Issue: QK Norm + RoPE fusion did not trigger on B200 for
  `Qwen3-30B-A3B-FP8`.
- Symptom: fusion log showed `Fused QK Norm+RoPE on 0 sites`
  instead of `1`.

## Core learning

The fusion pass matched a **graph shape**, not just math
equivalence.

### Expected FX shape (working)

One split node with 3 users:

```text
[qkv]
  |
[split_with_sizes]   (users = 3)
   |        |        |
[get0]    [get1]    [get2]
  |         |         |
 q path    k path    v path
```

### Actual B200+FP8 shape (broken)

Three equivalent split nodes, each with 1 user:

```text
[qkv] -> [split_1] -> [get0] -> q path
[qkv] -> [split_2] -> [get1] -> k path
[qkv] -> [split_3] -> [get2] -> v path
```

Math was equivalent, but pattern matching failed because the
structure differed.

## Terminology (quick)

- **FX graph**: PyTorch's intermediate graph (a flowchart of
  tensor ops) used by compiler passes.
- **RoPE**: Rotary Positional Embedding, which injects
  token-position info into attention `q` and `k`.
- **User / Consumer**: A node that reads another node's output.
  - Example: if `split` feeds 3 `getitem` nodes, then `split`
    has 3 users.

## Fix summary

- Extracted a standalone `SplitCoalescingPass` (in
  `vllm/compilation/passes/utility/split_coalescing.py`) that
  merges duplicate `split_with_sizes` nodes into one canonical
  node.
- Registered it in the `PostGradPassManager` to run immediately
  before `QKNormRoPEFusionPass`.
- Removed the coalescing logic from `QKNormRoPEFusionPass`
  itself, keeping concerns separated.
- This preserves semantics but stabilizes graph shape across
  hardware/compiler variance.

## Why this matters

- `torch.compile` pipelines can produce slightly different FX
  shapes by hardware, dtype, and optimization path.
- A robust pass often needs normalization/canonicalization before
  strict pattern matching.

## Practical debugging workflow for compile passes

1. Dump pre/post pass graphs.
2. Compare matched vs unmatched subgraphs structurally.
3. Identify shape assumptions in matcher patterns (like
   `_users=3`).
4. Normalize graph or relax matcher safely.
5. Add a regression test reproducing the failing structure.

## Deep-dive documents

For detailed explanations with diagrams:

1. [01-torch-compile-pipeline.md](01-torch-compile-pipeline.md)
   -- how Python becomes GPU code
2. [02-fx-graphs-deep-dive.md](02-fx-graphs-deep-dive.md)
   -- how FX graphs work in detail
3. [03-kernel-fusion-and-the-split-coalescing-fix.md](03-kernel-fusion-and-the-split-coalescing-fix.md)
   -- the bug, the fix, and the testing strategy

## High-quality resources to learn more

### PyTorch compile stack (official)

- torch.compiler overview:
  https://docs.pytorch.org/docs/stable/user_guide/torch_compiler/torch.compiler.html
- torch.compile tutorial:
  https://docs.pytorch.org/tutorials/intermediate/torch_compile_tutorial.html
- torch.compile programming model:
  https://docs.pytorch.org/docs/stable/user_guide/torch_compiler/compile/programming_model.html
- torch.compile troubleshooting:
  https://docs.pytorch.org/docs/stable/user_guide/torch_compiler/torch.compiler_troubleshooting.html
- torch.fx docs:
  https://docs.pytorch.org/docs/stable/fx.html
- torch._logging / TORCH_LOGS:
  https://docs.pytorch.org/docs/stable/logging

### CUDA + profiling (official NVIDIA)

- CUDA C Programming Guide:
  https://docs.nvidia.com/cuda/cuda-programming-guide/index.html
- CUDA C++ Best Practices Guide:
  https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html
- Nsight Systems docs:
  https://docs.nvidia.com/nsight-systems/index.html
- Nsight Compute docs (latest):
  https://docs.nvidia.com/nsight-compute/2024.2/index.html
- CUPTI docs:
  https://docs.nvidia.com/cupti/

## Suggested study order

1. `01-torch-compile-pipeline.md` (this repo)
2. `02-fx-graphs-deep-dive.md` (this repo)
3. `03-kernel-fusion-and-the-split-coalescing-fix.md` (this repo)
4. `torch.compile` tutorial (link above)
5. `torch.compile` programming model (link above)
6. FX docs (`torch.fx`) (link above)
7. CUDA Programming Guide (programming model + memory hierarchy)
8. Nsight Systems + Nsight Compute docs for profiling practice
9. torch.compile troubleshooting + TORCH_LOGS in real debugging
   sessions
