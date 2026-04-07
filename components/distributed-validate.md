# Distributed Validation

Per-node tests (GPU health, NCCL intra-node) prove that each machine works in isolation. Distributed validation proves that the machines work together, exercising the cross-node communication paths that large-model training depends on: RDMA over InfiniBand, NCCL all-reduce across NVSwitch and IB fabrics, tensor parallel scaling, pipeline parallel bubble overhead, and the hybrid TP+PP pattern used by Megatron-LM.


## The 4-Level Verdict System

Every test in this module returns one of four verdict levels. Three thresholds per metric define the boundaries:

```python
def verdict(value, thresholds, higher_is_better=True):
    """Return OPTIMAL/PASS/DEGRADED/FAIL based on thresholds."""
    if higher_is_better:
        if value >= thresholds["optimal"]:
            return "OPTIMAL"
        elif value >= thresholds["pass"]:
            return "PASS"
        elif value >= thresholds["floor"]:
            return "DEGRADED"
        else:
            return "FAIL"
    else:
        if value <= thresholds["optimal"]:
            return "OPTIMAL"
        elif value <= thresholds["pass"]:
            return "PASS"
        elif value <= thresholds["floor"]:
            return "DEGRADED"
        else:
            return "FAIL"
```

The `higher_is_better` flag flips the comparison direction. Most metrics (bandwidth, TFLOPS) are higher-is-better. Pipeline bubble overhead is the exception: lower bubble means less wasted time, so it uses `higher_is_better=False`.

The four levels mean different things operationally:

- **OPTIMAL** -- within 95% of theoretical peak. This node is performing as expected for its hardware class.
- **PASS** -- above the minimum floor for training viability. The node can be used but may not deliver peak performance.
- **DEGRADED** -- below floor but still functional. Investigate before assigning production workloads.
- **FAIL** -- non-functional. The communication path does not work at all.

## Threshold Loading

Thresholds come from two sources: hardcoded defaults and ConfigMap overrides set by Terraform. The merge logic ensures that operators can tune thresholds per-cluster without touching code:

```python
_DEFAULTS = {
    "nccl_16gpu_allreduce": {"optimal": 450, "pass": 400, "floor": 300},
    "nccl_8gpu_allreduce":  {"optimal": 470, "pass": 440, "floor": 350},
    "rdma_bandwidth_gbps":  {"optimal": 390, "pass": 350, "floor": 200},
    "tp8_vs_tp16_ratio":    {"optimal": 0.85, "pass": 0.70, "floor": 0.50},
    "pp_bubble_overhead":   {"optimal": 0.10, "pass": 0.20, "floor": 0.35},
}


def _load_thresholds():
    """Merge ConfigMap thresholds over defaults."""
    thresholds = {}
    mapping = {
        "nccl_16gpu_allreduce": "distributed_nccl_16gpu",
        "nccl_8gpu_allreduce":  "distributed_nccl_8gpu",
        "rdma_bandwidth_gbps":  "distributed_rdma_bw",
        "tp8_vs_tp16_ratio":    "distributed_tp_ratio",
        "pp_bubble_overhead":   "distributed_pp_bubble",
    }
    for key, defaults in _DEFAULTS.items():
        prefix = mapping.get(key, key)
        thresholds[key] = {
            "optimal": CONFIG.get(f"{prefix}_optimal", defaults["optimal"]),
            "pass":    CONFIG.get(f"{prefix}_pass", defaults["pass"]),
            "floor":   CONFIG.get(f"{prefix}_floor", defaults["floor"]),
        }
    # Honor the legacy single-value multinode threshold
    legacy = CONFIG.get("nccl_multinode_bw_gbps_threshold")
    if legacy:
        thresholds["nccl_16gpu_allreduce"]["pass"] = legacy
    return thresholds
```

The `mapping` dict translates internal metric names to ConfigMap key prefixes. This indirection exists because the ConfigMap key names were established before the 4-level verdict system was added, and renaming ConfigMap keys across running clusters is painful. The legacy `nccl_multinode_bw_gbps_threshold` override at the bottom preserves backward compatibility with clusters that still use the old single-threshold ConfigMap format, overwriting only the `pass` level.

The resulting thresholds table:

