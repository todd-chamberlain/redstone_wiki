# Orchestrator

Master sequencer that runs all preflight phases, waits for completion, and aggregates results into a final READY / DEGRADED / NOT_READY verdict. Every test phase is a child of its OpenTelemetry root span.

## Source

| | |
|---|---|
| **File** | `orchestrator.py` |
| **Lines** | ~1119 |
| **Depends on** | [tracing](tracing.md), [configmap](../patterns/configmap-thresholds.md) |
| **Runs as** | Job on GPU node (`orchestrator-job.yaml`) -- runs on GPU nodes for image caching, does not request GPU resources |

---

## 1. Config Bootstrap

Before any test runs, the orchestrator establishes its identity and loads cluster configuration. Four environment variables define the runtime context:

```python
CONFIG_PATH = os.environ.get("REDSTONE_CONFIG", "/etc/redstone/config.json")
NODE_NAME   = os.environ.get("NODE_NAME", "unknown")
RUN_ID      = os.environ.get("RUN_ID", "unknown")
REDSTONE_IMAGE = os.environ.get("REDSTONE_IMAGE", "")
NAMESPACE   = "redstone"
```

`RUN_ID` scopes every Kubernetes Job, result file, and label selector, so multiple preflight runs can coexist without collision. `REDSTONE_IMAGE` is the container image that every child Job will use; it contains all test scripts.

The config file is loaded eagerly and hard-fails if missing, because nothing downstream can function without it:

```python
try:
    with open(CONFIG_PATH) as f:
        CONFIG = json.load(f)
except (FileNotFoundError, json.JSONDecodeError) as e:
    print(f"FATAL: Failed to load config from {CONFIG_PATH}: {e}")
    print("Ensure the redstone-config ConfigMap is mounted at /etc/redstone/")
    sys.exit(1)
```

The results directory is created immediately from the config, namespaced under the run ID:

```python
RESULTS_BASE = CONFIG.get("results_base_path", "/mnt/data/redstone-results")
RESULTS_DIR  = f"{RESULTS_BASE}/{RUN_ID}"
os.makedirs(RESULTS_DIR, exist_ok=True)
```

Three Kubernetes API clients are initialized using in-cluster auth (the orchestrator always runs inside the cluster):

```python
config.load_incluster_config()
core_v1  = client.CoreV1Api()
batch_v1 = client.BatchV1Api()
apps_v1  = client.AppsV1Api()
```

`core_v1` discovers nodes and reads node IPs. `batch_v1` creates and monitors Jobs. `apps_v1` is used later to delete the GPU health DaemonSet once health checks complete (freeing GPUs for NCCL tests).

---

## 2. Helper Functions

### Node Discovery

```python
def get_gpu_nodes() -> list:
    """Get list of GPU node names."""
    nodes = core_v1.list_node(label_selector="nvidia.com/gpu.present=true")
    return [n.metadata.name for n in nodes.items]
```

Called at the top of every reconciliation loop iteration. The label `nvidia.com/gpu.present=true` is applied by the NVIDIA GPU Operator; the orchestrator does not manage this label.

### Polling for Job Completion

The orchestrator uses polling, not Kubernetes watches or callbacks. `wait_for_jobs` is the core polling primitive:

```python
def wait_for_jobs(label_selector: str, expected: int, timeout: int = 600) -> tuple:
    """Wait for jobs to complete."""
    start = time.time()
    while time.time() - start < timeout:
        jobs = batch_v1.list_namespaced_job(
            NAMESPACE, label_selector=label_selector
        )
        succeeded = [j for j in jobs.items if j.status.succeeded and j.status.succeeded >= 1]
        failed    = [j for j in jobs.items if j.status.failed and j.status.failed >= 1]

        if len(succeeded) + len(failed) >= expected:
            return succeeded, failed
        time.sleep(5)

    raise TimeoutError(f"Timeout waiting for jobs ({label_selector})")
```

Key design choices here:

- **5-second poll interval** -- fast enough to notice completion promptly, slow enough to avoid hammering the API server.
- **Success + failure counting** -- the function does not wait for all jobs to succeed. A failed job counts toward completion. A single broken node does not stall the entire run.
- **TimeoutError on expiry** -- callers catch this and continue rather than crashing, so partial results are still aggregated.

