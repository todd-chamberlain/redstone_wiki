# Docker

Container images for the preflight and burn-in test suites.

## Source

| File | Purpose |
|------|---------|
| `Dockerfile` | Preflight image: CUDA, NCCL tests, fio, iperf3, perftest |
| `Dockerfile.burnin` | Burn-in image: adds vLLM, SGLang |
| `scripts/` | All test scripts (copied to `/opt/redstone/`) |

## Preflight Image

`docker/Dockerfile` -- 44 lines

Base: `pytorch/pytorch:2.11.0-cuda12.8-cudnn9-devel`

### Layer Structure

1. **System tools** (L8-L17): fio, iperf3, perftest, infiniband-diags, iproute2, jq, curl, git
2. **NCCL dev** (L20-L22): libnccl-dev, libnccl2
3. **NCCL tests** (L25-L28): Built from source, binaries symlinked to `/usr/local/bin/` (all_reduce_perf, all_gather_perf, etc.)
4. **Python deps** (L31-L38): opentelemetry, kubernetes, prometheus_client, requests, boto3
5. **Scripts** (L41-L43): Copied to `/opt/redstone/`

### Key Binaries

| Binary | Source | Used By |
|--------|--------|---------|
| `all_reduce_perf` | nccl-tests | nccl_test.py, distributed_validate.py |
| `all_gather_perf` | nccl-tests | nccl_test.py |
| `reduce_scatter_perf` | nccl-tests | nccl_test.py |
| `ib_send_lat` | perftest | ib_topology.py |
| `ib_send_bw` | perftest | ib_topology.py |
| `fio` | apt | storage_test.py |
| `iperf3` | apt | tcp_test.py (client), orchestrator manifests (server) |

## Build Targets

| Makefile Target | Image | Notes |
|-----------------|-------|-------|
| `docker-build` | `$(REDSTONE_IMAGE)` | `--platform linux/amd64` |
| `docker-build-burnin` | `$(BURNIN_IMAGE)` | Uses `Dockerfile.burnin` |
| `build-remote` | `$(REDSTONE_IMAGE)` | Kaniko on CPU node in eu-north1 |

The `docker-push` target retries up to 5 times, refreshing IAM auth on failure.

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- where the container image fits in the system
- [Execution Flow](../architecture/execution-flow.md) -- the Docker image is built and pushed before the preflight pipeline runs
- [Makefile](makefile.md) -- build and push targets
- All [component pages](../components/) -- scripts that run inside this image
