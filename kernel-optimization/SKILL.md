---
name: nvidia-kernel-optimization
description: "Use when developing, profiling, or optimizing NVIDIA CUDA/Triton kernels: building representative harnesses, validating correctness, choosing sound benchmark methodology, using Nsight Systems and Nsight Compute, diagnosing bottlenecks, running target-GPU sweeps, and deciding whether to gate, integrate, saturate, or reject a kernel candidate."
---

# NVIDIA Kernel Optimization

Use this workflow for CUDA/Triton kernel work on NVIDIA GPUs. The loop is:
representative harness, correctness check, target-GPU baseline, Nsight
Systems/Compute profile, targeted structural change, parameter sweep, then
integration validation.

## Rules

- Use evidence from the target NVIDIA GPU, driver, CUDA runtime, clock policy,
  and Python environment. Treat unrelated local GPU timings as exploratory.
- Work in the project checkout that matches the code under test.
- Verify CUDA before profiling:

```bash
export PYTHON=<path-to-python>
export CUDA_VISIBLE_DEVICES=0
"$PYTHON" - <<'PY'
import torch
print(torch.__version__, torch.cuda.is_available(), torch.cuda.device_count())
if torch.cuda.is_available():
    print(torch.cuda.get_device_name(0))
PY
nvidia-smi
```

- Keep clocks, power policy, competing GPU jobs, and thermal state consistent
  across baseline and candidate runs.
- Record the timing method, warmup count, measured iteration count, repeat
  count, cache policy, GPU name, and CUDA driver/runtime versions with every
  benchmark result.
- Benchmark cache policy must match the production workload. For large streaming
  inputs or weights, use cold-L2 by default. Hot-cache timing may exist only as
  an explicitly labeled diagnostic.
- Prefer hardware-level kernel timing such as CUPTI when the project already
  supports it. Do not silently switch timing methods; if the preferred timing
  method is unavailable, ask the user before using CUDA events and label the
  result accordingly.
- Do not trust a kernel-bench win until correctness, representative timing, and
  Nsight evidence agree.
- Do not add fallback or try/except runtime paths without explicit user
  approval.
- Keep runtime defaults unchanged until the candidate is accepted.

## Harness Requirements

Every kernel candidate should have a standalone benchmark or test harness that
supports:

- direct correctness checks against the reference path
- target shape, dtype, layout, and stride arguments
- explicit reference and candidate modes
- reference checking enabled for acceptance runs
- warmup and measured iteration controls
- repeat count or enough iterations to report stable median and standard
  deviation
- cache behavior matching the production workload
- a mode that exercises the exact replaced boundary for fused or replacement
  work
- `--profile-capture` or an equivalent CUDA profiler API capture window
- a printed reproducer command or enough logged arguments to rerun the exact
  benchmark case

`--profile-capture` must run warmup first, call `cudaProfilerStart()`, push an
NVTX range with the target name and shape, run exactly the requested target
iterations, synchronize before popping the range, then call
`cudaProfilerStop()`.

Do not profile a kernel that is already known to be numerically invalid for the
target dtype or shape. For runtime-facing replacements, also run the smallest
integration check that exercises the real call path.

## Benchmark Discipline

Use median latency as the primary microbenchmark statistic and standard
deviation as the stability signal. When operation counts are meaningful, also
report achieved TFLOPS and effective bandwidth. Treat large standard deviation,
wide repeat-to-repeat spread, or drift across runs as a benchmark problem before
calling a kernel faster.

For unstable timing, first increase warmup and measured iterations, then verify
cache policy, clock stability, synchronization boundaries, and competing GPU
work. For cold-cache workloads, use buffer rotation or explicit L2 eviction in
the harness so repeated iterations do not accidentally become warm-cache tests.

Compare against every relevant baseline in the same process or session when
possible. If a candidate changes numerical precision or accumulation order,
record the tolerance policy and inspect mismatch statistics before treating a
latency win as valid.

## Architecture And API Checks

State the supported compute capabilities for every candidate. Compile only for
architectures the kernel actually supports, and make tests skip unsupported GPUs
with a clear reason instead of failing deep inside JIT, import, or launch code.
Do not assume future architectures are compatible with architecture-specific
instructions or layouts.

For public or runtime-facing APIs, keep allocation out of the measured hot path:
accept caller-provided output/workspace buffers when the project style supports
it, and validate shape, dtype, device, stride, and backend requirements before
the timing region. Validation may be bypassed only when the project already has
an explicit checked fast path and correctness has been established.

## Nsight Systems

Use Nsight Systems before Nsight Compute when launch names, launch order, or
capture windows are uncertain.

```bash
export NSYS=<path-to-nsys>
"$NSYS" --version
mkdir -p profile
"$NSYS" profile \
  --force-overwrite=true \
  --stats=false \
  --trace=cuda,nvtx \
  --capture-range=cudaProfilerApi \
  --capture-range-end=stop \
  --flush-on-cudaprofilerstop=true \
  -o profile/<target>_<mode>_name_capture \
  "$PYTHON" <bench.py> <args> --warmup 5 --iterations 1 --profile-capture

"$NSYS" stats \
  --force-export=true \
  --report cuda_gpu_kern_sum \
  --format table \
  profile/<target>_<mode>_name_capture.nsys-rep
```