`wait_for_phase_jobs` is a convenience wrapper that auto-injects the `redstone.run_id` label:

```python
def wait_for_phase_jobs(app_label: str, expected: int, timeout: int = 600, role: str = None) -> tuple:
    """Wait for jobs with automatic run_id filtering."""
    selector = f"app={app_label},redstone.run_id={RUN_ID}"
    if role:
        selector += f",role={role}"
    return wait_for_jobs(selector, expected, timeout)
```

### Reading Results from the Shared Filesystem

All test phases write JSON results to the shared filesystem rather than passing data through the Kubernetes API (see [Shared FS Coordination](../patterns/shared-fs-coordination.md)). `read_json_results` globs for these files:

```python
def read_json_results(pattern: str) -> list:
    """Read all JSON result files matching glob pattern. Skips corrupt files."""
    results = []
    for path in glob.glob(pattern):
        try:
            with open(path) as f:
                results.append(json.load(f))
        except (json.JSONDecodeError, OSError) as e:
            print(f"  WARNING: Skipping corrupt result file {path}: {e}")
    return results
```

Corrupt files are logged and skipped, not fatal. This matters because a node that crashes mid-write leaves a truncated JSON file. The orchestrator treats that node as "no result received" rather than crashing.

---

## 3. Phase 1: GPU Health

The [GPU Health](gpu-health.md) check runs as a DaemonSet that is deployed *before* the orchestrator starts (by `make preflight-run`). DaemonSet pods do not terminate on their own, so the orchestrator cannot use `wait_for_jobs`. Instead it polls the shared filesystem for result files:

```python
def phase_gpu_health(gpu_nodes: list) -> dict:
    # Poll for result files on the shared filesystem
    results = []
    start = time.time()
    timeout = 300
    while time.time() - start < timeout:
        results = read_json_results(f"{RESULTS_DIR}/gpu-health/*.json")
        if len(results) >= len(gpu_nodes):
            print(f"  Found {len(results)} result files")
            break
        print(f"  Waiting for results... ({len(results)}/{len(gpu_nodes)} files, {int(time.time()-start)}s)")
        time.sleep(10)
    else:
        print(f"  WARNING: Timeout waiting for GPU health results ({len(results)}/{len(gpu_nodes)})")
```

The `for/else` pattern matters here: if the `while` loop exhausts its timeout without breaking, the `else` block runs and the function continues with whatever partial results it has. It does not raise an error. Nodes that never reported are simply absent from the results -- they will not appear in any "healthy" or "failed" list.

After collecting results, nodes are classified into three buckets:

```python
    healthy_nodes = []
    failed_nodes = []
    degraded_nodes = []

    for r in results:
        if r["overall"] == "PASS":
            healthy_nodes.append(r["node"])
        elif r["overall"] == "DEGRADED":
            degraded_nodes.append(r["node"])
        else:
            failed_nodes.append(r["node"])
```

A DEGRADED node has some issue (for example, one GPU running hot) but its GPUs are still functional. The orchestrator treats degraded nodes as *viable* -- they proceed to NCCL and storage testing. Only FAIL nodes are excluded from further phases.

---

## 4. Phase 2: NCCL Intra-Node

The [NCCL test](nccl-test.md) measures GPU-to-GPU bandwidth *within* a single node via NVSwitch/NVLink. The orchestrator creates one Kubernetes Job per healthy node, each requesting all 8 GPUs:

```python
def phase_nccl(healthy_nodes: list) -> dict:
    for node in healthy_nodes:
        job_manifest = {
            "apiVersion": "batch/v1",
            "kind": "Job",
            "metadata": {
                "name": f"redstone-nccl-{node}",
                "namespace": NAMESPACE,
                "labels": {
                    "app": "redstone-nccl-intranode",
                    "type": "preflight",
                    "redstone.run_id": RUN_ID,
                },
            },
            "spec": {
                "backoffLimit": 0,
                "activeDeadlineSeconds": 600,
                "template": {
                    "spec": {
                        "nodeSelector": {"kubernetes.io/hostname": node},
                        "containers": [{
                            "name": "nccl-test",
                            "image": REDSTONE_IMAGE,
                            "command": ["python", "/opt/redstone/nccl_test.py"],
                            "resources": {
                                "limits": {"nvidia.com/gpu": 8, "cpu": "8", "memory": "32Gi"},
                            },
                            "env": [
                                # ...
                                {"name": "NCCL_IB_DISABLE", "value": "1"},
                                {"name": "NCCL_DEBUG", "value": "INFO"},
                            ],
                            # ...
                        }],
                    },
                },
            },
        }
```

