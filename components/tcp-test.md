# TCP Test

## Purpose

GPU training clusters have two distinct network planes. The **data plane** (InfiniBand) carries gradient and activation traffic between GPUs. The **control plane** (TCP/Ethernet) carries everything else: `kubectl` commands, Prometheus metric scraping, OpenTelemetry trace exports, health checks, and orchestration signals.

iperf3 runs between node pairs to confirm TCP bandwidth is sufficient for these operational services. The threshold is intentionally modest (10 Gb/s) because this path does not carry training data. But degraded TCP connectivity means the cluster loses observability and manageability.

| | |
|---|---|
| **Depends on** | [tracing](tracing.md), [configmap](../patterns/configmap-thresholds.md) |
| **Runs as** | Client Job per node pair (created by [Orchestrator](orchestrator.md)) |

---

## iperf3 Invocation

The [Orchestrator](orchestrator.md) creates an iperf3 server job on one node and a client job on the other (using `hostNetwork: true` for both). The client runs the test:

```python
def run_iperf_client(server_addr: str, duration: int = 20) -> dict:
    cmd = [
        "iperf3", "-c", server_addr,
        "-t", str(duration),
        "-P", "4",   # 4 parallel streams
        "-J",        # JSON output
    ]

    result = subprocess.run(cmd, capture_output=True, text=True, timeout=duration + 30)
```

Flags:

- **4 parallel streams (`-P 4`)**: A single TCP stream on a modern kernel typically cannot saturate a 25 Gb/s or 100 Gb/s NIC due to per-flow CPU bottlenecks. Four streams exercise the NIC while keeping the test lightweight. The goal is verifying the path works, not finding theoretical max.
- **20-second duration (`-t 20`)**: Long enough for TCP congestion control to stabilize past slow-start but short enough to not block the test pipeline.
- **JSON output (`-J`)**: Produces structured output that can be parsed reliably without fragile regex against human-readable tables.

The timeout is set to `duration + 30` seconds. The extra 30 seconds accommodates TCP connection setup, slow-start, and JSON serialization overhead.

---

## Parsing the Results

iperf3's JSON output contains detailed per-stream and aggregate statistics. The module extracts two values: aggregate received bandwidth (the true end-to-end throughput after retransmissions) and the retransmit count:

```python
    try:
        data = json.loads(result.stdout)
        bw_bps = data["end"]["sum_received"]["bits_per_second"]
        bw_gbps = bw_bps / 1e9

        return {
            "bw_gbps": round(bw_gbps, 2),
            "retransmits": data["end"]["sum_sent"].get("retransmits", 0),
            "streams": 4,
            "duration_s": duration,
        }
    except (json.JSONDecodeError, KeyError) as e:
        return {"error": str(e), "bw_gbps": 0, "pass": False}
```

The bandwidth is read from `sum_received` rather than `sum_sent` because received bytes represent what actually made it through the network. Sent bytes include data that may have been lost and retransmitted.

The retransmit count is captured as a diagnostic signal. High retransmits (even with acceptable bandwidth) indicate network congestion or packet loss that could cause latency spikes in Prometheus scrapes or OTel exports. The current implementation records retransmits for the operator to evaluate without failing on them, but this is a natural extension point for future thresholds.

---

## The 10 Gb/s Threshold

The pass/fail decision compares aggregate bandwidth against a single configurable threshold:

```python
def main():
    threshold = CONFIG.get("tcp_bw_gbps_threshold", 10)

    with tracer.start_as_current_span("tcp_network", attributes={
        "node": NODE_NAME, "run_id": RUN_ID,
        "server": SERVER_NODE, "client": CLIENT_NODE,
    }) as root_span:

        with tracer.start_as_current_span("iperf3_bandwidth") as span:
            result = run_iperf_client(SERVER_NODE)
            passed = result.get("bw_gbps", 0) >= threshold

            result["threshold"] = threshold
            result["pass"] = passed

            span.set_attribute("bw_gbps", result.get("bw_gbps", 0))
            span.set_attribute("threshold", threshold)
            span.set_attribute("pass", passed)

        root_span.set_attribute("pass", passed)
```

Why 10 Gb/s? Most GPU cluster nodes have 25 Gb/s or 100 Gb/s Ethernet NICs for the management network. 10 Gb/s gives generous headroom, catching outright failures (bad cables, misconfigured VLANs, downed links) rather than performance-testing the Ethernet fabric. If TCP bandwidth drops below 10 Gb/s, something is fundamentally broken:

| ConfigMap Key | Default | Rationale |
|---|---|---|
| `tcp_bw_gbps_threshold` | 10 Gb/s | Catches broken links without false positives on healthy 25G/100G NICs |

---

## Output Format

Results are written to `<results_base_path>/<run_id>/tcp/tcp-<client>-to-<server>.json`:

```json
{
  "run_id": "20250415-001",
  "timestamp": "2025-04-15T11:00:00Z",
  "server": "gpu-node-1",
  "client": "gpu-node-0",
  "result": {
    "bw_gbps": 24.81,
    "retransmits": 12,
    "streams": 4,
    "duration_s": 20,
    "threshold": 10,
    "pass": true
  }
}
```

The filename encodes the direction (`tcp-<client>-to-<server>`) because TCP performance is sometimes asymmetric -- a VLAN misconfiguration or NIC driver bug affects one direction but not the other. The [Orchestrator](orchestrator.md) can run tests in both directions for each pair.

The exit code reflects the result: 0 for pass, 1 for failure. The orchestrator.s Kubernetes job watcher detects failures from the exit code without parsing JSON.

```python
    shutdown_tracer()
    sys.exit(0 if passed else 1)
```

---

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- where TCP testing fits in the system
- [Execution Flow](../architecture/execution-flow.md) -- TCP is Phase 4 of the preflight pipeline
- [Decision Logic](../architecture/decision-logic.md) -- TCP results feed into the READY/DEGRADED/NOT_READY verdict
- [Observability](../architecture/observability.md) -- iperf3 results are captured as OTel span attributes
- [Orchestrator](orchestrator.md) -- `phase_tcp` creates iperf3 job pairs
- [Thresholds](../reference/thresholds.md) -- all threshold values
