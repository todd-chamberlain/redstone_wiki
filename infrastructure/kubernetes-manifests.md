# Kubernetes Manifests

All K8s manifests for the preflight and burn-in suites.

## Preflight Manifests (`preflight/`)

| Manifest | Kind | Purpose |
|----------|------|---------|
| `namespace.yaml` | Namespace | `redstone` namespace |
| `rbac.yaml` | SA, ClusterRole, Role, Bindings | `redstone-runner` service account |
| `gpu-health-daemonset.yaml` | DaemonSet | GPU health on all GPU nodes |
| `orchestrator-job.yaml` | Job | Orchestrator on CPU node |
| `distributed-validate-job.yaml` | Job (Indexed) | torchrun across all GPU nodes |
| `cronjob.yaml` | CronJob | Daily scheduled preflight (04:00 UTC) |
| `nccl-intranode-job.yaml` | Job | NCCL per-node (standalone) |
| `nccl-multinode-job.yaml` | Job | NCCL multi-node (standalone) |
| `ib-topology-job.yaml` | Job | IB topology (standalone) |
| `storage-test-job.yaml` | Job | Storage test (standalone) |
| `tcp-network-test-job.yaml` | Job | TCP test (standalone) |

### RBAC

`rbac.yaml` defines:

- **ServiceAccount** `redstone-runner` with `imagePullSecrets` for Nebius registry
- **ClusterRole** `redstone-node-access`: get/list/patch nodes (needed for [tainting](../patterns/node-tainting.md))
- **Role** `redstone-preflight` (namespace-scoped): create/manage jobs, daemonsets, read configmaps and pods

### GPU Health DaemonSet

`gpu-health-daemonset.yaml`:
- `nodeSelector: nvidia.com/gpu.present: "true"` -- only GPU nodes
- Tolerates `nvidia.com/gpu`, `preflight`, and `redstone` taints
- Requests 8 GPUs, 2 CPU, 8Gi memory
- Mounts ConfigMap at `/etc/redstone` and results hostPath

### Orchestrator Job

`orchestrator-job.yaml`:
- **Anti-affinity**: `nvidia.com/gpu.present DoesNotExist` -- runs on CPU nodes
- `backoffLimit: 0`, `activeDeadlineSeconds: 1800` (30 min)
- Gets `REDSTONE_IMAGE` env for creating child jobs
- S3 credentials via `envFrom: secretRef`

### Distributed Validate Job

`distributed-validate-job.yaml`:
- `completionMode: Indexed` -- one pod per GPU node
- `hostNetwork: true`, `privileged: true` (for RDMA)
- Master IP discovery via shared filesystem (entrypoint script L49-L74)
- Mounts `distributed_validate.py` from ConfigMap (allows updates without image rebuild)
- `/dev/shm` as 32Gi emptyDir (Memory medium) for NCCL shared memory

### CronJob

`cronjob.yaml`:
- Schedule: daily at 04:00 UTC
- `concurrencyPolicy: Forbid` -- never overlap runs
- Keeps 7 successful + 3 failed job history

## Burn-in Manifests (`burnin/`)

| Manifest | Kind | Purpose |
|----------|------|---------|
| `training-job.yaml` | Job | LLM burn-in on GPU node |
| `model-stage-job.yaml` | Job | S3 -> shared FS model/dataset pull |

### Training Job

`training-job.yaml`:
- `schedulerName: volcano` -- GPU-aware gang scheduling
- Init container verifies model weights are staged
- Burn-in config from env vars (`BURNIN_PROVIDER`, `BURNIN_MODEL`, `BURNIN_EVAL_DATASET`)
- S3 credentials via `envFrom: secretRef`

## Template Variables

All manifests use `${VARIABLE}` placeholders substituted by `sed` in the Makefile:
- `${REDSTONE_IMAGE}` -- preflight container image
- `${BURNIN_IMAGE}` -- burn-in container image
- `${RUN_ID}` -- unique run identifier (timestamp)
- `${GPU_NODES}` -- number of GPU nodes (for distributed validate)

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- how manifests map to system components
- [Execution Flow](../architecture/execution-flow.md) -- manifests are deployed as part of the preflight pipeline
- [Makefile](makefile.md) -- targets that apply these manifests
