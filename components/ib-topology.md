# IB Topology Discovery

## Purpose

InfiniBand fabrics in GPU clusters are built as fat trees. At the bottom layer, groups of nodes connect to leaf switches. Leaf switches connect upward to spine switches. When two nodes share a leaf switch, traffic between them traverses a single hop with latency around 1-2 microseconds. When they sit under different leaf switches, traffic must cross the spine -- three hops, 3-6 microseconds.

This latency difference matters for distributed training. NCCL's ring and tree algorithms perform best when they can route adjacent ranks through low-latency paths. If the scheduler blindly assigns nodes to a training job without knowing the topology, it places communicating ranks on opposite sides of the spine, adding microseconds of overhead to every collective.

The topology is discovered by probing latency and bandwidth between every HCA port pair across a node pair. The resulting latency matrix classifies each path as same-leaf or cross-spine, feeding into the [Orchestrator](orchestrator.md)'s placement-aware scheduling decisions.

| | |
|---|---|
| **Depends on** | [tracing](tracing.md), [configmap](../patterns/configmap-thresholds.md) |
| **Runs as** | Server + Client Job pairs (created by [Orchestrator](orchestrator.md)) |
| **Pipeline position** | Phase 2.5 -- after intra-node NCCL validates NVSwitch, before multi-node NCCL stresses the IB fabric |

---

## Server/Client Architecture

IB performance testing requires two cooperating processes: one listening, one probing. The [Orchestrator](orchestrator.md) creates two Kubernetes jobs for each node pair:

1. **Server job** on node A -- starts `ib_write_lat` and `ib_write_bw` listeners on all 8 HCA ports
2. **Client job** on node B -- connects to each port, measures latency and bandwidth, builds the matrix

The role is selected via the `IB_TOPO_ROLE` environment variable:

```python
ROLE = os.environ.get("IB_TOPO_ROLE", "client")  # "server" or "client"
NUM_PORTS = int(os.environ.get("IB_NUM_PORTS", "8"))
BASE_PORT = int(os.environ.get("IB_BASE_PORT", "19200"))
```

The `main` function dispatches based on role:

```python
def main():
    if ROLE == "server":
        server_mode()
        shutdown_tracer()
    else:
        client_mode()
```

---

## Server Mode and the Readiness Signal

The server starts listeners on two port ranges: latency servers on ports `BASE_PORT` through `BASE_PORT + 7`, and bandwidth servers on ports `BASE_PORT + 100` through `BASE_PORT + 107`. Each listener binds to a specific Mellanox device (`mlx5_0` through `mlx5_7`):

```python
def server_mode():
    servers = []
    bw_servers = []
    try:
        for i in range(NUM_PORTS):
            dev = f"mlx5_{i}"
            port = BASE_PORT + i
            proc = run_latency_server(dev, port)
            servers.append(proc)

        for i in range(NUM_PORTS):
            dev = f"mlx5_{i}"
            port = BASE_PORT + 100 + i
            cmd = ["ib_write_bw", "-d", dev, "-p", str(port),
                   "-D", "5", "--report_gbits", "-q", "4"]
            proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
            bw_servers.append(proc)

        # Signal readiness via shared filesystem
        ready_file = f"{RESULTS_DIR}/server-ready-{NODE_NAME}-{RUN_ID}"
        with open(ready_file, "w") as f:
            f.write(f"{NODE_NAME}\n")
```

The client cannot connect until the server is listening. Rather than using a fixed sleep (fragile and wasteful), the server writes a sentinel file to the shared filesystem.

The file path includes the `RUN_ID` to prevent stale signals. Without the run ID, a ready file from a previous test run could trick a new client into connecting to a server that no longer exists. Each test run has its own unique signal file, and stale files from old runs are ignored.

---

## Client Mode: Building the Latency Matrix

The client waits for the server's ready signal, then probes each port sequentially. It builds a latency matrix by running `ib_write_lat` against each device:

