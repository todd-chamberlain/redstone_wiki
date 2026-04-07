# GPU Health

Per-node GPU validation that runs as a DaemonSet on every GPU node. Checks the full AI stack (driver, CUDA, NCCL, PyTorch versions, GPU count, ECC, thermals, and raw compute) then writes a JSON verdict to the shared filesystem. On critical failures, the node is automatically tainted to prevent workload scheduling.

## Source

| | |
|---|---|
| **File** | `gpu_health.py` — 452 lines |
| **Runs as** | DaemonSet on every `nvidia.com/gpu.present=true` node |
| **Depends on** | [tracing](tracing.md), [ConfigMap thresholds](../patterns/configmap-thresholds.md) |
| **Called by** | [Orchestrator](orchestrator.md) polls for results via shared filesystem |

## Startup

Every script in the suite follows the same config bootstrap: load the ConfigMap JSON, init the OTel tracer, and set up a results directory.

```python
CONFIG_PATH = os.environ.get("REDSTONE_CONFIG", "/etc/redstone/config.json")
NODE_NAME   = os.environ.get("NODE_NAME", "unknown")
RUN_ID      = os.environ.get("RUN_ID", "unknown")

with open(CONFIG_PATH) as f:
    CONFIG = json.load(f)

RESULTS_DIR = f"{CONFIG['results_base_path']}/{RUN_ID}/gpu-health"
tracer = init_tracer("redstone.gpu_health")
```

`NODE_NAME` comes from the Kubernetes downward API (`fieldRef: spec.nodeName`). `RUN_ID` is the timestamp-based identifier that ties all phases of a preflight run together.

## The Check Loop

The main function runs 9 checks sequentially. Each gets its own OTel span so you can trace individual check durations in Grafana:

```python
checks = [
    ("driver_version",  check_driver_version),
    ("cuda_version",    check_cuda_version),
    ("nccl_version",    check_nccl_version),
    ("pytorch_version", check_pytorch_version),
    ("gpu_count",       check_gpu_count),       # CRITICAL
    ("ecc_status",      check_ecc_status),       # CRITICAL
    ("utilization",     check_utilization),
    ("temperature",     check_temperature),
    ("matmul_bench",    check_matmul_benchmark),
]

for name, check_fn in checks:
    with tracer.start_as_current_span(f"gpu_health.{name}") as span:
        result = check_fn(span)
        results["checks"][name] = result
        if not result["pass"]:
            if name in ("gpu_count", "ecc_status"):
                critical_failure = True   # → taint node, verdict FAIL
```

The two-tier system matters: `gpu_count` and `ecc_status` failures mean the hardware is fundamentally broken, so the node gets tainted. Everything else (driver mismatch, high temp, slow matmul) produces DEGRADED. The GPUs work, but something needs attention.

## Version Checks

### Driver, CUDA, NCCL, PyTorch

All four version checks follow the same pattern: query the actual version, compare against a compatibility range from the [ConfigMap](../patterns/configmap-thresholds.md).

The driver check is representative:

```python
def check_driver_version(span):
    result = subprocess.run(
        ["nvidia-smi", "--query-gpu=driver_version", "--format=csv,noheader,nounits"],
        capture_output=True, text=True, timeout=30
    )
    actual = result.stdout.strip().split("\n")[0]
    compat = CONFIG.get("compatibility", {})

    actual_major = int(actual.split(".")[0])
    min_major = compat.get("driver_min_major", 0)    # default: 550
    max_major = compat.get("driver_max_major", 9999)  # default: 590
    in_range = min_major <= actual_major <= max_major
```

The compatibility matrix is defined in Terraform and supports range-based validation rather than exact pinning, since GPU Operator may install different driver minor versions across clusters:

| Check | Source | Compatibility Range |
|-------|--------|-------------------|
| Driver | `nvidia-smi` | major 550–590 |
| CUDA | `nvcc` or `torch.version.cuda` | major 12–13 |
| NCCL | `torch.cuda.nccl.version()` | major >= 2 |
| PyTorch | `torch.__version__` | >= 2.0 |