Key manifest choices:

1. **`NCCL_IB_DISABLE=1`** -- This forces NCCL to use only the intra-node NVSwitch fabric, ignoring InfiniBand entirely. The goal is to isolate NVLink health from network health. If this test fails, the problem is inside the box, not in the network.

2. **`backoffLimit: 0`** -- No retries. A failing NCCL test should fail fast and report, not retry and mask intermittent hardware issues.

3. **`nodeSelector` pinning** -- Each Job is pinned to a specific node by hostname. This guarantees the test runs on the node it is meant to validate.

The function handles the 409 Conflict (job already exists) without failing, which matters during reconciliation loop retries:

```python
        try:
            batch_v1.create_namespaced_job(NAMESPACE, body=job_manifest)
        except client.exceptions.ApiException as e:
            if e.status == 409:  # Already exists
                print(f"  NCCL job for {node} already exists")
            else:
                raise
```

---

## 5. Phase 2.5: IB Topology Discovery

The [IB Topology](ib-topology.md) phase probes InfiniBand latency between every pair of nodes to map the physical fat-tree topology. This reveals whether two nodes share a leaf switch (~1-3 us latency, 1 hop) or communicate across spine switches (~3-6 us, 3 hops). The topology information feeds into training job placement decisions.

The phase requires at least 2 nodes and tests all unique pairs:

```python
def phase_ib_topology(healthy_nodes: list) -> dict:
    if len(healthy_nodes) < 2:
        return {"skipped": True, "reason": "insufficient nodes"}

    from itertools import combinations
    pairs = list(combinations(healthy_nodes, 2))
```

For each pair, the orchestrator looks up the server node's InternalIP from the Kubernetes API (not DNS, which may not resolve across namespaces):

```python
    for server_node, client_node in pairs:
        node_info = core_v1.read_node(server_node)
        server_ip = None
        for addr in node_info.status.addresses:
            if addr.type == "InternalIP":
                server_ip = addr.address
                break
```

Each pair gets two Jobs -- a server and a client. Both use `hostNetwork: true` and `IPC_LOCK`:

```python
                        "hostNetwork": True,
                        "containers": [{
                            "name": "ib-topo-server",
                            "image": REDSTONE_IMAGE,
                            "command": ["python", "/opt/redstone/ib_topology.py"],
                            "securityContext": {"capabilities": {"add": ["IPC_LOCK"]}},
                            "env": [
                                {"name": "IB_TOPO_ROLE", "value": "server"},
                                {"name": "IB_NUM_PORTS", "value": "8"},
                                {"name": "IB_BASE_PORT", "value": "19200"},
                                # ...
                            ],
                        }],
```

Why these settings:

- **`hostNetwork: true`** -- IB verbs operate at the host network level. Pod networking would add an overlay hop and invalidate latency measurements.
- **`IPC_LOCK` capability** -- Required by `ibv_reg_mr()` to pin memory pages for RDMA. Without it, the kernel refuses to lock the DMA buffers.
- **`IB_NUM_PORTS: 8`** -- Each H200 node has 8 InfiniBand HCA ports (mlx5_0 through mlx5_7). The test probes all 64 port-pair combinations between two nodes.

The server launches first, then the orchestrator waits 5 seconds before launching the client. The client needs the server to be listening before it can connect:

```python
        try:
            batch_v1.create_namespaced_job(NAMESPACE, body=server_job)
        except client.exceptions.ApiException as e:
            if e.status != 409:
                raise

        time.sleep(5)

        try:
            batch_v1.create_namespaced_job(NAMESPACE, body=client_job)
        except client.exceptions.ApiException as e:
            if e.status != 409:
                raise
```

After all client jobs finish, the function summarizes topology:

```python
    all_same_leaf = all(
        r.get("topology", {}).get("same_leaf_placement", False)
        for r in results
    )

    return {
        "results": results,
        "all_same_leaf": all_same_leaf,
        "pairs_tested": len(pairs),
        "pairs_failed": len(failed),
    }
```

