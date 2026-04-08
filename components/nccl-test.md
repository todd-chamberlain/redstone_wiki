# NCCL Intra-Node Test

## Purpose

Every GPU node in the cluster connects its 8 GPUs through an NVLink/NVSwitch fabric -- a high-bandwidth mesh internal to the node. Before a node can participate in distributed training, this internal fabric must be proven healthy. The test runs three standard NCCL collective operations across all local GPUs and measures the bus bandwidth each achieves.

NCCL collectives are the communication primitives behind distributed deep learning. When a training framework calls `all_reduce` to average gradients, or `all_gather` to broadcast activations, it is NCCL routing data through whatever interconnect is available -- NVLink within a node, InfiniBand across nodes. By benchmarking these collectives in isolation, we get a direct measurement of fabric health before any real workload touches it.

| | |
|---|---|
| **Depends on** | [tracing](tracing.md), [configmap](../patterns/configmap-thresholds.md) |
| **Runs as** | Job per GPU node (created by [Orchestrator](orchestrator.md)) |

---

## Why NCCL_IB_DISABLE=1

The environment variable `NCCL_IB_DISABLE=1` must be set before running this test. Without it, NCCL will discover the InfiniBand NICs and potentially route traffic over the IB fabric instead of purely through NVSwitch. That would conflate two different failure domains: an NVSwitch problem would be masked by IB fallback, or an IB problem would pollute what should be a purely intra-node measurement.

By disabling IB, we isolate the test to the NVLink/NVSwitch fabric exclusively. The multi-node NCCL test (see [Distributed Validate](distributed-validate.md)) runs separately with IB enabled.

```python
TEST_MODE = os.environ.get("TEST_MODE", "intranode")
# For multi-node burn-in, set TEST_MODE=multinode and NCCL_IB_DISABLE=0.
```

---

## Parsing nccl-tests Output

The nccl-tests binaries (`all_reduce_perf`, `all_gather_perf`, `reduce_scatter_perf`) produce tabular stdout. Each row represents a message size, and the columns include throughput metrics. The output looks roughly like this:

```
#       size  count  type  redop  time  algbw  busbw  error
           8      2  float    sum  25.3   0.00   0.00  0e+00
          16      4  float    sum  25.1   0.00   0.00  0e+00
  ...
  8589934592  ...    float    sum  17.9  479.2  479.2  0e+00
```

Lines starting with `#` or `!` are headers/warnings. Data rows start with a numeric message size. The parser identifies these by trying to parse the first column as a float:

```python
def parse_nccl_output(stdout: str) -> dict:
    data_rows = []
    for line in stdout.split("\n"):
        line = line.strip()
        if line and not line.startswith("#") and not line.startswith("!"):
            parts = line.split()
            if len(parts) >= 8:
                try:
                    float(parts[0])  # Verify first column is numeric
                    data_rows.append(parts)
                except ValueError:
                    continue

    if not data_rows:
        return {"error": "No data rows parsed", "bus_bw_gbps": 0}

    last = data_rows[-1]
    small_msg_time_us = float(data_rows[0][5]) if len(data_rows[0]) > 5 else 0

    return {
        "msg_size": int(last[0]),
        "alg_bw_gbps": float(last[-3]),
        "bus_bw_gbps": float(last[-2]),
        "small_msg_latency_us": small_msg_time_us,
        "rows": len(data_rows),
    }
```

The metric that matters: **bus bandwidth** (`bus_bw_gbps`), taken from the **last data row** (largest message size). Bus bandwidth normalizes for the collective's communication pattern, representing the actual per-link throughput rather than the algorithm-level throughput. Directly reflects NVSwitch health.

The small-message latency from the first row is captured as a secondary diagnostic. If the fabric has high latency at small sizes, it can indicate switch congestion even when large-message bandwidth looks fine.

---

## Per-Collective Thresholds

Not all collectives achieve the same bandwidth. `all_reduce` combines data in both directions (reduce + broadcast), so on a fully connected NVSwitch topology it can saturate more links simultaneously than `all_gather` or `reduce_scatter`, which only move data in one direction. The thresholds reflect this:

| Collective | ConfigMap Key | Default Threshold |
|---|---|---|
| all_reduce | `nccl_allreduce_bw_threshold` | 450 GB/s |
| all_gather | `nccl_allgather_bw_threshold` | 340 GB/s |
| reduce_scatter | `nccl_reducescatter_bw_threshold` | 340 GB/s |

The `get_threshold` function implements a fallback chain. It first looks for the per-collective key in the config. If that key is missing (for example, if someone is running an older config that predates per-collective thresholds), it falls back to the legacy single-value key `nccl_bw_gbps_threshold`. If even that is absent, it defaults to 400 GB/s:

```python
def get_threshold(collective: str) -> float:
    key_map = {
        "all_reduce": "nccl_allreduce_bw_threshold",
        "all_gather": "nccl_allgather_bw_threshold",
        "reduce_scatter": "nccl_reducescatter_bw_threshold",
    }
    key = key_map.get(collective, "nccl_allreduce_bw_threshold")
    # Fall back to legacy single threshold if per-collective not set
    return CONFIG.get(key, CONFIG.get("nccl_bw_gbps_threshold", 400))
```

