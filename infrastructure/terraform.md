# Terraform

Cluster provisioning using the Nebius k8s-training solution library module.

## Source Files

| File | Lines | Purpose |
|------|-------|---------|
| `main.tf` | 125 | k8s-training module, K8s providers, Volcano, namespace, quota |
| `variables.tf` | 425 | All input variables: cluster config, thresholds, S3, burn-in |
| `preflight-config.tf` | 198 | ConfigMap, Grafana datasources, S3 credentials secret |

## Architecture

### k8s-training Module

`main.tf` L8-L56 wraps the upstream `nebius/nebius-solutions-library/k8s-training` module:

- **GPU nodes**: Configurable count, preset (`8gpu-128vcpu-1600gb`), platform (`gpu-h200-sxm`)
- **CPU nodes**: For system workloads (orchestrator runs here)
- **InfiniBand**: Fabric selection (`fabric-7` for H200 in eu-north1)
- **Shared filestore**: 2TB, 4KB block size
- **Observability**: Nebius o11y agent, Prometheus, Grafana

### Root-Level Resources

Resources outside the module use a separate K8s provider (L61-L66):

| Resource | Purpose |
|----------|---------|
| Volcano Helm release (L78-L93) | Batch scheduler for multi-node GPU jobs |
| Redstone namespace (L96-L106) | Isolation for preflight resources |
| Resource quota (L109-L124) | Prevent preflight from starving other workloads |

### ConfigMap

The redstone-config ConfigMap (`preflight-config.tf` L14-L113) is the central configuration surface. It contains:

- Version matrix (driver, CUDA, NCCL, PyTorch compatibility ranges)
- Hardware expectations (GPU count per node)
- Compute thresholds (matmul TFLOPS)
- NCCL thresholds (per-collective, intra-node and multi-node)
- IB thresholds (per-port bandwidth and latency)
- Storage thresholds (sequential/random, contention degradation)
- TCP threshold
- Distributed validation thresholds (3-level: optimal/pass/floor)
- OTel endpoint
- S3 config for model/dataset storage
- Burn-in defaults (provider, model, eval dataset)

See [ConfigMap Thresholds](../patterns/configmap-thresholds.md) for the design rationale.

### Grafana Integration

- Data sources ConfigMap (`preflight-config.tf` L128-L157) -- in-cluster Prometheus
- Dashboard ConfigMap (`preflight-config.tf` L160-L176) -- auto-provisioned from `grafana/preflight-dashboard.json`

### S3 Credentials

Secret (`preflight-config.tf` L179-L197) injected into pods via `envFrom: secretRef`. Provides `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT_URL`.

## Key Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `gpu_nodes_count` (L45-L49) | 2 | GPU nodes per group |
| `gpu_nodes_platform` (L57-L61) | `gpu-h200-sxm` | GPU platform |
| `infiniband_fabric` (L101-L105) | `fabric-7` | IB fabric (H200) |
| `gpu_disk_size` (L87-L91) | 200 GB | Boot disk (reduced from default 1023) |
| `filestore_disk_size_gibibytes` (L109-L113) | 2048 | Shared FS |

See [Thresholds](../reference/thresholds.md) for all threshold variables.

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- how the provisioned infrastructure maps to system components
- [Execution Flow](../architecture/execution-flow.md) -- Terraform provisions the cluster that the preflight pipeline runs on
- [Observability](../architecture/observability.md) -- Terraform provisions the observability stack (Nebius o11y agent, Prometheus, Grafana)
- [ConfigMap Thresholds](../patterns/configmap-thresholds.md) -- why ConfigMap-driven
- [Environment](../reference/environment.md) -- env vars, secrets, ConfigMap keys
- [Makefile](makefile.md) -- `tf-init`, `tf-apply`, `tf-destroy` targets