The `all_same_leaf` flag is surfaced in the final verdict reason string, because leaf-vs-spine placement affects all-reduce performance during training.

---

## 6. Phase 3: Storage I/O

The [Storage test](storage-test.md) has two sub-phases that must run in a specific order because the second depends on the first for its baseline measurement.

### Phase 3a: Isolation

Each node runs fio alone, one at a time, to measure its maximum bandwidth to the shared filesystem. Sequential execution is deliberate: running them in parallel would turn the isolation test into a contention test.

```python
def phase_storage(viable_nodes: list) -> dict:
    # Phase 3a: Isolation -- run sequentially, one node at a time
    completed_isolation = 0
    for node in viable_nodes:
        _create_storage_job(node, "isolation", suffix)
        completed_isolation += 1
        try:
            wait_for_jobs(
                f"app=redstone-storage,phase=isolation,redstone.run_id={RUN_ID}",
                completed_isolation, timeout=300,
            )
        except TimeoutError:
            print(f"    WARNING: Isolation job timed out for {node}")
```

The `completed_isolation` counter enforces strict serialization: the function waits for all jobs created *so far* to finish before creating the next one.

### Phase 3b: Contention

Once isolation baselines are established, all nodes hammer the shared filesystem simultaneously:

```python
    # Phase 3b: Contention -- all nodes simultaneously
    if len(viable_nodes) >= 2:
        for node in viable_nodes:
            _create_storage_job(node, "concurrent", suffix)

        succeeded, failed = wait_for_jobs(
            f"app=redstone-storage,phase=concurrent,redstone.run_id={RUN_ID}",
            len(viable_nodes), timeout=600,
        )
```

### Degradation Calculation

The key metric is how much each node's write bandwidth drops under contention compared to its isolation baseline:

```python
    contention_max_pct = CONFIG.get("storage_contention_max", 50)
    degradation = {}
    for node in viable_nodes:
        iso = isolation_bw.get(node, 0)
        con = concurrent_bw.get(node, 0)
        if iso > 0:
            pct_drop = round((1 - con / iso) * 100, 1)
            degradation[node] = {
                "isolation_mbps": iso,
                "concurrent_mbps": con,
                "degradation_pct": pct_drop,
                "pass": pct_drop <= contention_max_pct,
            }
```

The default threshold is 50%. If a node's write bandwidth drops more than half under contention, something is wrong (bad NIC, saturated switch, misconfigured filestore). Configurable via `storage_contention_max` in the [ConfigMap](../patterns/configmap-thresholds.md).

---

## 7. Phase 4: TCP Network

The [TCP test](tcp-test.md) measures baseline TCP bandwidth between every node pair using iperf3. Like IB topology, it requires at least 2 nodes and tests all combinations:

```python
def phase_tcp(viable_nodes: list) -> dict:
    if len(viable_nodes) < 2:
        return {"skipped": True}

    from itertools import combinations
    pairs = list(combinations(viable_nodes, 2))
```

Each pair gets a server Job running `iperf3 -s --one-off -J` (listen for one connection, output JSON) and a client Job running `tcp_test.py`. Both use `hostNetwork: true` to measure real network bandwidth without pod overlay overhead:

```python
                        "hostNetwork": True,
                        "containers": [{
                            "name": "iperf-server",
                            "image": REDSTONE_IMAGE,
                            "command": ["iperf3", "-s", "--one-off", "-J"],
                        }],
```

`--one-off` makes the iperf3 server exit after handling one client connection. Without it, the server Job would never complete and the polling loop would time out.

The same server-then-client sequencing pattern from IB topology is used here, with a 5-second sleep between launching the server and client Jobs.

---

## 8. Aggregation: The Verdict Engine

The `aggregate()` function (~170 lines) takes all phase results and produces the final verdict. It implements a cascading decision tree.

### First gate: GPU count

Before looking at any test results, it checks whether enough GPUs are viable at all:

