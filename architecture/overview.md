# Architecture Overview

Redstone is a GPU cluster preflight validation suite that runs automated hardware and software checks before training workloads start. It validates the full stack — from individual GPU health through NVLink fabric, InfiniBand topology, shared storage, and TCP control plane — then emits a READY/DEGRADED/NOT_READY verdict.

## Why It Exists

GPU training clusters are complex: 8 GPUs per node connected via NVSwitch, nodes linked over InfiniBand fat trees, shared NFS for checkpoints, and a Kubernetes control plane coordinating everything. Any failure in this stack — a bad GPU, a miscabled IB port, a slow filesystem — silently degrades training throughput or causes hard failures hours into a run.

## Cluster Topology

```mermaid
%%{init: {"securityLevel": "loose", "flowchart": {"useMaxWidth": true, "padding": 12}} }%%
graph TD
    subgraph gpu_pool["GPU Node Pool - InfiniBand Fabric"]
        gn0["GPU Node 0<br/>8x H200 SXM<br/>NVSwitch 900 GB/s"]
        gn1["GPU Node 1<br/>8x H200 SXM<br/>NVSwitch 900 GB/s"]
        gn0 <-->|"IB 400 Gb/s"| gn1
    end

    subgraph cpu_pool["CPU Node Pool 3x"]
        cpu["Orchestrator<br/>Monitoring<br/>System workloads"]
    end

    gpu_pool -->|"/mnt/data"| storage
    cpu_pool -->|"/mnt/data"| storage

    subgraph storage["Storage Layer"]
        fs["Filestore NFS 2TB<br/>Checkpoints + Results"]
        ssd["Network SSD<br/>Node-Local Scratch"]
    end

    gpu_pool -->|"OTLP gRPC"| o11y

    subgraph o11y["Observability Stack"]
        agent["o11y Agent"] --> tempo["Tempo"]
        tempo --> grafana["Grafana"]
    end

    click gn0 "../components/gpu-health.md" "GPU Health Checks"
    click gn1 "../components/gpu-health.md" "GPU Health Checks"
    click cpu "../components/orchestrator.md" "Orchestrator"
    click fs "../components/storage-test.md" "Storage Test"
    click agent "../components/tracing.md" "Tracing Setup"
    click grafana "observability.md" "Observability"

    style gpu_pool fill:none,stroke:#e74
    style cpu_pool fill:none,stroke:#4a9
    style storage fill:none,stroke:#48d
    style o11y fill:none,stroke:#a6d
```

## Preflight Execution Flow

```mermaid
%%{init: {"securityLevel": "loose", "flowchart": {"useMaxWidth": true, "padding": 12}} }%%
flowchart TD
    start([make up]) --> tf["Terraform Apply"]
    tf --> build["Docker Build"]
    build --> setup["Preflight Setup"]
    setup --> phase1

    phase1["Phase 1: GPU Health"] --> eval1{"Healthy?"}

    eval1 -->|"fail"| taint["Taint Node"]
    eval1 -->|"pass"| phase2
    taint --> viable{"GPUs >= min?"}
    viable -->|"no"| abort([NOT_READY])
    viable -->|"yes"| phase2

    phase2["Phase 2: NCCL"] --> phase3["Phase 3: Storage"]
    phase3 --> phase4["Phase 4: IB Topology"]
    phase4 --> phase5["Phase 5: TCP"]
    phase5 --> agg["Aggregate"]

    agg --> ready{{"Verdict"}}
    ready -->|"pass"| r([READY])
    ready -->|"degraded"| d([DEGRADED])
    ready -->|"fail"| nr([NOT_READY])

    r --> dist["Phase 6: Distributed<br/>RDMA, TP, PP"]
    dist --> burnin["Phase 7: Burn-in<br/>LLM Inference"]

    click tf "../infrastructure/terraform.md" "Terraform"
    click build "../infrastructure/docker.md" "Docker Build"
    click setup "../infrastructure/makefile.md" "Makefile"
    click phase1 "../components/gpu-health.md" "GPU Health"
    click taint "../patterns/node-tainting.md" "Node Tainting"
    click phase2 "../components/nccl-test.md" "NCCL Test"
    click phase3 "../components/storage-test.md" "Storage Test"
    click phase4 "../components/ib-topology.md" "IB Topology"
    click phase5 "../components/tcp-test.md" "TCP Test"
    click agg "decision-logic.md" "Decision Logic"
    click dist "../components/distributed-validate.md" "Distributed Validation"
    click burnin "../components/training-burnin.md" "Training Burn-in"

    style start fill:#2a6,color:#fff
    style abort fill:#c33,color:#fff
    style r fill:#2a6,color:#fff
    style d fill:#da3,color:#fff
    style nr fill:#c33,color:#fff
```

