# Makefile

Pipeline orchestration: provisions the cluster, builds images, runs preflight, and launches burn-in. Single entry point is `make up`.

## Source

| | |
|---|---|
| **File** | `Makefile` |
| **Lines** | 611 |

## Pipeline Stages

`make up` (L56-L126) runs the full pipeline:

| Stage | Target | Skip Flag |
|-------|--------|-----------|
| 1. Terraform init | `tf-init` | -- |
| 2. Terraform apply | `tf-apply` | -- |
| 3. Kubeconfig | `kubeconfig` | -- |
| 4. Docker build + push | `docker-build`, `docker-push` | `SKIP_DOCKER=1` |
| 5a. Preflight setup | `preflight-setup` | `SKIP_PREFLIGHT=1` |
| 5b. Enable RDMA | `enable-rdma` | `SKIP_PREFLIGHT=1` |
| 5c. Preflight run | `preflight-run` | `SKIP_PREFLIGHT=1` |
| 5d. Distributed validation | `preflight-distributed` | `SKIP_PREFLIGHT=1` |
| 6a. Stage models | `stage-models` | `SKIP_BURNIN=1` (default) |
| 6b. Burn-in | `burnin-training` | `SKIP_BURNIN=1` (default) |

`make down` (L127-L142) destroys with confirmation prompt (bypass with `FORCE=1`).

## Key Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `CONTAINER_CLI` | auto-detected | `container` / `docker` / `podman` |
| `REGISTRY` | `cr.nebius.cloud/your-registry` | Container registry |
| `NAMESPACE` | `redstone` | K8s namespace |
| `RUN_ID` | `$(date +%Y%m%d-%H%M%S)` | Unique run identifier |
| `NEBIUS_PROFILE` | `sandbox` | Nebius CLI profile |
| `SKIP_DOCKER` | `0` | Skip docker build stage |
| `SKIP_PREFLIGHT` | `0` | Skip preflight stages |
| `SKIP_BURNIN` | `1` | Skip burn-in (off by default) |

## Key Targets

### Terraform

| Target | Purpose | Lines |
|--------|---------|-------|
| `tf-init` | `terraform init` | L172-L173 |
| `tf-apply` | `terraform apply -auto-approve` | L179-L181 |
| `tf-destroy` | `terraform destroy -auto-approve` | L183-L185 |

All terraform targets fetch IAM token at shell time via `nebius iam get-access-token`.

### Docker

| Target | Purpose | Lines |
|--------|---------|-------|
| `docker-build` | Build preflight image (linux/amd64) | L204-L205 |
| `docker-push` | Push with retry (5 attempts, auth refresh) | L212-L221 |
| `build-remote` | Kaniko build on CPU node | L237-L262 |

### Preflight

| Target | Purpose | Lines |
|--------|---------|-------|
| `preflight-setup` | Namespace, RBAC, secrets, GPU operator wait | L354-L379 |
| `enable-rdma` | Patch GPU operator, build peermem | L315-L352 |
| `preflight-run` | Deploy DaemonSet + orchestrator | L381-L417 |
| `preflight-distributed` | Launch distributed validation | L536-L548 |
| `preflight-status` | Show job status + recent logs | L419-L423 |
| `preflight-results` | Show verdict (cluster or S3) | L425-L430 |
| `preflight-schedule` | Deploy daily CronJob | L471-L475 |

### Burn-in

| Target | Purpose | Lines |
|--------|---------|-------|
| `stage-models` | Pull models + evals from S3 | L492-L518 |
| `burnin-training` | Launch LLM burn-in job | L559-L570 |

### Debug

| Target | Purpose |
|--------|---------|
| `watch` | Live provision tracker |
| `debug` | 4-pane tmux console (logs + events + pods + GPUs) |
| `status` | Full cluster status |

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- how `make up` provisions the entire system
- [Execution Flow](../architecture/execution-flow.md) -- what `make up` does step-by-step
- [Terraform](terraform.md) -- what `tf-apply` provisions
- [Docker](docker.md) -- what gets built
- [Kubernetes Manifests](kubernetes-manifests.md) -- what gets deployed