```python
def aggregate(phases: dict) -> dict:
    gpu_health = phases.get("gpu_health", {})
    failed_nodes = gpu_health.get("failed", [])
    healthy_nodes = gpu_health.get("healthy", [])
    degraded_nodes = gpu_health.get("degraded", [])

    # DEGRADED nodes have working GPUs -- count them as viable
    viable_nodes = healthy_nodes + degraded_nodes
    total_viable_gpus = len(viable_nodes) * CONFIG["expected_gpu_count"]
    min_viable = CONFIG["min_viable_gpu_count"]

    if total_viable_gpus < min_viable:
        verdict = "NOT_READY"
        reason = f"Only {total_viable_gpus} viable GPUs, need {min_viable}"
```

The only path to `NOT_READY`. Every other failure mode produces `DEGRADED`, because if you have enough GPUs, you can train, even if not optimally. See [Decision Logic](../architecture/decision-logic.md) for the full rationale.

### Second gate: Node health status

If GPU count is sufficient but some nodes are failed or degraded, the verdict is `DEGRADED` with remediation actions:

```python
    elif failed_nodes or degraded_nodes:
        verdict = "DEGRADED"
        if failed_nodes:
            remediation.append({
                "action": "exclude_failed_nodes",
                "nodes": failed_nodes,
                "detail": "These nodes are tainted NoSchedule and will not receive workloads",
            })
        if degraded_nodes:
            remediation.append({
                "action": "investigate_degraded_nodes",
                "nodes": degraded_nodes,
                "detail": "Check per-node results for specific failing checks (driver, temp, matmul)",
            })
```

### Third gate: Per-phase test results

If all nodes are healthy, the function checks each phase individually (NCCL, storage throughput, storage contention, and IB topology):

```python
        nccl_failed_nodes = [
            r.get("node", "?") for r in nccl_results
            if r.get("overall") == "FAIL"
        ]
        nccl_ok = len(nccl_failed_nodes) == 0

        storage_failed_nodes = [
            r.get("node", "?") for r in storage_results
            if r.get("overall") == "FAIL"
        ]
        storage_ok = len(storage_failed_nodes) == 0

        contention_failed_nodes = [
            node for node, d in storage_degradation.items()
            if not d.get("pass", True)
        ]
        contention_ok = len(contention_failed_nodes) == 0

        ib_topo_ok = ib_topo.get("skipped", False) or all(
            r.get("overall") != "FAIL"
            for r in ib_topo.get("results", [])
        )
```

`ib_topo_ok` defaults to `True` if skipped (single-node cluster). A skipped test never blocks a READY verdict.

If all checks pass, the verdict includes topology context:

```python
        if nccl_ok and storage_ok and contention_ok and ib_topo_ok:
            verdict = "READY"
            reason = "All tests passed"
            if ib_topo.get("all_same_leaf"):
                reason += " (all nodes on same leaf switch)"
            elif not ib_topo.get("skipped"):
                reason += " (nodes span multiple leaf switches)"
```

### Remediation output

Every non-READY verdict includes a `remediation` list with structured actions that downstream automation (or an on-call engineer) can act on:

```python
            remediation.append({
                "action": "investigate_nccl",
                "nodes": nccl_failed_nodes,
                "detail": "Check NVLink health and GPU operator status on these nodes",
            })
```

The function also aggregates per-node storage metrics (min/max/average bandwidth) and NCCL per-node detail into the final summary structure.

---

## 9. The Reconciliation Loop

The `main()` function (~180 lines) implements a reconciliation loop rather than a linear pipeline. See [Poll-Based Orchestration](../patterns/poll-based-orchestration.md) for the reasoning.

### State tracking

Four sets track what has been validated across loop iterations:

```python
EXPECTED_GPU_NODES = int(os.environ.get("EXPECTED_GPU_NODES", "2"))
RECONCILE_INTERVAL = int(os.environ.get("RECONCILE_INTERVAL", "30"))
MAX_RUNTIME = int(os.environ.get("MAX_RUNTIME", "1500"))  # 25 min max

def main():
    validated_nodes = set()       # nodes that passed health check
    nccl_tested    = set()        # nodes with NCCL results
    storage_tested = set()        # nodes with storage results
    multinode_tested = False      # IB topology + TCP done
    phases = {}
```

These sets let the orchestrator avoid re-testing nodes that have already passed. When a new node joins the cluster mid-run, only that node goes through the test pipeline.

### The while loop