## NVSwitch vs InfiniBand: What Actually Matters

Each H200 node has 8x 400 Gb/s NDR InfiniBand ports (mlx5_0 through mlx5_7), aggregating to ~400 GB/s per node. Measured NCCL bus bandwidths:

```mermaid
%%{init: {"securityLevel": "loose", "flowchart": {"useMaxWidth": true, "padding": 12}} }%%
graph LR
    subgraph intra["Intra-Node (NVSwitch)"]
        g0["GPU 0"] <-->|"479 GB/s bus BW"| g7["GPU 7"]
    end

    subgraph inter["Multi-Node (8x NDR IB)"]
        n0["Node 0"] <-->|"482 GB/s bus BW"| n1["Node 1"]
    end

    note["~1:1 throughput<br/>Latency is the real gap"]

    intra --> note
    inter --> note

    click g0 "../components/nccl-test.md" "NCCL Intra-Node"
    click n0 "../components/ib-topology.md" "IB Topology"
    click note "../components/distributed-validate.md" "Distributed Validation"

    style intra fill:none,stroke:#2a6
    style inter fill:none,stroke:#2a6
    style note fill:#48d,color:#fff
```

With 8x 400 Gb/s NDR ports per node, aggregate multi-node NCCL bandwidth (482 GB/s) nearly matches intra-node NVSwitch bandwidth (479 GB/s). The raw NVSwitch fabric provides 900 GB/s bidirectional, but NCCL bus bandwidth at the application level lands around 480 GB/s in both cases.

**Where the gap actually matters is latency**, not throughput. NVSwitch all-reduce completes in a single hop across a full mesh. IB all-reduce traverses a fat-tree topology with 1-3 us per hop depending on leaf/spine placement. For latency-sensitive operations like TP all-reduce (called every transformer layer), this per-operation overhead accumulates. For bulk transfers like PP activation sends, the aggregate bandwidth is what matters, and IB keeps up.

The [NCCL test](../components/nccl-test.md) isolates NVSwitch with `NCCL_IB_DISABLE=1`. The [IB topology](../components/ib-topology.md) maps leaf/spine placement and per-port latency. The [distributed validation](../components/distributed-validate.md) measures the TP=8 vs TP=16 scaling ratio, quantifying the latency gap's practical cost.

## System Components

### Test Scripts (9 scripts)

All scripts live in `docker/scripts/` and share:
- Config from a K8s ConfigMap mounted at `/etc/redstone/config.json`
- OTel tracing via [tracing.py](../components/tracing.md)
- JSON results written to shared filesystem

| Script | Purpose | Runs As | Wiki Page |
|--------|---------|---------|-----------|
| `orchestrator.py` | Sequences phases, aggregates verdict | Job (CPU node) | [Orchestrator](../components/orchestrator.md) |
| `gpu_health.py` | Per-GPU driver/CUDA/matmul checks | DaemonSet (GPU nodes) | [GPU Health](../components/gpu-health.md) |
| `nccl_test.py` | NCCL collective bandwidth (NVSwitch) | Job (per GPU node) | [NCCL Test](../components/nccl-test.md) |
| `ib_topology.py` | IB fat tree latency mapping | Job (server + client) | [IB Topology](../components/ib-topology.md) |
| `storage_test.py` | fio shared FS + local SSD | Job (per GPU node) | [Storage Test](../components/storage-test.md) |
| `tcp_test.py` | iperf3 control plane bandwidth | Job (per pair) | [TCP Test](../components/tcp-test.md) |
| `distributed_validate.py` | torchrun RDMA/TP/PP benchmarks | Job (indexed) | [Distributed Validate](../components/distributed-validate.md) |
| `training_burnin.py` | LLM inference burn-in | Job (GPU node) | [Training Burn-in](../components/training-burnin.md) |
| `tracing.py` | Shared OTel setup | Imported by all | [Tracing](../components/tracing.md) |

