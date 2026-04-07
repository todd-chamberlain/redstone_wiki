# Storage Test

## Purpose

GPU training clusters depend on two storage tiers: a shared filesystem (typically Lustre or GPFS) for checkpoint I/O and dataset loading, and local network-attached SSDs for scratch space and fast caching. Both tiers must meet minimum performance requirements, and both degrade under contention when many nodes hit them simultaneously.

`fio` benchmarks five specific I/O patterns that map to real training workloads:

- **Sequential read/write with large blocks** -- this is the checkpoint pattern. When a training run saves or restores a checkpoint, it writes or reads multi-gigabyte files in large sequential chunks.
- **Random read with small blocks** -- this is the DataLoader pattern. PyTorch DataLoader workers read random samples from datasets stored on the shared filesystem, issuing many small random reads concurrently.

The [Orchestrator](orchestrator.md) runs this module twice per node: once in isolation to establish a baseline, and once with all nodes hitting storage simultaneously to measure contention-induced degradation.

| | |
|---|---|
| **Depends on** | [tracing](tracing.md), [configmap](../patterns/configmap-thresholds.md) |
| **Runs as** | Job per GPU node (created by [Orchestrator](orchestrator.md)) |

---

## The run_fio Wrapper

Every benchmark flows through a single wrapper function that handles fio invocation, output parsing, and cleanup:

```python
SHARED_FS_PATH = "/mnt/data/redstone-fio-test"
LOCAL_SSD_PATH = "/tmp/redstone-fio-test"

def run_fio(name: str, target_path: str, rw: str, bs: str,
            size: str = "10G", numjobs: int = 8, runtime: int = 30) -> dict:
    os.makedirs(target_path, exist_ok=True)
    fio_file = os.path.join(target_path, f"fio-{name}")

    cmd = [
        "fio",
        f"--name={name}",
        f"--filename={fio_file}",
        f"--rw={rw}",
        f"--bs={bs}",
        f"--size={size}",
        f"--numjobs={numjobs}",
        "--direct=1",
        f"--runtime={runtime}",
        "--time_based",
        "--group_reporting",
        "--output-format=json",
    ]

    result = subprocess.run(cmd, capture_output=True, text=True, timeout=300)
```

Key flags:

- **`--direct=1`** bypasses the OS page cache. Without direct I/O, fio would measure the kernel's ability to cache data in RAM, not the storage system's actual throughput. For storage validation, we need to hit the actual hardware.
- **`--time_based`** combined with `--runtime=30` means fio runs for exactly 30 seconds regardless of how much data it transfers. Without `--time_based`, fio would stop after transferring `--size` bytes, which produces inconsistent test durations across different storage speeds.
- **`--group_reporting`** aggregates results across all `numjobs` workers into a single summary, simplifying parsing.

The parser extracts bandwidth, IOPS, and completion latency from fio's JSON output, converting units along the way:

```python
    try:
        fio_data = json.loads(result.stdout)
        job = fio_data["jobs"][0]

        if "read" in rw:
            bw_mbps = job["read"]["bw"] / 1024     # KB/s -> MB/s
            iops = job["read"]["iops"]
            lat_us = job["read"]["clat_ns"]["mean"] / 1000  # ns -> us
        else:
            bw_mbps = job["write"]["bw"] / 1024
            iops = job["write"]["iops"]
            lat_us = job["write"]["clat_ns"]["mean"] / 1000

        return {
            "name": name,
            "rw": rw,
            "bs": bs,
            "bw_mbps": round(bw_mbps, 1),
            "iops": round(iops, 1),
            "lat_us": round(lat_us, 1),
        }
    except (json.JSONDecodeError, KeyError, IndexError) as e:
        return {"name": name, "error": str(e), "pass": False}
    finally:
        try:
            os.remove(fio_file)
        except OSError:
            pass
```

The `finally` block cleans up the test file. fio creates a 10 GB file on every run; leaving these behind on a shared filesystem across dozens of nodes would waste significant space.

---

## The Five Benchmarks

The module runs three benchmarks against the shared filesystem and two against the local SSD. Each benchmark is tuned with block sizes and job counts that match the workload it simulates:

### Shared Filesystem Tests

```python
shared_tests = [
    ("shared_fs.seq_read",  SHARED_FS_PATH, "read",     "1M", 8,
     CONFIG.get("storage_seq_read_mbps", 500) * threshold_mult, "bw_mbps"),
    ("shared_fs.seq_write", SHARED_FS_PATH, "write",    "1M", 8,
     CONFIG.get("storage_seq_write_mbps", 400) * threshold_mult, "bw_mbps"),
    ("shared_fs.rand_read", SHARED_FS_PATH, "randread", "4K", 16,
     CONFIG.get("storage_rand_read_iops", 5000) * threshold_mult, "iops"),
]
```