```python
    while time.time() - start_time < MAX_RUNTIME:
        current_gpu_nodes = get_gpu_nodes()
        new_nodes = set(current_gpu_nodes) - validated_nodes

        if not current_gpu_nodes:
            time.sleep(RECONCILE_INTERVAL)
            continue
```

Each iteration discovers the current set of GPU nodes and computes the diff against already-validated nodes. If no GPU nodes exist yet (cluster still booting), it sleeps and retries.

### Progressive validation

New nodes go through health checks first. Results are read from the shared filesystem (the DaemonSet writes them):

```python
        if new_nodes:
            health_results = read_json_results(f"{RESULTS_DIR}/gpu-health/*.json")
            tested_in_health = {r["node"] for r in health_results}

            missing = new_nodes - tested_in_health
            if missing:
                for attempt in range(30):
                    health_results = read_json_results(f"{RESULTS_DIR}/gpu-health/*.json")
                    tested_in_health = {r["node"] for r in health_results}
                    missing = new_nodes - tested_in_health
                    if not missing:
                        break
                    time.sleep(10)
```

After health results arrive, the orchestrator deletes the GPU health DaemonSet to free up the GPUs. The DaemonSet claims all 8 GPUs on each node, and the NCCL phase needs those same GPUs:

```python
        if validated_nodes and reconcile_count <= 2:
            try:
                apps_v1.delete_namespaced_daemon_set("redstone-gpu-health", NAMESPACE)
                time.sleep(15)
            except Exception:
                pass
```

The `reconcile_count <= 2` guard prevents repeated delete attempts on later iterations.

### Phase execution with merge

Each single-node phase (NCCL, storage) only runs on nodes that have not been tested yet, and merges results into the cumulative `phases` dict:

```python
        nodes_needing_nccl = [n for n in viable if n not in nccl_tested]
        if nodes_needing_nccl:
            nccl_result = phase_nccl(nodes_needing_nccl)
            existing = phases.get("nccl", {}).get("results", [])
            phases["nccl"] = {"results": existing + nccl_result.get("results", [])}
            nccl_tested.update(nodes_needing_nccl)
```

Multi-node phases (IB topology, TCP) run once when at least 2 nodes are viable:

```python
        if len(viable) >= 2 and not multinode_tested:
            phases["ib_topology"] = phase_ib_topology(viable)
            phases["tcp"] = phase_tcp(viable)
            multinode_tested = True
```

### Exit conditions

The loop has two exit conditions:

```python
        if len(validated_nodes) >= EXPECTED_GPU_NODES and multinode_tested:
            print(f"\nAll {EXPECTED_GPU_NODES} nodes validated with multi-node tests. Done.")
            break

        if len(validated_nodes) >= EXPECTED_GPU_NODES and len(viable) < 2:
            # Edge case: expected 1 node, got it
            print(f"\nAll expected nodes validated (single-node mode). Done.")
            break
```

The second condition handles single-node clusters where multi-node tests are not applicable.

If neither condition is met before `MAX_RUNTIME` expires, the `while/else` block fires and the orchestrator produces a verdict from whatever results it has.

---

## 10. Results and S3 Upload

### Writing the summary

After each reconciliation iteration that produces new results, `write_summary` persists the verdict:

```python
def write_summary(summary: dict):
    """Write summary JSON to results directory."""
    output_path = f"{RESULTS_DIR}/summary.json"
    with open(output_path, "w") as f:
        json.dump(summary, f, indent=2)
```

The summary is overwritten each iteration, so the file always contains the latest verdict.

### Symlinking latest

After the loop exits, a `latest` symlink is created for easy access:

```python
    latest_link = f"{RESULTS_BASE}/latest"
    try:
        os.remove(latest_link)
    except OSError:
        pass
    os.symlink(RESULTS_DIR, latest_link)
```

Consumers can always read `<RESULTS_BASE>/latest/summary.json` without knowing the current run ID.

### S3 persistence

`upload_results_to_s3` walks the entire results directory and uploads every file. It also copies `summary.json` to a well-known `results/latest/summary.json` key for easy retrieval:

