# Environment

All environment variables, secrets, and ConfigMap keys used by the redstone preflight suite.

---

## Environment Variables

### Common (All Scripts)

| Variable | Source | Default | Used By |
|----------|--------|---------|---------|
| `NODE_NAME` | `fieldRef: spec.nodeName` | `unknown` | All scripts |
| `RUN_ID` | Makefile `$(date)` | `unknown` | All scripts |
| `REDSTONE_CONFIG` | — | `/etc/redstone/config.json` | All scripts |
| `PYTHONUNBUFFERED` | Manifest | `1` | All scripts |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Manifest | `http://nebius-observability-agent.o11y.svc.cluster.local:4317` | All scripts |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Manifest | `grpc` | All scripts |

### Orchestrator

| Variable | Source | Default | Purpose |
|----------|--------|---------|---------|
| `REDSTONE_IMAGE` | Makefile sed | — | Container image for child jobs |
| `EXPECTED_GPU_NODES` | — | `2` | Target node count |
| `RECONCILE_INTERVAL` | — | `30` | Seconds between reconciliation loops |
| `MAX_RUNTIME` | — | `1500` | Max orchestrator runtime (25 min) |

### NCCL Test

| Variable | Source | Default | Purpose |
|----------|--------|---------|---------|
| `TEST_MODE` | — | `intranode` | `intranode` or `multinode` |
| `NCCL_IB_DISABLE` | Manifest | `1` | Isolate NVSwitch from IB |
| `NCCL_DEBUG` | Manifest | `INFO` | NCCL debug logging |

### IB Topology

| Variable | Source | Default | Purpose |
|----------|--------|---------|---------|
| `IB_TOPO_ROLE` | Orchestrator | `client` | `server` or `client` |
| `PEER_IP` | Orchestrator | — | Server node IP |
| `SERVER_NODE_NAME` | Orchestrator | — | Server node name (for ready signal) |
| `IB_NUM_PORTS` | Orchestrator | `8` | Number of IB HCA ports |
| `IB_BASE_PORT` | Orchestrator | `19200` | Starting port for perftest |

### Storage Test

| Variable | Source | Default | Purpose |
|----------|--------|---------|---------|
| `TEST_PHASE` | Orchestrator | `baseline` | `isolation` or `concurrent` |

### TCP Test

| Variable | Source | Default | Purpose |
|----------|--------|---------|---------|
| `SERVER_NODE` | Orchestrator | — | iperf3 server IP |
| `CLIENT_NODE` | Orchestrator | — | Client node name |

### Distributed Validation

| Variable | Source | Default | Purpose |
|----------|--------|---------|---------|
| `RANK` | torchrun | `0` | Global rank |
| `WORLD_SIZE` | torchrun | `1` | Total GPU count |
| `LOCAL_RANK` | torchrun | `0` | Per-node rank |
| `MASTER_ADDR` | Shared FS | — | Master node IP |
| `GPU_NODES` | Makefile | — | Number of GPU nodes |
| `NCCL_NET_GDR_LEVEL` | Manifest | `5` | GPUDirect RDMA level |
| `NCCL_IB_DISABLE` | Manifest | `0` | IB enabled for cross-node |
| `NCCL_IB_HCA` | Manifest | `mlx5` | HCA device prefix |
| `NCCL_SOCKET_IFNAME` | Manifest | `eth0` | Control socket interface |
| `GLOO_SOCKET_IFNAME` | Manifest | `eth0` | Gloo backend interface |

### Burn-in

| Variable | Source | Default | Purpose |
|----------|--------|---------|---------|
| `BURNIN_PROVIDER` | Makefile / ConfigMap | `vllm` | Inference provider |
| `BURNIN_MODEL` | Makefile / ConfigMap | `models/qwen3-coder-next/` | Model path |
| `BURNIN_EVAL_DATASET` | Makefile / ConfigMap | `eval-datasets/humaneval/` | Eval dataset |
| `MODEL_PATH` | Manifest | `/mnt/data/models` | Local model directory |
| `EVAL_PATH` | Manifest | `/mnt/data/eval-datasets` | Local eval directory |
| `MASTER_ADDR` | `fieldRef: status.podIP` | — | For distributed training |

---

## K8s Secrets

### `redstone-s3-credentials`

Created by [Terraform](../infrastructure/terraform.md). Injected via `envFrom: secretRef`.

| Key | Purpose |
|-----|---------|
| `AWS_ACCESS_KEY_ID` | S3 access key |
| `AWS_SECRET_ACCESS_KEY` | S3 secret key |
| `AWS_ENDPOINT_URL` | S3 endpoint |
| `AWS_DEFAULT_REGION` | Region |

### `nebius-registry`

Created by [`preflight-setup`](../infrastructure/makefile.md). Docker registry pull secret.

---

## ConfigMap Keys (`redstone-config`)

Full ConfigMap definition: [`preflight-config.tf` L14-L113](../infrastructure/terraform.md)

### Version Matrix

| Key | Default | Script |
|-----|---------|--------|
| `expected_driver_version` | `580.95.05` | gpu_health.py |
| `expected_cuda_version` | `13.0` | gpu_health.py |
| `expected_nccl_version` | (auto) | gpu_health.py |
| `compatibility` | See [variables.tf L230](../infrastructure/terraform.md) | gpu_health.py |

### Thresholds

See [Thresholds](thresholds.md) for the complete table.

### Infrastructure

| Key | Value | Purpose |
|-----|-------|---------|
| `otel_endpoint` | `http://nebius-observability-agent...` | OTel export target |
| `results_base_path` | `/mnt/data/redstone-results` | Shared FS results root |
| `infiniband_fabric` | From Terraform | Trace attribute |
| `gpu_platform` | From Terraform | Trace attribute |
| `sharp_enabled` | `false` | SHARP collective offload |
| `s3_endpoint` | From Terraform | S3 for results upload |
| `s3_bucket` | From Terraform | S3 bucket name |

### Burn-in Defaults

| Key | Default | Purpose |
|-----|---------|---------|
| `burnin_provider` | `vllm` | Default inference provider |
| `burnin_model` | `models/qwen3-coder-next/` | Default model |
| `burnin_eval_dataset` | `eval-datasets/humaneval/` | Default eval dataset |

---

## Makefile Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `CONTAINER_CLI` | Auto-detected | container/docker/podman |
| `REGISTRY` | `cr.nebius.cloud/your-registry` | Container registry |
| `IMAGE_NAME` | `redstone` | Image name |
| `IMAGE_TAG` | `latest` | Image tag |
| `NAMESPACE` | `redstone` | K8s namespace |
| `RUN_ID` | `$(date +%Y%m%d-%H%M%S)` | Unique run ID |
| `NEBIUS_PROFILE` | `sandbox` | Nebius CLI profile |
| `SKIP_DOCKER` | `0` | Skip docker build |
| `SKIP_PREFLIGHT` | `0` | Skip preflight |
| `SKIP_BURNIN` | `1` | Skip burn-in |
| `PUSH_RETRIES` | `5` | Docker push retry count |
| `STAGE_MODELS` | `kimi-k2.5 glm-5` | Models to stage |
| `STAGE_EVALS` | `mmlu-pro gsm8k humaneval ...` | Eval datasets to stage |
| `STAGE_NODES` | `3` | Parallel staging pods |