| Metric | Optimal | Pass | Floor | Unit |
|--------|---------|------|-------|------|
| `nccl_16gpu_allreduce` | 450 | 400 | 300 | GB/s |
| `nccl_8gpu_allreduce` | 470 | 440 | 350 | GB/s |
| `rdma_bandwidth_gbps` | 390 | 350 | 200 | Gb/s |
| `tp8_vs_tp16_ratio` | 0.85 | 0.70 | 0.50 | ratio |
| `pp_bubble_overhead` | 0.10 | 0.20 | 0.35 | fraction (lower=better) |

## RDMA Check

The first test verifies that GPUDirect RDMA is functional. Without it, cross-node NCCL falls back to staging data through host memory, which roughly halves bandwidth.

```python
def check_rdma():
    results = {}

    # Check nvidia-peermem module
    peermem = subprocess.run(
        ["bash", "-c", "lsmod | grep nvidia_peermem || cat /proc/modules | grep nvidia_peermem"],
        capture_output=True, text=True, timeout=10
    )
    results["nvidia_peermem_loaded"] = "nvidia_peermem" in peermem.stdout

    # Check NCCL GDR env
    gdr_level = os.environ.get("NCCL_NET_GDR_LEVEL", "not set")
    results["nccl_gdr_level"] = gdr_level

    # Check IB devices
    ibstat = subprocess.run(["ibstat"], capture_output=True, text=True, timeout=10)
    ib_active = ibstat.stdout.count("State: Active")
    results["ib_ports_active"] = ib_active

    # Try loading peermem if not loaded
    if not results["nvidia_peermem_loaded"]:
        load_result = subprocess.run(
            ["modprobe", "nvidia-peermem"], capture_output=True, text=True, timeout=10
        )
        if load_result.returncode == 0:
            results["nvidia_peermem_loaded"] = True
        else:
            results["peermem_error"] = load_result.stderr.strip()
            if "Invalid argument" in load_result.stderr:
                print(f"  DIAGNOSIS: NVIDIA driver and MOFED versions are ABI-incompatible")
                results["diagnosis"] = "driver_mofed_abi_mismatch"

    results["pass"] = results["nvidia_peermem_loaded"] and ib_active >= 8
    results["verdict"] = "PASS" if results["pass"] else "FAIL"
    return results
```

The check covers:

1. **nvidia-peermem kernel module** -- this is the bridge between the NVIDIA GPU driver and the Mellanox OFED (MOFED) InfiniBand stack. It enables GPUDirect RDMA, where the NIC can DMA directly to/from GPU memory without going through the CPU. The check tries both `lsmod` and `/proc/modules` because container runtimes sometimes hide kernel module information from `lsmod`.

2. **IB ports active** -- an H200 node has 8 InfiniBand ports (mlx5_0 through mlx5_7), each rated at 400 Gb/s NDR. All 8 should be in "Active" state. Fewer active ports means some NICs failed to link, which reduces available cross-node bandwidth.

3. **Demand loading** -- if peermem is not loaded, the check tries `modprobe nvidia-peermem`. Handles the case where the module was built but not auto-loaded at boot. If the modprobe fails with "Invalid argument", the diagnosis is specific: the NVIDIA GPU operator driver version and the network operator's MOFED version have an ABI mismatch. Common issue when GPU and network operators are upgraded independently -- peermem is compiled against a specific driver ABI, and a version skew makes them incompatible.

The pass condition requires both peermem loaded AND at least 8 active IB ports. Having peermem without IB ports (or vice versa) is still a failure.

## NCCL Benchmark

This test runs the standard NCCL `all_reduce_perf` benchmark to measure raw collective bandwidth:

```python
def run_nccl_benchmark(gpu_count, label):
    cmd = [
        "all_reduce_perf",
        "-b", "8", "-e", "8G", "-f", "2",
        "-g", str(gpu_count),
        "-n", "20",
    ]

    result = subprocess.run(cmd, capture_output=True, text=True, timeout=600)

    # Parse last data line for bus bandwidth
    bus_bw = 0
    for line in result.stdout.split("\n"):
        line = line.strip()
        if line and not line.startswith("#") and not line.startswith("!"):
            parts = line.split()
            if len(parts) >= 8:
                try:
                    float(parts[0])          # verify it's a data line
                    bus_bw = float(parts[-2]) # in-place busbw column
                except (ValueError, IndexError):
                    continue

    threshold_key = f"nccl_{gpu_count}gpu_allreduce"
    thresholds = THRESHOLDS.get(threshold_key, {"optimal": 400, "pass": 350, "floor": 250})
    v = verdict(bus_bw, thresholds)

    return {"bus_bw_gbps": round(bus_bw, 2), "verdict": v, "thresholds": thresholds}
```

