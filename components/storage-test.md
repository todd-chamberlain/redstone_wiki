# Storage Test

## Purpose

GPU training clusters depend on two storage tiers: a shared filesystem (typically Lustre or GPFS) for checkpoint I/O and dataset loading, and local network-attached SSDs for scratch space and fast caching. Both tiers must meet minimum performance requirements, and both degrade under contention when many nodes hit them simultaneously.

`fio` benchmarks five specific I/O patterns that map to real training workloads:

- **Sequential read/write with large blocks** -- this is the checkpoint pattern. When a training run saves or restores a checkpoint, it writes or reads multi-gigabyte files in large sequential chunks.
- **Random read with small blocks** -- this is the DataLoader pattern. PyTorch DataLoader workers read random samples from datasets stored on the shared filesystem, issuing many small random reads concurrently. This is split into cold and warm passes to measure both first-access and cached performance.

The [Orchestrator](orchestrator.md) runs this module twice per node: once in isolation to establish a baseline, and once with all nodes hitting storage simultaneously to measure contention-induced degradation.

| | |
|---|---|
| **Depends on** | [tracing](tracing.md), [configmap](../patterns/configmap-thresholds.md) |
| **Runs as** | Job per GPU node (created by [Orchestrator](orchestrator.md)) |

---

## Paths and Per-Node Isolation

The shared filesystem path is namespaced per node using `NODE_NAME` to prevent file conflicts when multiple nodes run benchmarks simultaneously:

```python
SHARED_FS_PATH = f"/mnt/data/redstone-fio-test/{NODE_NAME}"
LOCAL_SSD_PATH = "/var/tmp/redstone-fio-test"
```

The local SSD path uses `/var/tmp` instead of `/tmp` because `/tmp` is often a tmpfs (RAM-backed) in container environments, which would measure memory speed rather than disk speed. `/var/tmp` is typically backed by the actual block device.

---

## The run_fio Wrapper

Every benchmark flows through a single wrapper function that handles fio invocation, output parsing, and cleanup:

```python
def run_fio(name: str, target_path: str, rw: str, bs: str,
            size: str = "2G", numjobs: int = 8, runtime: int = 10) -> dict:
    os.makedirs(target_path, exist_ok=True)
    # Warm test reuses cold test's file (strip _cold/_warm suffix for consistent filename)
    base_name = name.replace("_cold", "").replace("_warm", "")
    fio_file = os.path.join(target_path, f"fio-{base_name}")
```

### Pre-creation for Random Reads

For random read tests, fio needs a pre-existing file to read from. The module pre-creates the test file using `dd` because fio's default file creation (fallocate) is not supported on virtiofs and falls back to slow sequential zero-write that can timeout on shared filesystems:

```python
    if "rand" in rw and "read" in rw:
        if not os.path.exists(fio_file):
            print(f"  Pre-creating {size} test file for {name}...")
            try:
                subprocess.run(
                    ["dd", "if=/dev/zero", f"of={fio_file}",
                     "bs=1M", f"count={_parse_size_mb(size)}", "conv=fsync"],
                    capture_output=True, timeout=120,
                )
            except subprocess.TimeoutExpired:
                return {"name": name, "error": "test file creation timed out", "pass": False}
```

### Conditional O_DIRECT

Sequential I/O uses `O_DIRECT` (bypass page cache) to measure actual device throughput. Random reads skip `O_DIRECT` because DataLoaders use buffered I/O in practice, and virtiofs `O_DIRECT` random reads perform poorly due to metadata overhead:

```python
    use_direct = 0 if ("rand" in rw and "read" in rw) else 1

    cmd = [
        "fio",
        f"--name={name}",
        f"--filename={fio_file}",
        f"--rw={rw}",
        f"--bs={bs}",
        f"--size={size}",
        f"--numjobs={numjobs}",
        f"--direct={use_direct}",
        "--fallocate=none",
        f"--runtime={runtime}",
        "--time_based",
        "--group_reporting",
        "--output-format=json",
    ]

    result = subprocess.run(cmd, capture_output=True, text=True, timeout=runtime + 30)
```

Key flags:

- **`--direct={use_direct}`** -- conditional: on for sequential (bypass cache), off for random reads (match DataLoader behavior).
- **`--fallocate=none`** -- disables fallocate-based file creation, which is not supported on virtiofs. Without this, fio fails on shared filesystems that use virtiofs passthrough.
- **`--time_based`** combined with `--runtime=10` means fio runs for exactly 10 seconds regardless of how much data it transfers. Without `--time_based`, fio would stop after transferring `--size` bytes, which produces inconsistent test durations across different storage speeds.
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
        # Clean up test files UNLESS this is rand_read_cold (warm reuses the file)
        if "_cold" not in name:
            for f in glob.glob(f"{fio_file}*"):
                try:
                    os.remove(f)
                except OSError:
                    pass