```python
def client_mode():
    # Wait for server ready signal
    server_node = os.environ.get("SERVER_NODE_NAME", "")
    ready_file = f"{RESULTS_DIR}/server-ready-{server_node}-{RUN_ID}"
    if ready_file:
        for attempt in range(30):
            if os.path.exists(ready_file):
                break
            time.sleep(2)
        else:
            print("WARNING: Server ready signal not found after 60s, proceeding anyway")
```

The wait loop polls every 2 seconds for up to 60 seconds. If the signal never appears, it proceeds anyway with a warning. Partial results are better than hanging indefinitely.

### Latency Probing

Each latency probe runs `ib_write_lat` with 1000 iterations over 3 seconds. The parser extracts the average latency from perftest's tabular output:

```python
def run_latency_client(device: str, port: int, server_ip: str) -> dict:
    cmd = [
        "ib_write_lat", "-d", device, "-p", str(port),
        "-D", "3", "-n", "1000", server_ip
    ]
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=30)
        output = result.stdout

        # Parse: #bytes  #iterations  t_avg[usec]  tps average
        for line in output.split("\n"):
            line = line.strip()
            if line and not line.startswith("#") and not line.startswith("-"):
                parts = line.split()
                if len(parts) >= 3:
                    try:
                        avg_lat = float(parts[2])
                        return {
                            "device": device,
                            "latency_us": avg_lat,
                            "iterations": int(parts[1]),
                        }
                    except (ValueError, IndexError):
                        continue

        return {"device": device, "latency_us": -1, "error": "parse_failed"}
    except subprocess.TimeoutExpired:
        return {"device": device, "latency_us": -1, "error": "timeout"}
```

Perftest output has header lines starting with `#` or `-`, then one data line with the results. The third column (`parts[2]`) is `t_avg[usec]` -- average latency in microseconds.

### Bandwidth Probing

After latency, the client probes bandwidth on each port using `ib_write_bw` with 4 queue pairs (QPs). Multiple QPs are necessary because a single QP cannot saturate a 400 Gb/s NDR link -- the protocol overhead per operation limits single-QP throughput. Using 4 QPs lets the NIC pipeline enough operations to approach line rate:

```python
def run_bw_probe(device: str, port: int, server_ip: str) -> dict:
    cmd = [
        "ib_write_bw", "-d", device, "-p", str(port),
        "-D", "3", "--report_gbits", "-q", "4", server_ip
    ]
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=30)
        for line in result.stdout.split("\n"):
            line = line.strip()
            if line and not line.startswith("#") and not line.startswith("-") and not line.startswith("*"):
                parts = line.split()
                if len(parts) >= 4:
                    try:
                        bw = float(parts[3])
                        return {"device": device, "bw_gbps": bw}
                    except (ValueError, IndexError):
                        continue
        return {"device": device, "bw_gbps": -1, "error": "parse_failed"}
    except Exception as e:
        return {"device": device, "bw_gbps": -1, "error": str(e)}
```

---

## Topology Inference: The Spread-Based Classification

Once the latency matrix is built, `infer_topology` classifies each path. Rather than using an absolute latency threshold to distinguish same-leaf from cross-spine, the algorithm uses **spread**: the difference between the highest and lowest latency measurements.

```python
def infer_topology(latency_matrix: list) -> dict:
    latencies = [r["latency_us"] for r in latency_matrix if r["latency_us"] > 0]
    if not latencies:
        return {"error": "no valid latency measurements"}

    avg = sum(latencies) / len(latencies)
    min_lat = min(latencies)
    max_lat = max(latencies)
    spread = max_lat - min_lat

    if spread < 1.0:
        # All measurements are similar -- same leaf switch
        same_leaf = [r for r in latency_matrix if r["latency_us"] > 0]
        cross_spine = []
    else:
        midpoint = (min_lat + max_lat) / 2
        same_leaf = [r for r in latency_matrix if 0 < r["latency_us"] <= midpoint]
        cross_spine = [r for r in latency_matrix if r["latency_us"] > midpoint]
```