The flags: `-b 8 -e 8G -f 2` sweeps message sizes from 8 bytes to 8 GB, doubling each step. `-n 20` runs 20 iterations per size. The bus bandwidth from the last (largest) data line is what matters -- small messages are latency-bound, not bandwidth-bound.

The parsing logic skips comment lines (starting with `#` or `!`) and validates that the first column is a number (confirming it is a data line, not a header). The in-place bus bandwidth is in the second-to-last column of NCCL's output format.

For the intra-node test, `NCCL_IB_DISABLE=1` is set to force NVSwitch-only communication (see the main function). This isolates intra-node NVSwitch bandwidth from cross-node IB bandwidth.

## TP Benchmark

Tensor parallelism shards a model's layers across GPUs. Each GPU computes a partial result, then an all-reduce combines them. The benchmark uses matmul + all-reduce as a proxy for real TP workloads:

```python
def benchmark_tp(tp_size):
    rank = dist.get_rank()
    world_size = dist.get_world_size()

    # 16384x16384 FP16 matmul as compute proxy
    n = 16384
    device = torch.device(f"cuda:{rank % 8}")
    torch.cuda.set_device(device)

    a = torch.randn(n, n, dtype=torch.float16, device=device)
    b = torch.randn(n, n, dtype=torch.float16, device=device)
    c = torch.empty(n, n, dtype=torch.float16, device=device)

    # Warmup -- detect cross-node NCCL failures
    try:
        for _ in range(3):
            torch.mm(a, b, out=c)
            if tp_size > 8:
                dist.all_reduce(c)
        torch.cuda.synchronize()
    except Exception as e:
        return {
            "tp_size": tp_size,
            "verdict": "FAIL",
            "error": f"cross-node all-reduce failed: {e}",
            "diagnosis": "nvidia-peermem not loaded -- NCCL cannot use IB verbs for GPU data",
        }

    # Timed benchmark
    iters = 20
    torch.cuda.synchronize()
    start = time.perf_counter()
    for _ in range(iters):
        torch.mm(a, b, out=c)
        if tp_size > 8:
            dist.all_reduce(c)
    torch.cuda.synchronize()
    elapsed = time.perf_counter() - start

    tflops = 2 * n**3 * iters / elapsed / 1e12
```

**Why TP>8 crosses nodes.** An H200 node has 8 GPUs connected by NVSwitch (NV18 full mesh, 900 GB/s bidirectional raw). TP=8 keeps all communication intra-node (NVSwitch only, fast). TP=16 requires 16 GPUs, which means two nodes, and the all-reduce must cross the InfiniBand fabric. The benchmark runs both to measure the scaling ratio.

**Warmup failure detection.** The warmup phase deliberately calls `dist.all_reduce` for TP>8 and wraps it in a try/except. When nvidia-peermem is not loaded, NCCL falls back to a socket-based transport for GPU data that usually fails outright on large tensors. Catching this during warmup gives a clear diagnosis ("nvidia-peermem not loaded") instead of a cryptic NCCL error during the timed phase.

**The scaling comparison** happens in `main()` after both TP=8 and TP=16 benchmarks complete:

```python
tp8 = results["tests"]["tp_intranode"].get("per_gpu_tflops", 0)
tp16 = results["tests"]["tp_crossnode"].get("per_gpu_tflops", 0)
if tp8 > 0:
    ratio = tp16 / tp8
    v = verdict(ratio, THRESHOLDS["tp8_vs_tp16_ratio"])
```

An OPTIMAL ratio of 0.85 means the per-GPU TFLOPS with cross-node TP=16 is at least 85% of the intra-node TP=8 baseline. Below 0.50 (FAIL) means cross-node communication overhead is eating more than half the compute. Something is seriously wrong with the IB fabric.

## PP Benchmark

Pipeline parallelism partitions the model into sequential stages. Each stage runs on a subset of GPUs, and activations flow from one stage to the next. The overhead is "bubble time," periods where stages are idle waiting for activations.