```

The `finally` block cleans up the test file, but skips cleanup for `_cold` tests because the warm pass reuses the same file. This is critical for the cold/warm random read split -- the warm pass measures cached performance against a file that was just read in the cold pass.

---

## The Benchmarks

The module runs four benchmarks against the shared filesystem. Local SSD tests only run if the local SSD path is an actual mount point (not just a directory on the boot disk).

### Shared Filesystem Tests

```python
shared_tests = [
    ("shared_fs.seq_read", SHARED_FS_PATH, "read", "1M", 8, "2G",
     CONFIG.get("storage_seq_read_mbps", 500) * threshold_mult, "bw_mbps"),
    ("shared_fs.seq_write", SHARED_FS_PATH, "write", "1M", 8, "2G",
     CONFIG.get("storage_seq_write_mbps", 400) * threshold_mult, "bw_mbps"),
    ("shared_fs.rand_read_cold", SHARED_FS_PATH, "randread", "4K", 4, "256M",
     CONFIG.get("storage_rand_read_cold_iops", 1500) * threshold_mult, "iops"),
    ("shared_fs.rand_read_warm", SHARED_FS_PATH, "randread", "4K", 4, "256M",
     CONFIG.get("storage_rand_read_warm_iops", 5000) * threshold_mult, "iops"),
]
```

| Test | Block Size | Jobs | Size | Metric | Default Threshold | Simulates |
|---|---|---|---|---|---|---|
| `shared_fs.seq_read` | 1M | 8 | 2G | Bandwidth (MB/s) | 500 | Checkpoint restore |
| `shared_fs.seq_write` | 1M | 8 | 2G | Bandwidth (MB/s) | 400 | Checkpoint save |
| `shared_fs.rand_read_cold` | 4K | 4 | 256M | IOPS | 1500 | First-access DataLoader reads |
| `shared_fs.rand_read_warm` | 4K | 4 | 256M | IOPS | 5000 | Cached DataLoader reads |

Sequential tests use 1 MB blocks with 8 parallel jobs, mirroring how checkpoint frameworks like PyTorch's `torch.save` issue large sequential writes across multiple file handles. The file size is 2G (reduced from the older 10G) and runtime is 10 seconds -- enough to get stable throughput measurements without wasting time on longer tests.

Random reads are split into cold and warm passes. The cold pass (1500 IOPS threshold) measures first-access performance when the file is not in any cache. The warm pass (5000 IOPS threshold) measures cached performance -- the same file is read again, and the page cache should serve most of the requests. Both use 4 parallel jobs and a smaller 256M file to keep the working set manageable. The cold/warm split catches storage systems that have acceptable raw IOPS but poor caching behavior.

### Local SSD Tests

Local SSD tests only run if `/var/tmp/redstone-fio-test` is an actual mount point. On Nebius k8s-training clusters, the boot disk is the only local storage exposed to pods -- there is no separate NVMe. Skipping the test on non-mounted paths avoids testing the boot disk and producing misleading results:

```python
if os.path.ismount(LOCAL_SSD_PATH):
    local_tests = [
        ("network_ssd.seq_read", LOCAL_SSD_PATH, "read", "128K", 4,
         CONFIG.get("storage_local_seq_read_mbps", 500), "bw_mbps"),
        ("network_ssd.seq_write", LOCAL_SSD_PATH, "write", "128K", 4,
         CONFIG.get("storage_local_seq_write_mbps", 400), "bw_mbps"),
    ]
else:
    print(f"  Skipping network_ssd tests: {LOCAL_SSD_PATH} is not a mount point")
```

| Test | Block Size | Jobs | Threshold |
|---|---|---|---|
| `network_ssd.seq_read` | 128K | 4 | 500 MB/s (from ConfigMap) |
| `network_ssd.seq_write` | 128K | 4 | 400 MB/s (from ConfigMap) |

Local SSD thresholds come from the ConfigMap (`storage_local_seq_read_mbps` and `storage_local_seq_write_mbps`) with defaults of 500/400 MB/s. This makes them tunable per cluster rather than hardcoded, allowing operators to adjust for different NVMe hardware specs.

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
    "shared_fs.seq_read":       { "bw_mbps": 612.3, "iops": 612.3, "threshold": 500, "pass": true },
    "shared_fs.seq_write":      { "bw_mbps": 478.1, "iops": 478.1, "threshold": 400, "pass": true },
    "shared_fs.rand_read_cold": { "bw_mbps": 5.9,   "iops": 1520,  "threshold": 1500, "pass": true },
    "shared_fs.rand_read_warm": { "bw_mbps": 22.1,  "iops": 5650,  "threshold": 5000, "pass": true },
    "network_ssd.seq_read":     { "bw_mbps": 1823.5, "threshold": 500, "pass": true },
    "network_ssd.seq_write":    { "bw_mbps": 1512.0, "threshold": 400, "pass": true }
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