Why spread-based instead of absolute thresholds? Because absolute latency varies by hardware generation, cable length, and switch firmware. A pair of nodes can both be on the same leaf but show 2.5 us instead of 1.5 us due to longer cables. What stays consistent is the *relative* difference: same-leaf paths cluster together, and cross-spine paths cluster higher. A spread under 1.0 us means everything is in one cluster (same leaf). A spread above 1.0 us means there are two distinct clusters, and the midpoint separates them.

The function then groups results by local device to build leaf group mappings, showing which local ports connect through the same leaf as the peer:

```python
    leaf_groups = defaultdict(list)
    for r in latency_matrix:
        if r["latency_us"] > 0:
            bucket = "leaf" if id(r) in same_leaf_set else "spine"
            leaf_groups[r["local_device"]].append({
                "peer_device": r["peer_device"],
                "latency_us": r["latency_us"],
                "path": bucket,
            })

    return {
        "avg_latency_us": round(avg, 3),
        "min_latency_us": round(min_lat, 3),
        "max_latency_us": round(max_lat, 3),
        "spread_us": round(spread, 3),
        "same_leaf_count": len(same_leaf),
        "cross_spine_count": len(cross_spine),
        "same_leaf_placement": len(cross_spine) == 0,
        "leaf_groups": dict(leaf_groups),
    }
```

---

## Thresholds and Pass/Fail

After topology inference, the client checks two independent thresholds:

```python
ib_lat_threshold = CONFIG.get("ib_latency_us_threshold", 5)
ib_bw_threshold = CONFIG.get("ib_bw_gbps_threshold", 390)

lat_pass = all(
    r["latency_us"] <= ib_lat_threshold
    for r in latency_matrix if r["latency_us"] > 0
)
bw_pass = all(
    r["bw_gbps"] >= ib_bw_threshold
    for r in bw_results if r["bw_gbps"] > 0
)

overall = "PASS" if (lat_pass and bw_pass) else "DEGRADED"
```

| ConfigMap Key | Default | Rationale |
|---|---|---|
| `ib_latency_us_threshold` | 5 us | Even cross-spine paths should stay under 5 us in a healthy fabric |
| `ib_bw_gbps_threshold` | 390 Gb/s | NDR ports are rated at 400 Gb/s; 390 allows for minor overhead |

Both thresholds must be met for all ports. A single port dropping below 390 Gb/s indicates a cable or transceiver issue. A single port exceeding 5 us indicates a path anomaly.

---

## Output Format

Results are written to `<results_base_path>/<run_id>/ib_topology/<node_name>.json`:

```json
{
  "node": "gpu-node-1",
  "peer": "10.0.1.5",
  "run_id": "20250415-001",
  "latency_matrix": [
    {"device": "mlx5_0", "latency_us": 1.82, "local_device": "mlx5_0", "peer_device": "mlx5_0"},
    {"device": "mlx5_1", "latency_us": 1.91, "local_device": "mlx5_1", "peer_device": "mlx5_1"}
  ],
  "bandwidth": [
    {"device": "mlx5_0", "bw_gbps": 395.2},
    {"device": "mlx5_1", "bw_gbps": 396.8}
  ],
  "topology": {
    "avg_latency_us": 1.87,
    "spread_us": 0.3,
    "same_leaf_placement": true,
    "same_leaf_count": 8,
    "cross_spine_count": 0
  },
  "overall": "PASS"
}
```

---

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- where IB topology fits in the system
- [Execution Flow](../architecture/execution-flow.md) -- IB topology is Phase 2.5 of the preflight pipeline
- [Decision Logic](../architecture/decision-logic.md) -- topology results feed into the READY/DEGRADED/NOT_READY verdict
- [Observability](../architecture/observability.md) -- latency and bandwidth measurements are recorded as OTel span attributes
- [Orchestrator](orchestrator.md) -- `phase_ib_topology` creates server/client job pairs
- [Shared FS Coordination](../patterns/shared-fs-coordination.md) -- the ready-signal pattern used here
- [Thresholds](../reference/thresholds.md) -- all threshold values
