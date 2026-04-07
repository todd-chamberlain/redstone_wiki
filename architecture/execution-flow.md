# Execution Flow

End-to-end: `make up` through final verdict.

## Pipeline Stages

### Stage 1-3: Provisioning

```
make up
  → make tf-init        # terraform init
  → make tf-apply       # provision cluster, GPU nodes, o11y, ConfigMap
  → make kubeconfig     # fetch cluster credentials
```

Terraform provisions via the [k8s_training module](../infrastructure/terraform.md), creating:
- Managed K8s cluster with GPU and CPU node groups
- InfiniBand fabric ([`fabric-7`](../infrastructure/terraform.md) for H200)
- Shared filestore (2TB)
- Observability stack (Nebius o11y agent, Prometheus, Grafana)
- [ConfigMap](../infrastructure/terraform.md) with all thresholds
- [S3 credentials secret](../infrastructure/terraform.md)

### Stage 4: Docker Build

```
make docker-build docker-push
```

Builds the [preflight image](../infrastructure/docker.md) based on `pytorch/pytorch:2.11.0-cuda12.8` with fio, iperf3, perftest, nccl-tests, and OTel libraries. The burn-in image adds vLLM and SGLang.

For clusters in eu-north1, [`make build-remote`](../infrastructure/makefile.md) uses Kaniko on a CPU node to avoid transatlantic push.

### Stage 5a: Preflight Setup

[`make preflight-setup`](../infrastructure/makefile.md):
1. Apply [namespace](../infrastructure/kubernetes-manifests.md) + [RBAC](../infrastructure/kubernetes-manifests.md)
2. Create registry pull secret
3. Upload `distributed_validate.py` as ConfigMap (for torchrun mounting)
4. Wait for GPU operator to label nodes with `nvidia.com/gpu.present=true`
5. Verify OTLP endpoint is available

### Stage 5b: Enable RDMA

[`make enable-rdma`](../infrastructure/makefile.md):
1. Patch GPU operator ClusterPolicy to enable RDMA with host MOFED
2. Wait for GPU driver pods to restart
3. Build `nvidia-peermem` kernel module on each GPU node (fixes driver/MOFED ABI mismatch)

### Stage 5c: Preflight Run

[`make preflight-run`](../infrastructure/makefile.md):

1. **Deploy GPU health DaemonSet** — [`gpu-health-daemonset.yaml`](../infrastructure/kubernetes-manifests.md) runs [`gpu_health.py`](../components/gpu-health.md) on every GPU node
2. **Wait for health results** — poll for restart count (DaemonSet pods write results then exit)
3. **Launch orchestrator** — [`orchestrator-job.yaml`](../infrastructure/kubernetes-manifests.md) runs on a CPU node

#### Orchestrator Reconciliation Loop

The [orchestrator](../components/orchestrator.md) runs a reconciliation loop:

```
while time < MAX_RUNTIME (25 min):
    discover GPU nodes via K8s API
    if new nodes found:
        wait for health results from shared FS
        classify: healthy / degraded / failed

    delete GPU health DaemonSet (free GPUs)

    for untested healthy nodes:
        run NCCL intra-node jobs (Phase 2)
        run storage jobs (Phase 3: isolation then contention)

    if >= 2 viable nodes and not yet tested:
        run IB topology discovery (Phase 2.5)
        run TCP iperf3 tests (Phase 4)

    aggregate() → READY / DEGRADED / NOT_READY
    write summary.json

    if all expected nodes validated: break
    wait RECONCILE_INTERVAL (30s)
```

### Stage 5d: Distributed Validation

[`make preflight-distributed`](../infrastructure/makefile.md):

Launches an [indexed Job](../infrastructure/kubernetes-manifests.md) with one pod per GPU node. Pods discover the master IP via shared filesystem, then run [`torchrun`](../components/distributed-validate.md) with:
1. RDMA verification (nvidia-peermem, IB ports)
2. NCCL 8-GPU intra-node all-reduce
3. NCCL 16-GPU cross-node all-reduce (if multi-node)
4. TP=8 vs TP=16 scaling comparison
5. PP=2 pipeline bubble measurement
6. Hybrid TP=8 PP=2

### Stage 6: Burn-in

[`make burnin-training`](../infrastructure/makefile.md):

Launches [burn-in Job](../infrastructure/kubernetes-manifests.md) on a GPU node:
1. Init container verifies model weights staged on shared FS
2. Start inference server (vLLM/SGLang/Ollama)
3. Run eval prompts, measure throughput + latency
4. Upload results to S3

## Cross-References

- [Overview](overview.md)
- [Decision Logic](decision-logic.md) — how `aggregate()` computes the verdict
- [Orchestrator](../components/orchestrator.md) — the reconciliation loop