| Test | Block Size | Jobs | Metric | Default Threshold | Simulates |
|---|---|---|---|---|---|
| `shared_fs.seq_read` | 1M | 8 | Bandwidth (MB/s) | 500 | Checkpoint restore |
| `shared_fs.seq_write` | 1M | 8 | Bandwidth (MB/s) | 400 | Checkpoint save |
| `shared_fs.rand_read` | 4K | 16 | IOPS | 5000 | DataLoader sample reads |

Sequential tests use 1 MB blocks with 8 parallel jobs, mirroring how checkpoint frameworks like PyTorch's `torch.save` issue large sequential writes across multiple file handles. The read threshold (500 MB/s) is higher than write (400 MB/s) because reads are typically faster on parallel filesystems that can serve from multiple OSTs.

Random read uses 4K blocks with 16 parallel jobs to simulate the high-concurrency, small-read pattern of data loading. The metric here is IOPS, not bandwidth, because at 4K block sizes, IOPS is the bottleneck, not throughput.

### Local SSD Tests

```python
local_tests = [
    ("network_ssd.seq_read",  LOCAL_SSD_PATH, "read",  "128K", 4, 1000, "bw_mbps"),
    ("network_ssd.seq_write", LOCAL_SSD_PATH, "write", "128K", 4, 1000, "bw_mbps"),
]
```

| Test | Block Size | Jobs | Threshold |
|---|---|---|---|
| `network_ssd.seq_read` | 128K | 4 | 1000 MB/s |
| `network_ssd.seq_write` | 128K | 4 | 1000 MB/s |

Local SSDs use 128K blocks and 4 jobs. This tests the NVMe link without overwhelming the device queue. The 1000 MB/s threshold is hardcoded rather than config-driven because local SSD performance is a hardware spec, not a tunable parameter.

---

## Contention Threshold Relaxation

The [Orchestrator](orchestrator.md) runs storage tests in two phases: **isolation** first (one node at a time), then **contention** (all nodes simultaneously). The test phase is communicated via the `TEST_PHASE` environment variable.

During the contention phase, it is expected that every node will see lower throughput because they are all competing for the same shared filesystem bandwidth. The thresholds are therefore relaxed by a configurable factor:

```python
TEST_PHASE = os.environ.get("TEST_PHASE", "baseline")

contention_factor = CONFIG.get("storage_contention_max", 50) / 100.0
is_contention = TEST_PHASE == "concurrent"
threshold_mult = (1 - contention_factor) if is_contention else 1.0
```

With the default `storage_contention_max` of 50%, the contention-phase thresholds become half of the isolation thresholds. So a shared filesystem that must deliver 500 MB/s in isolation only needs to deliver 250 MB/s per node under contention.

The isolation run establishes what each node *can* achieve when it has the filesystem to itself. The contention run measures how much performance degrades under realistic concurrent access. The orchestrator compares the two results to compute a per-node degradation percentage, catching nodes that degrade disproportionately more than their peers (a sign of a bad network path to the storage servers).

---

## Output Format

Results are written to `<results_base_path>/<run_id>/storage/<node_name>-<phase>.json`:

```json
{
  "node": "gpu-node-0",
  "run_id": "20250415-001",
  "phase": "baseline",
  "timestamp": "2025-04-15T10:45:00Z",
  "overall": "PASS",
  "tests": {
    "shared_fs.seq_read":  { "bw_mbps": 612.3, "iops": 612.3, "threshold": 500, "pass": true },
    "shared_fs.seq_write": { "bw_mbps": 478.1, "iops": 478.1, "threshold": 400, "pass": true },
    "shared_fs.rand_read": { "bw_mbps": 22.1,  "iops": 5650,  "threshold": 5000, "pass": true },
    "network_ssd.seq_read":  { "bw_mbps": 1823.5, "threshold": 1000, "pass": true },
    "network_ssd.seq_write": { "bw_mbps": 1512.0, "threshold": 1000, "pass": true }
  }
}
```

The filename includes the phase (`baseline` or `concurrent`) so the orchestrator can find and compare both results for each node.

---

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- where storage testing fits in the system
- [Execution Flow](../architecture/execution-flow.md) -- storage is Phase 3 of the preflight pipeline
- [Decision Logic](../architecture/decision-logic.md) -- storage results feed into the READY/DEGRADED/NOT_READY verdict
- [Observability](../architecture/observability.md) -- storage benchmarks emit OTel spans for per-test tracing
- [Orchestrator](orchestrator.md) -- `phase_storage` runs isolation then contention
- [Thresholds](../reference/thresholds.md) -- all threshold values