Record full demangled kernel name, short/function name, launch order, grid and
block shape, register count, and static/dynamic shared memory. If the capture
contains unexpected launches, fix the harness or capture window before using
Nsight Compute.

For kernel-only timing, capture many target iterations inside the same profiler
window and compare fused kernels against the sum of the replaced baseline
kernels, not against only one baseline kernel.

## Nsight Compute

Use the exact short/function name discovered from Nsight Systems:

```bash
export NCU=<path-to-ncu>
"$NCU" --version
"$NCU" \
  --force-overwrite \
  --target-processes all \
  --profile-from-start off \
  --kernel-name-base function \
  --kernel-name <exact_function_name> \
  --launch-count 1 \
  --graph-profiling node \
  --cache-control all \
  --clock-control none \
  --section SpeedOfLight \
  --section Occupancy \
  --section MemoryWorkloadAnalysis \
  --section SchedulerStats \
  --section WarpStateStats \
  -o profile/<target>_<mode>_<kernel>_ncu \
  "$PYTHON" <bench.py> <args> --warmup 5 --iterations 1 --profile-capture
```

Export readable reports when needed:

```bash
"$NCU" --import profile/<target>_<mode>_<kernel>_ncu.ncu-rep --page details > profile/<target>_<mode>_<kernel>.details.txt
"$NCU" --import profile/<target>_<mode>_<kernel>_ncu.ncu-rep --csv > profile/<target>_<mode>_<kernel>.csv
```

Report duration, grid/block shape, registers/thread, shared memory, waves/SM,
achieved occupancy, active warps/SM, SM throughput, memory throughput, DRAM
throughput, issue slots, scheduler no-eligible, and warp cycles per issued
instruction.

Nsight Compute diagnoses a single kernel body. It does not include Python
overhead, Triton wrapper overhead, queueing gaps, or launch bubbles outside the
selected kernel.

## Diagnose

- High DRAM or memory throughput with low SM throughput: reduce traffic,
  improve locality, coalesce accesses, or avoid rereading the same data.
- High SM throughput and busy issue slots: estimate the compute floor; random
  sweeps are unlikely to help.
- Low occupancy with tiny grid: split the work, batch it, or fuse a boundary;
  register shaving alone will not create enough CTAs.
- High scheduler no-eligible with moderate occupancy: look for dependency
  chains, long reductions, serialized loads, or insufficient independent work
  per CTA.
- Near-zero SM and memory throughput usually means launch, parallelism, latency,
  or dependency-chain domination, not a clean memory-bound or compute-bound
  kernel.
- Benchmark and integration results disagree: first suspect cache state, launch
  path, missing neighboring work, or workload mismatch.

Common capture failures:

- `ERR_NVGPUCTRPERM`: fix hardware counter permissions on the profiling machine.
- `libcuda.so` missing or CUDA device count is zero: fix Python environment,
  driver visibility, container runtime, or `CUDA_VISIBLE_DEVICES`.
- no kernels profiled: ensure the CUDA profiler API window exists, the kernel
  name matches, and target processes are captured.
- unrelated report: fix kernel filtering or isolate the target launch.

## Iterate

Before running a broad sweep, make the kernel shape structurally credible:
intended dataflow, parallel decomposition, memory traffic pattern, epilogue
fusion, fixed-shape mask simplification, and avoidable global-store removal.

Change one structural variable at a time: work partitioning, tiling, data reuse,
vectorized loads, accumulator layout, shared-memory reuse, split/reduce
boundary, or fusion boundary. Prefer proven Triton IR shapes or previous
accepted CUDA patterns over starting from scratch.

For each credible structure, run serial target-GPU sweeps over the relevant
knobs: block sizes, tiling, split strategy, CTA mapping, warps, stages, vector
width, and structure-specific thresholds. Sweep against the accepted baseline
in the same harness.

For CUDA replacements of existing Triton kernels, benchmark the best relevant
Triton shape in the same run and inspect its IR/generated code or winning
tiling parameters before changing CUDA. For fused or tail-coupled boundaries,
measure both the isolated kernel and the full replaced boundary.

## Promote Or Stop

Integrate behind an explicit env gate first. Keep the gate disabled by default,
ensure it cannot silently combine with incompatible runtime modes, then run the
smallest integration check that exercises the real call path. Use
application-level Nsight Systems to confirm the new kernel is on the intended
hot path.

Promote only when:

- correctness passes direct and integration checks
- Nsight Systems captures only expected target launches
- Nsight Compute uses exact names discovered from Nsight Systems
- representative benchmark improves by more than noise
- Nsight evidence explains the improvement
- fused kernel-only time is not slower than the replaced kernel sum
- runtime integration does not regress relevant application metrics

Stop or redesign when correctness fails, kernel name discovery is ambiguous,
Nsight Compute needs fuzzy regex to hit the target, fused kernel-only time loses
to the replaced kernel sum, event timing is much worse despite good NCU numbers,
or application-level profiling shows the kernel is not on the intended hot
path.

If two iterations fail to beat the baseline for the same diagnosed reason, stop
and report the bound instead of continuing a parameter random walk.
