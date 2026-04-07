# Observability

OTel traces from every test phase flow to Grafana for debugging preflight results.

## Pipeline

```
Test Script
  └── tracing.py init_tracer()
        └── OTLPSpanExporter (gRPC)
              └── nebius-observability-agent.o11y.svc.cluster.local:4317
                    └── Managed Tempo
                          └── Grafana (Explore → Tempo data source)
```

## Tracer Setup

[`tracing.py`](../components/tracing.md) is imported by every test script:

1. Creates a `Resource` with `service.name`, `k8s.node.name`, `redstone.run_id`
2. Configures `OTLPSpanExporter` pointing at the o11y agent
3. Wraps in `BatchSpanProcessor` for efficient batching
4. Returns a `Tracer` instance

Every script calls [`shutdown_tracer()`](../components/tracing.md) before exit to flush pending spans.

## Service Names

| Service Name | Script |
|-------------|--------|
| `redstone.orchestrator` | orchestrator.py |
| `redstone.gpu_health` | gpu_health.py |
| `redstone.nccl_intranode` | nccl_test.py (TEST_MODE=intranode) |
| `redstone.nccl_multinode` | nccl_test.py (TEST_MODE=multinode) |
| `redstone.ib_topology` | ib_topology.py |
| `redstone.storage` | storage_test.py |
| `redstone.tcp_network` | tcp_test.py |
| `redstone.burnin` | training_burnin.py |

## Key Span Attributes

### Orchestrator Root Span

- `run_id`, `expected_gpu_nodes`
- `verdict`, `duration_s`, `reconcile_count`, `validated_nodes`

### GPU Health

Each check gets its own span (`gpu_health.driver_version`, `gpu_health.matmul_bench`, etc.) with:
- `pass` (boolean)
- Check-specific: `expected`, `actual`, `tflops`, `threshold`, etc.

### IB Topology

- `latency_matrix` span: `measurements` count
- `topology_inference` span: `avg_latency_us`, `spread_us`, `same_leaf_placement`

## Grafana Queries

The [`preflight-traces`](../infrastructure/makefile.md) target prints ready-to-use TraceQL:

```
# Full orchestrator trace
{ resource.service.name = "redstone.orchestrator" }

# GPU health by node
{ resource.service.name = "redstone.gpu_health" && resource.k8s.node.name = "gpu-node-0" }

# Failed tests across all phases
{ span.pass = false }

# IB topology — leaf/spine mapping
{ resource.service.name = "redstone.ib_topology" && span.same_leaf_placement = true }
```

## Infrastructure

- **Nebius o11y agent**: Deployed by Terraform via [enable_nebius_o11y_agent](../infrastructure/terraform.md)
- **Prometheus**: [enable_prometheus](../infrastructure/terraform.md) — local metric storage
- **Grafana dashboard**: Provisioned via [ConfigMap](../infrastructure/terraform.md) from `grafana/preflight-dashboard.json`
- **Grafana data source**: [In-cluster Prometheus](../infrastructure/terraform.md)

## Cross-References

- [Tracing](../components/tracing.md) — the shared OTel setup module
- [Overview](overview.md) — where observability fits in the system