```python
def upload_results_to_s3():
    s3_bucket  = CONFIG.get("s3_bucket")
    s3_endpoint = CONFIG.get("s3_endpoint")

    if not s3_bucket:
        print("S3 bucket not configured -- skipping results upload")
        return

    access_key = os.environ.get("AWS_ACCESS_KEY_ID")
    if not access_key:
        print("S3 credentials not available -- skipping results upload")
        return

    # ... boto3 setup ...

    prefix = f"results/{RUN_ID}"
    for root, _dirs, files in os.walk(RESULTS_DIR):
        for fname in files:
            local_path = os.path.join(root, fname)
            rel_path = os.path.relpath(local_path, RESULTS_DIR)
            s3_key = f"{prefix}/{rel_path}"
            s3.upload_file(local_path, s3_bucket, s3_key)

    # Also upload summary as results/latest/summary.json
    if os.path.exists(summary_path):
        s3.upload_file(summary_path, s3_bucket, "results/latest/summary.json")
```

Two design choices:

- **S3 failure is non-fatal.** The entire upload is wrapped in a broad `try/except`. A missing bucket, expired credentials, or network partition will print a warning but will not change the preflight verdict. The local results on the shared filesystem are the source of truth.
- **Credentials come from environment variables** (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) injected from the `redstone-s3-credentials` Kubernetes Secret, not from the config file.

---

## Configuration

| ConfigMap Key | Purpose | Default |
|---|---|---|
| `results_base_path` | Root directory for all result files | `/mnt/data/redstone-results` |
| `expected_gpu_count` | GPUs per node (used in viable GPU calculation) | 8 |
| `min_viable_gpu_count` | Minimum total GPUs for READY/DEGRADED (below = NOT_READY) | 8 |
| `otel_endpoint` | OpenTelemetry collector endpoint injected into all child Jobs | `http://nebius-observability-agent...` |
| `storage_contention_max` | Maximum allowed contention degradation (%) | 50 |
| `s3_bucket` | S3 bucket for result persistence | From Terraform |
| `s3_endpoint` | S3-compatible endpoint URL | From Terraform |

| Environment Variable | Purpose | Default |
|---|---|---|
| `EXPECTED_GPU_NODES` | Number of GPU nodes to wait for | 2 |
| `RECONCILE_INTERVAL` | Seconds between reconciliation iterations | 30 |
| `MAX_RUNTIME` | Maximum orchestrator runtime in seconds | 1500 (25 min) |

## Output

`<RESULTS_DIR>/summary.json`:

```json
{
  "verdict": "READY",
  "reason": "All tests passed (all nodes on same leaf switch)",
  "run_id": "run-20250115-143022",
  "timestamp": "2025-01-15T14:35:12.456Z",
  "duration_s": 372.1,
  "reconcile_count": 3,
  "viable_gpus": 16,
  "healthy_nodes": ["gpu-node-0", "gpu-node-1"],
  "failed_nodes": [],
  "degraded_nodes": [],
  "validated_nodes": ["gpu-node-0", "gpu-node-1"],
  "expected_nodes": 2,
  "storage_aggregate": { "..." : "..." },
  "storage_contention": { "..." : "..." },
  "nccl_per_node": { "..." : "..." },
  "ib_topology": { "all_same_leaf": true, "pairs_tested": 1 },
  "remediation": []
}
```

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- where the orchestrator fits in the system and how components interact
- [Execution Flow](../architecture/execution-flow.md) -- where the orchestrator fits in the overall pipeline
- [Decision Logic](../architecture/decision-logic.md) -- how `aggregate()` makes its verdict
- [Observability](../architecture/observability.md) -- the OTel pipeline that captures the orchestrator's root span and all child spans
- [Poll-Based Orchestration](../patterns/poll-based-orchestration.md) -- why a reconciliation loop instead of Argo or a DAG
- [Shared FS Coordination](../patterns/shared-fs-coordination.md) -- how result files pass between phases
- [GPU Health](gpu-health.md) -- Phase 1 DaemonSet that the orchestrator polls
- [NCCL Test](nccl-test.md) -- Phase 2 intra-node GPU bandwidth test
- [IB Topology](ib-topology.md) -- Phase 2.5 InfiniBand latency mapping
- [Storage Test](storage-test.md) -- Phase 3 fio-based I/O benchmark
- [TCP Test](tcp-test.md) -- Phase 4 iperf3 network bandwidth test
- [ConfigMap Thresholds](../patterns/configmap-thresholds.md) -- how threshold values are managed
