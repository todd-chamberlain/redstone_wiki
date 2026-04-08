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

1. **Server job** on node A -- waits for client requests and starts `ib_send_lat` and `ib_send_bw` listeners on demand
2. **Client job** on node B -- connects to each port, measures latency and bandwidth, builds the matrix

The code uses `ib_send_lat` and `ib_send_bw` (send/recv semantics) instead of `ib_write_lat`/`ib_write_bw` (RDMA write semantics). Send/recv works on Nebius passthrough NICs where RDMA write may not be available.

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

## Server Mode: On-Demand Signal-Based Coordination

Rather than pre-launching all latency and bandwidth servers upfront (which risks port binding conflicts and resource waste), the server mode uses a signal-based protocol coordinated through the shared filesystem. The server waits for client requests and spins up per-port listeners on demand:

```python
def server_mode():
    """Run per-port servers on demand, coordinated with client via shared FS."""
    print(f"[{NODE_NAME}] IB server: waiting for client requests...")
    signal_dir = f"{RESULTS_DIR}/signals"
    os.makedirs(signal_dir, exist_ok=True)

    # Signal that we're ready to accept requests
    ready_file = f"{RESULTS_DIR}/server-ready-{NODE_NAME}-{RUN_ID}"
    with open(ready_file, "w") as f:
        f.write(f"{NODE_NAME}\n")
    print(f"[{NODE_NAME}] Server ready (signaled via {ready_file})")

    served = 0
    total_expected = NUM_PORTS * 2  # latency + BW per port

    while served < total_expected:
        # Check for client requests: "request-lat-0", "request-bw-0", etc.
        for kind in ("lat", "bw"):
            for i in range(NUM_PORTS):
                req_file = f"{signal_dir}/request-{kind}-{i}"
                ack_file = f"{signal_dir}/ack-{kind}-{i}"
                done_file = f"{signal_dir}/done-{kind}-{i}"

                if os.path.exists(req_file) and not os.path.exists(ack_file):
                    dev = f"mlx5_{i}"
                    port = BASE_PORT + i if kind == "lat" else BASE_PORT + 100 + i

                    if kind == "lat":
                        cmd = ["ib_send_lat", "-d", dev, "-p", str(port), "-D", "60", "-n", "1000", "-F"]
                    else:
                        cmd = ["ib_send_bw", "-d", dev, "-p", str(port), "-D", "60", "--report_gbits", "-n", "1000", "-F"]

                    proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
                    time.sleep(1)  # let it bind

                    # Signal that server is listening
                    with open(ack_file, "w") as f:
                        f.write("ready\n")

                    # Wait for client to finish (it writes done file) or timeout
                    for _ in range(30):
                        if os.path.exists(done_file):
                            break
                        time.sleep(2)

                    proc.kill()
                    served += 1
```

The protocol uses three signal files per port per test type:

1. **`request-{kind}-{i}`** -- written by the client to ask for a server on port `i`
2. **`ack-{kind}-{i}`** -- written by the server after the listener is bound
3. **`done-{kind}-{i}`** -- written by the client after the probe completes

This handshake ensures the server is actually listening before the client tries to connect, and the server can kill the listener once the client is done. Each listener only runs for the duration of a single probe, freeing the port for the next test.

The server also watches for client result files and exits early if it detects them, preventing indefinite hanging.

---

## Client Mode: Building the Latency Matrix

The client waits for the server's ready signal, then probes each port sequentially using the same request/ack/done signal protocol:

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

Each latency probe runs `ib_send_lat` with 1000 iterations over 5 seconds. The `-F` flag enables unsorted completion processing for better throughput. The parser extracts the average latency from perftest's tabular output:

```python
def run_latency_client(device: str, port: int, server_ip: str) -> dict:
    cmd = [
        "ib_send_lat", "-d", device, "-p", str(port),
        "-D", "5", "-n", "1000", "-F", server_ip
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

        stderr = result.stderr[:200] if result.stderr else ""
        return {"device": device, "latency_us": -1, "error": "parse_failed", "raw": output[:200], "stderr": stderr}
    except subprocess.TimeoutExpired:
        return {"device": device, "latency_us": -1, "error": "timeout"}
    except Exception as e:
        return {"device": device, "latency_us": -1, "error": str(e)}
```

Perftest output has header lines starting with `#` or `-`, then one data line with the results. The third column (`parts[2]`) is `t_avg[usec]` -- average latency in microseconds. On parse failure, the error result now includes the first 200 characters of stderr, which helps diagnose issues like missing IB devices or permission errors.

### Bandwidth Probing

After latency, the client probes bandwidth on each port using `ib_send_bw` with 1000 iterations. Unlike the old approach that used 4 queue pairs (`-q 4`), the current code uses a single QP with `-n 1000 -F` flags. The `-F` flag enables unsorted completions for more realistic throughput measurement:

```python
def run_bw_probe(device: str, port: int, server_ip: str) -> dict:
    cmd = [
        "ib_send_bw", "-d", device, "-p", str(port),
        "-D", "5", "--report_gbits", "-n", "1000", "-F", server_ip
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
        stderr = result.stderr[:200] if result.stderr else ""
        return {"device": device, "bw_gbps": -1, "error": "parse_failed", "stderr": stderr}
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