```python
def benchmark_pp(pp_stages, microbatches=8):
    rank = dist.get_rank()
    world_size = dist.get_world_size()
    device = torch.device(f"cuda:{rank % 8}")

    gpus_per_stage = world_size // pp_stages
    stage = rank // gpus_per_stage
    stage_rank = rank % gpus_per_stage
```

GPUs are divided evenly across stages. With 16 GPUs and PP=2, each stage gets 8 GPUs. `stage_rank` is the rank within a stage. Only `stage_rank == 0` (the "leader" of each stage) participates in cross-stage P2P communication.

### Deadlock-safe P2P with batch_isend_irecv

A naive send-then-recv pattern between two NCCL ranks can deadlock because NCCL send and recv are blocking operations that share the same communication stream. The solution is `dist.batch_isend_irecv`, which submits send and receive operations as a batch, allowing NCCL to schedule them without deadlock:

```python
# P2P warmup using batch_isend_irecv (deadlock-safe)
ops = []
if stage < pp_stages - 1 and stage_rank == 0:
    next_rank = (stage + 1) * gpus_per_stage
    ops.append(dist.P2POp(dist.isend, send_buf, next_rank))
if stage > 0 and stage_rank == 0:
    prev_rank = (stage - 1) * gpus_per_stage
    ops.append(dist.P2POp(dist.irecv, recv_buf, prev_rank))
if ops:
    reqs = dist.batch_isend_irecv(ops)
    for r in reqs:
        r.wait()
```

### Bubble overhead formula

The core metric is bubble overhead, calculated as:

```python
# Theoretical minimum: startup drain adds (pp_stages-1) stage times
ideal_time = (microbatches + pp_stages - 1) * compute_time
bubble = max(0, (pipeline_time - ideal_time) / pipeline_time)
```

**Why ideal includes `pp_stages - 1`.** Even a perfect pipeline has startup and drain costs. With PP=2, the first microbatch must complete stage 0 before stage 1 can begin. The last microbatch in stage 0 finishes while stage 1 is still working on the previous one. This "pipeline fill/drain" adds exactly `pp_stages - 1` stage times to the theoretical minimum. So the ideal time for 8 microbatches with PP=2 is `(8 + 2 - 1) * stage_time = 9 * stage_time`, not `8 * stage_time`.

The formula `(actual - ideal) / actual` gives the fraction of wall-clock time that was wasted. An OPTIMAL bubble of 10% means 90% of the pipeline time was productive compute. Above 35% (FAIL) means the P2P communication overhead dominates the pipeline.

## Hybrid TP+PP