### Infrastructure

| Layer      | Purpose                                             | Wiki Page                                                  |
| ---------- | --------------------------------------------------- | ---------------------------------------------------------- |
| Terraform  | Cluster provisioning (Nebius k8s-training module)   | [Terraform](../infrastructure/terraform.md)                |
| Docker     | Container images with CUDA, NCCL tests, fio, iperf3 | [Docker](../infrastructure/docker.md)                      |
| Kubernetes | DaemonSets, Jobs, CronJob, RBAC                     | [K8s Manifests](../infrastructure/kubernetes-manifests.md) |
| Makefile   | Pipeline orchestration: `make up` through verdict   | [Makefile](../infrastructure/makefile.md)                  |

### Design Patterns

| Pattern | Why It Exists | Wiki Page |
|---------|---------------|-----------|
| ConfigMap thresholds | Tune without rebuilding images | [ConfigMap Thresholds](../patterns/configmap-thresholds.md) |
| Node tainting | Auto-exclude bad GPUs from training | [Node Tainting](../patterns/node-tainting.md) |
| Shared FS coordination | Result passing, master IP discovery | [Shared FS Coordination](../patterns/shared-fs-coordination.md) |
| Poll-based orchestration | Handle dynamic node availability | [Poll-Based Orchestration](../patterns/poll-based-orchestration.md) |
| Multi-provider inference | vLLM/SGLang/Ollama abstraction | [Multi-Provider Inference](../patterns/multi-provider-inference.md) |

## Codebase Structure

```mermaid
%%{init: {"securityLevel": "loose", "flowchart": {"useMaxWidth": true, "padding": 12}} }%%
graph TD
    root["redstone/"] --> tf["terraform/"]
    root --> dk["docker/"]
    root --> pf["preflight/"]
    root --> bi["burnin/"]
    root --> gr["grafana/"]

    tf --> tf1["main.tf"]
    tf --> tf6["preflight-config.tf"]
    tf --> tf2["variables.tf"]

    dk --> dk1["Dockerfile"]
    dk --> dk2["scripts/"]

    pf --> pf1["K8s manifests"]

    bi --> bi1["burn-in jobs"]

    gr --> gr1["dashboard.json"]

    click tf "../infrastructure/terraform.md" "Terraform"
    click dk "../infrastructure/docker.md" "Docker"
    click pf "../infrastructure/kubernetes-manifests.md" "K8s Manifests"
    click bi "../components/training-burnin.md" "Burn-in"
    click dk2 "../reference/code-index.md" "Code Index"
    click tf6 "../patterns/configmap-thresholds.md" "ConfigMap Thresholds"
    click tf2 "../reference/thresholds.md" "Thresholds"

    style root fill:#333,color:#fff
```

## Redstone Source Docs

The redstone repo includes its own documentation alongside the code:

- [Architecture](../redstone/docs/architecture.md) — compact architecture reference
- [Diagrams](../redstone/docs/diagrams.md) — Mermaid diagrams: cluster topology, trace flow, execution flow, Gantt timeline, observability stack, NVSwitch vs IB gap
- [Thresholds](../redstone/docs/thresholds.md) — threshold justification with hardware rationale
- [Known Limitations](../redstone/docs/known-limitations.md) — design trade-offs, planned improvements

## Cross-References

- [Execution Flow](execution-flow.md) — step-by-step `make up` to verdict
- [Decision Logic](decision-logic.md) — how READY/DEGRADED/NOT_READY is computed
- [Observability](observability.md) — OTel → Tempo → Grafana pipeline
- [Code Index](../reference/code-index.md) — master symbol lookup table