This fallback chain avoids breaking existing configs. Clusters that already have a single threshold will not break when the code starts looking for per-collective keys.

---

## Running a Collective Benchmark

Each collective is tested by invoking the corresponding nccl-tests binary. The command-line arguments sweep message sizes from 8 bytes to 8 GB, doubling at each step (`-f 2`), across all GPUs on the node:

```python
def run_nccl_collective(collective: str, gpu_count: int = 8) -> dict:
    binary = f"{collective}_perf"
    cmd = [binary, "-b", "8", "-e", "8G", "-f", "2", "-g", str(gpu_count)]

    result = subprocess.run(
        cmd, capture_output=True, text=True, timeout=300
    )

    if result.returncode != 0:
        return {
            "collective": collective,
            "error": result.stderr[:500],
            "bus_bw_gbps": 0,
            "pass": False,
        }

    parsed = parse_nccl_output(result.stdout)
    threshold = get_threshold(collective)
    passed = parsed["bus_bw_gbps"] >= threshold

    return {
        "collective": collective,
        "bus_bw_gbps": parsed["bus_bw_gbps"],
        "alg_bw_gbps": parsed.get("alg_bw_gbps", 0),
        "small_msg_latency_us": parsed.get("small_msg_latency_us", 0),
        "msg_size": parsed.get("msg_size", 0),
        "threshold": threshold,
        "pass": passed,
    }
```

The 300-second timeout is generous but necessary. On 8 GPUs sweeping from 8 bytes to 8 GB, the benchmark can take several minutes, especially at the large message sizes where it transfers hundreds of gigabytes.

The error-truncation pattern (`result.stderr[:500]`) prevents a noisy NCCL error (which can be thousands of lines of backtrace) from bloating the results JSON.

---

## Main Loop and Orchestration

The main function detects GPU count via PyTorch, then runs all three collectives sequentially. Each collective runs inside its own OpenTelemetry span for tracing. If any collective fails its threshold, the overall result is marked `DEGRADED` (not `FAIL`) -- a degraded NVSwitch link may still allow training, just more slowly:

```python
def main():
    import torch
    gpu_count = torch.cuda.device_count()

    results = {
        "node": NODE_NAME,
        "run_id": RUN_ID,
        "test_mode": TEST_MODE,
        "gpu_count": gpu_count,
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "collectives": {},
        "overall": "PASS",
    }

    collectives = ["all_reduce", "all_gather", "reduce_scatter"]

    with tracer.start_as_current_span(f"nccl_{TEST_MODE}", attributes={
        "node": NODE_NAME, "run_id": RUN_ID, "gpu_count": gpu_count,
        "test_mode": TEST_MODE,
    }) as root_span:

        for collective in collectives:
            with tracer.start_as_current_span(f"nccl.{collective}") as span:
                result = run_nccl_collective(collective, gpu_count)
                results["collectives"][collective] = result

                span.set_attribute("bus_bw_gbps", result["bus_bw_gbps"])
                span.set_attribute("threshold", result.get("threshold", 0))
                span.set_attribute("pass", result["pass"])

                if not result["pass"]:
                    results["overall"] = "DEGRADED"

        root_span.set_attribute("overall", results["overall"])

    output_path = f"{RESULTS_DIR}/{NODE_NAME}.json"
    with open(output_path, "w") as f:
        json.dump(results, f, indent=2)
```

`DEGRADED` rather than `FAIL` is intentional. The [Orchestrator](orchestrator.md) treats degraded nodes differently from failed nodes: flagged for investigation but not pulled from a training run.

---

## Output Format

Results are written as JSON to `<results_base_path>/<run_id>/nccl/<node_name>.json`:

```json
{
  "node": "gpu-node-0",
  "run_id": "20250415-001",
  "test_mode": "intranode",
  "gpu_count": 8,
  "timestamp": "2025-04-15T10:30:00Z",
  "overall": "PASS",
  "collectives": {
    "all_reduce": { "bus_bw_gbps": 479.2, "alg_bw_gbps": 479.2, "threshold": 450, "pass": true },
    "all_gather": { "bus_bw_gbps": 368.1, "alg_bw_gbps": 368.1, "threshold": 340, "pass": true },
    "reduce_scatter": { "bus_bw_gbps": 361.5, "alg_bw_gbps": 361.5, "threshold": 340, "pass": true }
  }
}
```

The [Orchestrator](orchestrator.md) reads this file after the job completes to decide whether the node can proceed to subsequent test phases.

---

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- where the NCCL test fits in the system
- [Execution Flow](../architecture/execution-flow.md) -- NCCL is Phase 2 of the preflight pipeline
- [Decision Logic](../architecture/decision-logic.md) -- NCCL results feed into the READY/DEGRADED/NOT_READY verdict
- [Observability](../architecture/observability.md) -- each collective gets its own OTel span for tracing in Grafana
- [Orchestrator](orchestrator.md) -- `phase_nccl` creates these jobs
- [Distributed Validate](distributed-validate.md) -- cross-node NCCL with IB enabled
- [Thresholds](../reference/thresholds.md) -- all threshold values