Real large-model training (Megatron-LM, DeepSpeed) uses both TP and PP simultaneously. TP handles intra-layer parallelism (sharding a single transformer layer's weights), while PP handles inter-layer parallelism (assigning different layers to different pipeline stages). The hybrid benchmark combines both:

```python
def benchmark_tp_pp(tp_size, pp_stages, microbatches=8):
    rank = dist.get_rank()
    world_size = dist.get_world_size()

    # Assign: pipeline stage, then TP rank within stage
    stage = rank // tp_size
    tp_rank = rank % tp_size

    # Create per-stage TP process groups (all ranks must call new_group)
    tp_groups = []
    for s in range(pp_stages):
        ranks_in_stage = list(range(s * tp_size, (s + 1) * tp_size))
        grp = dist.new_group(ranks_in_stage)
        tp_groups.append(grp)

    my_tp_group = tp_groups[stage]
```

**Per-stage process groups.** Each pipeline stage needs its own NCCL communicator for TP all-reduce operations. `dist.new_group` is a collective operation: every rank in the world must call it, even if they are not in the group being created. The loop creates all `pp_stages` groups on every rank, but each rank only uses `tp_groups[stage]`.

The pipeline loop combines TP all-reduce within each stage with P2P between stages:

```python
for mb in range(microbatches):
    # Recv activation from previous stage (stage leaders only)
    if stage > 0 and tp_rank == 0:
        ops = [dist.P2POp(dist.irecv, recv_buf, prev_leader)]
        reqs = dist.batch_isend_irecv(ops)
        for r in reqs: r.wait()

    # Compute + TP all-reduce (all ranks in stage)
    torch.mm(a, w, out=out)
    dist.all_reduce(out, group=my_tp_group)

    # Send activation to next stage (stage leaders only)
    if stage < pp_stages - 1 and tp_rank == 0:
        ops = [dist.P2POp(dist.isend, out, next_leader)]
        reqs = dist.batch_isend_irecv(ops)
        for r in reqs: r.wait()
```

Only `tp_rank == 0` (the stage leader) handles cross-stage P2P. In Megatron-LM, the layer output from the TP all-reduce is on all GPUs in the stage, but only the leader sends it forward. Avoids sending redundant copies across the IB fabric.

The bubble overhead formula is the same as the PP-only benchmark, but the stage time now includes both compute and TP all-reduce cost, making it a more realistic measure of hybrid scaling efficiency.

## Main -- Test Sequence and Overall Verdict

The `main()` function orchestrates all tests in sequence:

```python
def main():
    rank = int(os.environ.get("RANK", "0"))
    world_size = int(os.environ.get("WORLD_SIZE", "1"))
    gpu_count = torch.cuda.device_count()
    num_nodes = max(1, world_size // gpu_count)

    # Only rank 0 creates the results directory (avoid shared-fs mkdir race)
    if rank == 0:
        os.makedirs(RESULTS_DIR, exist_ok=True)
```

Tests run in this order:

1. **RDMA check** (rank 0 only) -- verifies the foundation for everything else
2. **NCCL 8-GPU intra-node** (rank 0 only, `NCCL_IB_DISABLE=1`) -- NVSwitch-only baseline
3. **TP=8 intra-node** (all ranks) -- compute + NVSwitch all-reduce
4. **TP=16 cross-node** (all ranks, if `world_size >= 16`) -- compute + IB all-reduce
5. **TP scaling comparison** -- ratio of TP=16/TP=8 per-GPU TFLOPS
6. **PP=2 pipeline** (if `world_size >= 4`) -- P2P bubble overhead
7. **Hybrid TP+PP** (if `world_size >= 4`) -- full Megatron-LM pattern

The order is intentional. RDMA is checked first because a failed RDMA check explains why the TP=16 benchmark will fail later. The intra-node NCCL test runs with IB disabled to isolate NVSwitch performance from IB issues.

Overall verdict aggregation is conservative:

```python
verdicts = [
    t.get("verdict", "SKIP")
    for t in results["tests"].values()
    if isinstance(t, dict) and "verdict" in t
]
if "FAIL" in verdicts:
    results["overall"] = "FAIL"
elif "DEGRADED" in verdicts:
    results["overall"] = "DEGRADED"
elif all(v in ("OPTIMAL", "PASS", "SKIP") for v in verdicts):
    results["overall"] = "PASS"
else:
    results["overall"] = "DEGRADED"
```

Any single FAIL makes the overall result FAIL. Any DEGRADED makes it DEGRADED. SKIP results (e.g., TP=16 on a single-node cluster) are neutral. Distributed training is only as strong as its weakest link: a node with perfect NVSwitch but broken IB cannot participate in multi-node training.

## How torchrun Is Launched

This script does not run standalone. The K8s manifest uses `completionMode: Indexed` to create one pod per GPU node. The entrypoint script discovers the master IP via a shared filesystem convention (the first pod writes its IP to a known path, other pods read it), then launches:

```bash
torchrun --nproc_per_node=8 --nnodes=2 --rdzv_backend=c10d \
  --rdzv_endpoint=$MASTER_ADDR:29500 distributed_validate.py
```

`torchrun` handles process spawning (8 per node), rank assignment (`RANK`, `LOCAL_RANK`, `WORLD_SIZE` environment variables), and the C10d rendezvous that synchronizes all processes before `dist.init_process_group` completes.

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- where distributed validation fits in the system
- [Execution Flow](../architecture/execution-flow.md) -- Stage 5d
- [Decision Logic](../architecture/decision-logic.md) -- 4-level verdict system
- [Observability](../architecture/observability.md) -- RDMA, NCCL, TP, and PP benchmarks emit OTel spans
- [Training Burn-in](training-burnin.md) -- the AI-stack validation that runs after distributed checks pass
- [ConfigMap Thresholds](../patterns/configmap-thresholds.md) -- how threshold values are managed in Terraform
