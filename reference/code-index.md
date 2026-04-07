# Code Index

Master symbol lookup. Every entry links to the wiki page where the function is documented with inline code.

For the raw source files, see `redstone/docker/scripts/` in the submodule.

---

## orchestrator.py

1119 lines — [full walkthrough](../components/orchestrator.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `CONFIG_PATH` | 24 | Config file path from env |
| `get_gpu_nodes` | 52-55 | K8s API: find nodes with `nvidia.com/gpu.present=true` label |
| `wait_for_jobs` | 59-73 | Poll K8s jobs until expected count completes or timeout |
| `wait_for_phase_jobs` | 77-82 | Wrapper: adds `run_id` label filter to `wait_for_jobs` |
| `read_json_results` | 85-94 | Glob-read JSON result files, skip corrupt |
| `phase_gpu_health` | 99-139 | Phase 1: poll shared FS for DaemonSet health results |
| `phase_nccl` | 142-220 | Phase 2: create per-node NCCL jobs, wait, read results |
| `phase_ib_topology` | 223-408 | Phase 2.5: IB latency probing between all node pairs |
| `_create_storage_job` | 411-477 | Helper: create fio storage test job for one node |
| `phase_storage` | 480-553 | Phase 3: storage isolation then contention benchmarks |
| `phase_tcp` | 556-691 | Phase 4: iperf3 TCP tests between node pairs |
| `aggregate` | 696-869 | Aggregate all phase results into READY/DEGRADED/NOT_READY verdict |
| `main` | 879-1056 | Reconciliation loop: discover nodes, run phases, emit verdict |
| `write_summary` | 1059-1064 | Write summary JSON to results directory |
| `upload_results_to_s3` | 1067-1114 | Upload run results to S3 for persistence |

---

## gpu_health.py

452 lines — [full walkthrough](../components/gpu-health.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `check_driver_version` | 51-82 | Validate NVIDIA driver against version matrix |
| `check_cuda_version` | 85-127 | Verify CUDA via nvcc or torch.cuda |
| `check_nccl_version` | 130-157 | Verify NCCL via PyTorch, check minimum major version |
| `check_pytorch_version` | 160-182 | Verify PyTorch meets minimum from compatibility matrix |
| `check_gpu_count` | 185-195 | Verify expected number of GPUs visible (critical) |
| `check_ecc_status` | 198-229 | Check ECC mode and uncorrected error counts (critical) |
| `check_utilization` | 232-265 | Check GPU utilization and power state |
| `check_temperature` | 268-298 | Check GPU temperatures against threshold |
| `check_matmul_benchmark` | 301-350 | FP16 matmul TFLOPS benchmark per GPU |
| `taint_node` | 355-372 | Apply NoSchedule taint via K8s API on critical failure |
| `main` | 377-447 | Run all checks, determine PASS/DEGRADED/FAIL, write results |

---

## nccl_test.py

170 lines — [full walkthrough](../components/nccl-test.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `parse_nccl_output` | 39-72 | Parse nccl-tests stdout for bus bandwidth at largest message size |
| `get_threshold` | 75-84 | Get per-collective threshold from ConfigMap |
| `run_nccl_collective` | 87-116 | Run a single NCCL collective benchmark binary |
| `main` | 119-165 | Run all_reduce, all_gather, reduce_scatter; write results |

---

## ib_topology.py

357 lines — [full walkthrough](../components/ib-topology.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `run_latency_server` | 48-51 | Start `ib_write_lat` server on specific device/port |
| `run_latency_client` | 54-85 | Run `ib_write_lat` client, parse average latency |
| `run_bw_probe` | 88-108 | Quick IB write bandwidth probe per port |
| `infer_topology` | 111-165 | Analyze latency matrix to infer leaf/spine groupings |
| `server_mode` | 168-209 | Run latency + BW servers on all ports, signal readiness |
| `client_mode` | 212-344 | Probe all ports, build latency matrix, infer topology, write results |
| `main` | 347-352 | Dispatch to server_mode or client_mode based on IB_TOPO_ROLE |

---

## storage_test.py

185 lines — [full walkthrough](../components/storage-test.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `run_fio` | 40-94 | Run a single fio benchmark, parse JSON output |
| `main` | 97-180 | Run shared FS + local SSD benchmarks, apply thresholds |

---

## tcp_test.py

108 lines — [full walkthrough](../components/tcp-test.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `run_iperf_client` | 38-64 | Run iperf3 client, parse JSON for bandwidth + retransmits |
| `main` | 67-103 | Run iperf3, check against threshold, write results |

---

## distributed_validate.py

714 lines — [full walkthrough](../components/distributed-validate.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `_DEFAULTS` | 49-55 | Hardcoded fallback thresholds for distributed tests |
| `_load_thresholds` | 58-79 | Merge ConfigMap thresholds over defaults |
| `verdict` | 85-104 | Return OPTIMAL/PASS/DEGRADED/FAIL based on thresholds |
| `check_rdma` | 107-158 | Verify GPUDirect RDMA: peermem module, IB ports, GDR |
| `run_nccl_benchmark` | 161-198 | Run `all_reduce_perf` and parse bus bandwidth |
| `benchmark_tp` | 201-279 | Tensor Parallel benchmark: matmul + all-reduce across GPUs |
| `benchmark_pp` | 282-418 | Pipeline Parallel benchmark: P2P send/recv with bubble measurement |
| `benchmark_tp_pp` | 421-562 | Hybrid TP+PP: tensor parallel within stage, pipeline across stages |
| `main` | 565-709 | Run all distributed tests, aggregate verdicts, write results |

---

## training_burnin.py

534 lines — [full walkthrough](../components/training-burnin.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `InferenceProvider` | 70-133 | Base class: start/stop/health_check/generate/wait_ready |
| `VLLMProvider` | 135-155 | vLLM: high-throughput batched inference via OpenAI API |
| `SGLangProvider` | 158-188 | SGLang: RadixAttention runtime |
| `OllamaProvider` | 191-247 | Ollama: local server for smaller models |
| `load_eval_dataset` | 259-320 | Load eval prompts from JSONL/JSON/txt files |
| `run_eval` | 323-374 | Run prompts through provider, collect latency/throughput metrics |
| `main` | 379-484 | Start provider, load eval, run eval, determine PASS/DEGRADED/FAIL |
| `upload_results_to_s3` | 494-529 | Upload burn-in results to S3 |

---

## tracing.py

52 lines — [full walkthrough](../components/tracing.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `init_tracer` | 15-44 | Initialize OTel tracer with OTLP gRPC exporter |
| `shutdown_tracer` | 47-51 | Flush and shut down the tracer provider |

---

## terraform/main.tf

125 lines — [full walkthrough](../infrastructure/terraform.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `module "k8s_training"` | 8-56 | Nebius k8s-training module: cluster, GPU nodes, o11y |
| `provider "kubernetes"` | 61-66 | Root-level K8s provider for resources outside module |
| `helm_release "volcano"` | 78-93 | Volcano batch scheduler for multi-node GPU jobs |
| `kubernetes_namespace "redstone"` | 96-106 | Create redstone namespace |
| `kubernetes_resource_quota` | 109-124 | Resource quota to prevent preflight from starving workloads |

---

## terraform/preflight-config.tf

198 lines — [full walkthrough](../infrastructure/terraform.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `kubernetes_config_map "redstone_config"` | 14-113 | ConfigMap: all thresholds, version matrix, otel endpoint |
| `kubernetes_config_map "grafana_datasources"` | 128-157 | Grafana data sources (Prometheus in-cluster) |
| `kubernetes_config_map "grafana_dashboard"` | 160-176 | Grafana dashboard provisioning |
| `kubernetes_secret "redstone_s3"` | 179-197 | S3 credentials for model/dataset access |

---

## Makefile

611 lines — [full walkthrough](../infrastructure/makefile.md)

| Symbol | Lines | Description |
|--------|-------|-------------|
| `up` | 56-126 | Full provision pipeline: init → apply → build → preflight → burnin |
| `down` | 127-142 | Full teardown with confirmation |
| `tf-init` / `tf-apply` / `tf-destroy` | 172-188 | Terraform lifecycle targets |
| `docker-build` / `docker-push` | 204-232 | Container build + push with retry |
| `build-remote` | 237-262 | Kaniko build on CPU node in eu-north1 |
| `enable-rdma` | 315-352 | Patch GPU operator for RDMA, build nvidia-peermem |
| `preflight-setup` | 354-379 | Namespace + RBAC + registry secret + wait for GPU operator |
| `preflight-run` | 381-417 | Deploy DaemonSet + orchestrator, monitor |
| `preflight-distributed` | 536-548 | Launch distributed validation torchrun job |
| `stage-models` | 492-518 | Pull models + eval datasets from S3 to shared FS |
| `burnin-training` | 559-570 | LLM burn-in: load model, serve, run eval |

---

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- the entry point for understanding how these modules fit together in the system
