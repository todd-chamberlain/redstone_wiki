# Redstone Wiki

Code-indexed knowledge map for the **redstone** GPU cluster preflight validation suite.

New here? Open this repo as an [Obsidian](https://obsidian.md) vault and start with [Architecture Overview](architecture/overview.md).

Setup instructions and conventions: **[WIKI.md](WIKI.md)** | Changelog: **[log.md](log.md)**

---

## Architecture

How redstone works at the system level.

| Page | Description |
|------|-------------|
| [Overview](architecture/overview.md) | What redstone is, why it exists, component map |
| [Execution Flow](architecture/execution-flow.md) | End-to-end: `make up` through verdict |
| [Decision Logic](architecture/decision-logic.md) | READY / DEGRADED / NOT_READY computation |
| [Observability](architecture/observability.md) | OTel → Tempo → Grafana pipeline |

## Components

One page per test script — narrative walkthrough with inline code.

| Page | Script | Purpose |
|------|--------|---------|
| [Orchestrator](components/orchestrator.md) | `orchestrator.py` | Master sequencer + verdict |
| [GPU Health](components/gpu-health.md) | `gpu_health.py` | Per-GPU hardware checks |
| [NCCL Test](components/nccl-test.md) | `nccl_test.py` | Collective bandwidth benchmarks |
| [IB Topology](components/ib-topology.md) | `ib_topology.py` | InfiniBand fat-tree mapping |
| [Storage Test](components/storage-test.md) | `storage_test.py` | fio isolation + contention |
| [TCP Test](components/tcp-test.md) | `tcp_test.py` | iperf3 control plane |
| [Distributed Validate](components/distributed-validate.md) | `distributed_validate.py` | torchrun TP/PP/RDMA |
| [Training Burn-in](components/training-burnin.md) | `training_burnin.py` | Multi-provider inference |
| [Tracing](components/tracing.md) | `tracing.py` | OTel setup shared by all scripts |

## Infrastructure

Provisioning, build, and deploy.

| Page | Description |
|------|-------------|
| [Terraform](infrastructure/terraform.md) | Cluster provisioning (main.tf, variables.tf, preflight-config.tf) |
| [Docker](infrastructure/docker.md) | Container images + Dockerfile layers |
| [Kubernetes Manifests](infrastructure/kubernetes-manifests.md) | DaemonSets, Jobs, CronJob, RBAC |
| [Makefile](infrastructure/makefile.md) | Pipeline orchestration targets |

## Patterns

Design decisions and rationale.

| Page | Description |
|------|-------------|
| [ConfigMap Thresholds](patterns/configmap-thresholds.md) | Why ConfigMap-driven, calibration workflow |
| [Node Tainting](patterns/node-tainting.md) | Auto-exclusion of bad nodes |
| [Shared FS Coordination](patterns/shared-fs-coordination.md) | Master IP discovery, result passing |
| [Poll-Based Orchestration](patterns/poll-based-orchestration.md) | Why no Argo, reconciliation loop design |
| [Multi-Provider Inference](patterns/multi-provider-inference.md) | vLLM / SGLang / Ollama abstraction |

## Results — READY

**[Results](results.md)** | **[Final Run Detail](results/final-run.md)**

NCCL 481 GB/s, TP scaling 85.1%, storage contention under 6%, burn-in 86.6 tok/s. [Per-layer breakdowns](results.md).

## Reference

Quick-lookup tables.

| Page | Description |
|------|-------------|
| [Thresholds](reference/thresholds.md) | All thresholds with code locations |
| [Environment](reference/environment.md) | Env vars, secrets, ConfigMap keys |
| [Code Index](reference/code-index.md) | Master symbol → file:line lookup |