CUDA has a fallback chain: tries `nvcc --version` first (more reliable when available), falls back to PyTorch's bundled CUDA version.

## Hardware Checks

### GPU Count

If the node does not see all 8 GPUs, something is seriously wrong (PCIe failure, GPU Operator misconfiguration):

```python
def check_gpu_count(span):
    actual = torch.cuda.device_count()
    expected = CONFIG.get("expected_gpu_count", 8)
    passed = actual == expected
```

A **critical check** — failure triggers [node tainting](#node-tainting).

### ECC Status

Queries per-GPU ECC mode and uncorrected error counts. Any GPU with ECC disabled or uncorrected errors > 0 fails. This catches silent data corruption that would produce garbage training gradients:

```python
result = subprocess.run(
    ["nvidia-smi",
     "--query-gpu=ecc.mode.current,ecc.errors.uncorrected.aggregate.total",
     "--format=csv,noheader,nounits"],
    capture_output=True, text=True, timeout=30
)

for i, line in enumerate(result.stdout.strip().split("\n")):
    parts = [p.strip() for p in line.split(",")]
    ecc_enabled = parts[0] == "Enabled"
    errors = int(parts[1]) if parts[1] != "N/A" else 0
    gpu_pass = ecc_enabled and errors == 0
```

Also a **critical check**. Uncorrected ECC errors are hardware failures. The node needs replacement.

### Utilization and Power

Checks that no GPU is drawing more power than its limit at idle. If `power_draw > power_limit`, the hardware is throttling or misconfigured:

```python
gpu_pass = power_draw <= power_limit if power_limit > 0 else True
```

### Temperature

Reads all GPU temperatures, fails if any exceeds the threshold (default 85C):

```python
max_temp = max(temps)
threshold = CONFIG.get("gpu_temperature_threshold_c", 85)
passed = max_temp < threshold
```

High temperatures at idle before any training workload indicate cooling failures or bad thermal paste.

## Compute Benchmark

The matmul benchmark validates that the GPU can compute at expected speeds, not just that the software stack reports the right versions:

```python
def check_matmul_benchmark(span):
    threshold = CONFIG.get("matmul_tflops_threshold", 900)

    for gpu_id in range(gpu_count):
        torch.cuda.set_device(gpu_id)
        n = 8192
        a = torch.randn(n, n, dtype=torch.float16, device="cuda")
        b = torch.randn(n, n, dtype=torch.float16, device="cuda")

        # Warmup
        for _ in range(5):
            torch.mm(a, b)
        torch.cuda.synchronize()

        # Benchmark
        iters = 20
        start = time.perf_counter()
        for _ in range(iters):
            torch.mm(a, b)
        torch.cuda.synchronize()
        elapsed = time.perf_counter() - start

        flops = 2 * n**3 * iters
        tflops = flops / elapsed / 1e12
        gpu_pass = tflops >= threshold
```

Key details:
- **FP16 8192x8192** — large enough to saturate the tensor cores
- **5 warmup iterations** — lets the GPU boost to max clock
- **`torch.cuda.synchronize()`** — critical for accurate timing (GPU ops are async)
- **900 TFLOPS threshold** — H200 theoretical peak is ~990 FP16 TFLOPS, so 900 is ~91% utilization
- **Per-GPU** — catches individual bad GPUs even if the average looks fine

Each GPU gets tested and cleaned up individually (`del a, b; torch.cuda.empty_cache()`) to avoid OOM on 8-GPU nodes.

## Node Tainting

When a critical check fails, the node is automatically excluded from future workloads:

```python
def taint_node(reason: str):
    from kubernetes import client, config
    config.load_incluster_config()
    v1 = client.CoreV1Api()

    node = v1.read_node(NODE_NAME)
    taints = node.spec.taints or []
    taints.append(client.V1Taint(
        key="redstone", value=reason, effect="NoSchedule"
    ))
    v1.patch_node(NODE_NAME, {"spec": {"taints": taints}})
```

The `NoSchedule` effect means existing pods keep running but no new pods land on this node. The taint key `redstone` is cleared at the start of each preflight run (`kubectl taint nodes {} redstone-`), so a repaired node will be re-tested.

This requires the `redstone-runner` ServiceAccount to have `patch` access on nodes, granted by the `redstone-node-access` ClusterRole. See [Node Tainting](../patterns/node-tainting.md) for the full pattern.

## Verdict Logic

```python
if critical_failure:
    results["overall"] = "FAIL"
    taint_node("failed")
elif any(not c.get("pass", True) for c in results["checks"].values()):
    results["overall"] = "DEGRADED"
else:
    results["overall"] = "PASS"
```

| Condition | Verdict | Effect |
|-----------|---------|--------|
| `gpu_count` or `ecc_status` fails, or any check throws | **FAIL** | Node tainted, excluded from all subsequent phases |
| Any non-critical check fails | **DEGRADED** | Node is viable, investigate non-critical failures |
| All checks pass | **PASS** | Node is healthy |

The orchestrator treats DEGRADED nodes as viable (they still have working GPUs), but includes them in remediation output.

## Output

Results are written to `<RESULTS_DIR>/gpu-health/<NODE_NAME>.json`:

```json
{
  "node": "gpu-node-0",
  "run_id": "20260402-005513",
  "timestamp": "2026-04-02T00:55:13Z",
  "overall": "PASS",
  "checks": {
    "driver_version": { "actual": "580.95.05", "in_range": true, "pass": true },
    "cuda_version":   { "actual": "13.0", "in_range": true, "pass": true },
    "gpu_count":      { "expected": 8, "actual": 8, "pass": true },
    "ecc_status":     { "gpus": [{"gpu": 0, "ecc_enabled": true, "uncorrected_errors": 0}], "pass": true },
    "temperature":    { "max": 42, "threshold": 85, "pass": true },
    "matmul_bench":   { "avg_tflops": 985.3, "threshold": 900, "pass": true }
  }
}
```

The DaemonSet pod exits after writing results. The orchestrator's [`phase_gpu_health`](orchestrator.md#how-it-works) polls for these JSON files on the shared filesystem.

## Configuration

All thresholds come from the [ConfigMap](../patterns/configmap-thresholds.md) and are tunable via Terraform without rebuilding the container:

| Key | Default | What It Controls |
|-----|---------|-----------------|
| `expected_gpu_count` | 8 | GPUs per node (critical check) |
| `gpu_temperature_threshold_c` | 85 | Max temp in Celsius |
| `matmul_tflops_threshold` | 900 | Min FP16 TFLOPS per GPU |
| `compatibility.driver_min_major` | 550 | Driver version floor |
| `compatibility.driver_max_major` | 590 | Driver version ceiling |
| `compatibility.cuda_min_major` | 12 | CUDA version floor |
| `compatibility.pytorch_min` | `2.0` | PyTorch minimum |

See [Thresholds](../reference/thresholds.md) for the complete table with Terraform variable names.

## Cross-References

- [Architecture Overview](../architecture/overview.md) — where GPU health fits as Phase 1 in the system
- [Execution Flow](../architecture/execution-flow.md) — GPU health is Phase 1 of the preflight pipeline
- [Orchestrator](orchestrator.md) — `phase_gpu_health` polls for these results on the shared FS
- [Node Tainting](../patterns/node-tainting.md) — the auto-exclusion pattern
- [Decision Logic](../architecture/decision-logic.md) — how FAIL/DEGRADED/PASS feeds into the cluster verdict
- [Observability](../architecture/observability.md) — TraceQL: `{ resource.service.name = "redstone.gpu_health" }`
