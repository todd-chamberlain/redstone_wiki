# Changelog

## 2026-04-07 — Final Audit

- Fixed GPU health temperature: "Max 30C" corrected to "Max 32C" to reflect node1 data
- Added node1 per-GPU matmul TFLOPS breakdown to results/gpu-health.md
- Fixed overview.md NVSwitch/IB diagram: swapped numbers to match sources (479 GB/s intra-node from nccl-intranode.json, 482 GB/s multi-node from distributed-16gpu-rdma.json)
- Fixed burnin model name inconsistency: 16-GPU run uses Qwen3-Next-80B-MoE, not Qwen3-Coder-Next (8-GPU only)
- Removed all external file links (../redstone/*.tf, *.yaml, Makefile, Dockerfile) from infrastructure pages — inline references kept as plain text with line numbers
- Fixed tone: removed "worth noting", "worth calling out", "might", hedging language across 5 component pages
- Fixed grammar in overview.md ("why the..." sentence fragment)
- Updated WIKI.md: added results/ and results-data/ to repo structure, removed hardcoded "25 pages" count, replaced "teaching content" with "component walkthroughs"
- Verified all threshold values match JSON sources and code defaults
- Verified pipeline bubble formula, NCCL bus bandwidth, TP scaling ratio, all TFLOPS values against result JSONs
- Confirmed NVLink 4.0 = 900 GB/s bidi (18 links x 50 GB/s), H200 FP16 dense = 990 TFLOPS, NDR = 400 Gb/s, ConnectX-7 HCA, batch_isend_irecv usage — all correct
- No broken internal links found; all .md cross-references resolve to existing pages

## 2026-04-06 — Real Result JSONs Indexed

- Replaced synthetic sample data with 9 actual result JSONs pulled from S3
- Preflight run 20260403: DEGRADED (770 TFLOPS matmul vs 900 threshold due to CUDA 12.4/driver mismatch)
- NCCL intra-node: all_reduce 478.9, all_gather 367.9, reduce_scatter 362.3 GB/s
- Distributed 16-GPU: NCCL 481.72 GB/s cross-node (OPTIMAL), TP scaling 85.1% (OPTIMAL)
- PP=2 bubble 5.2%, hybrid TP4+PP2 7.2% — both OPTIMAL
- IB topology probes failed (IB verbs container access issue), ports confirmed active via ibstat
- Storage: seq read 5.1 GB/s, seq write 389 MB/s (marginal fail), network SSD 99 MB/s (boot disk, not NVMe)
- TCP: 11.39 Gb/s with 57K retransmits
- Called out synthetic sample-results/summary.json as non-representative

## 2026-04-06 — Raw Probe Logs Added

- Copied raw hardware probe logs from redstone/probe/results/ into reference/logs/
- Two GPU node logs with full nvidia-smi, ibstat, NVLink topology, DCGM, CUDA inventory
- Indexed key findings: ConnectX-7 HCAs, NV18 full mesh, 700W power limit, Xeon 8468 CPU
- Linked from validated-results.md with summary table of key data points

## 2026-04-06 — Validated Results Page

- Added reference/validated-results.md with measured performance from actual preflight runs
- Indexed against probe calibration data (2026-04-02, fabric-7, H200, eu-north1)
- Documents what was validated (GPU, NCCL, IB, storage, TCP) and what was not (distributed torchrun, LLM burn-in, FP8, multi-cluster)
- Added to index.md and cross-referenced from thresholds and overview

## 2026-04-06 — Technical Accuracy Audit

- Fixed misleading NVSwitch vs IB diagram in architecture/overview.md: replaced "~50 GB/s" per-port / "9x gap" with actual measured NCCL bus bandwidths (481 GB/s intra-node vs 478 GB/s multi-node). The real gap is latency, not throughput.
- Fixed H100 references to H200 in orchestrator.md (IB_NUM_PORTS comment), distributed-validate.md (RDMA check, TP explanation), training-burnin.md (vLLM TP comment)
- Fixed HDR references to NDR in ib-topology.md (400 Gb/s is NDR, not HDR)
- Verified all threshold values match variables.tf defaults: matmul 900, NCCL all_reduce 450, all_gather 340, reduce_scatter 340, IB BW 390, IB latency 5, storage seq_read 500, seq_write 400, rand_read 5000, contention 50%, TCP 10, min_viable 8, temperature 85
- Verified version numbers: driver 580.95.05, CUDA 13.0 match current defaults
- Verified H200 SXM theoretical peak ~990 FP16 TFLOPS (correct, same compute die as H100)
- Verified NVSwitch 900 GB/s bidirectional raw bandwidth claim (correct for NV18 full mesh)
- Verified NCCL bus bandwidth explanation in nccl-test.md is accurate
- No stale threshold values or version numbers found beyond the fixes above

## 2026-04-06 — Diagrams and Top-Down Navigation

- Added Mermaid diagrams to architecture/overview.md: cluster topology, execution flow, NVSwitch vs IB gap, codebase structure map
- Linked redstone source docs (architecture, diagrams, thresholds, known-limitations) into the wiki graph via overview.md
- Removed Obsidian exclusion of redstone/ — source docs are now navigable in the graph
- overview.md is now the top-down entry point: diagrams → component table → pattern table → source docs

## 2026-04-06 — Setup Docs and Portability

- Rewrote WIKI.md with getting-started guide: clone, submodule init, Obsidian setup
- Added re-indexing instructions for when redstone changes
- Removed hardcoded path references (`Projects/redstone`)
- Updated index.md with Obsidian-first onboarding
- Submodule URL uses relative `../redstone` (resolves to sibling repo on any host)

## 2026-04-06 — Inline Code Rewrite

- Rewrote all 9 component pages as teaching narratives with code embedded inline
- Removed all `../redstone/docker/scripts/*.py` links across every page (319 links total)
- Component pages now walk through each module's logic with fenced Python blocks, explaining the why alongside the what
- Converted code-index.md from a link table to a lookup hub with "full walkthrough" links to component pages
- Updated architecture, pattern, and reference pages to cross-reference component pages instead of raw source files
- All navigation stays within Obsidian — zero clicks open external editors

## 2026-04-06 — Initial Wiki Creation

- Created wiki structure with 25 pages across 5 categories + 3 meta files
- Added `redstone` as git submodule (pinned to `7b88360`)
- Built code-index.md with line-linked entries for all key symbols
- Wrote component pages for all 9 test scripts
- Wrote infrastructure pages for Terraform, Docker, K8s manifests, Makefile
- Wrote architecture pages for overview, execution flow, decision logic, observability
- Wrote pattern pages for ConfigMap thresholds, node tainting, shared-fs coordination, poll-based orchestration, multi-provider inference
- Wrote reference pages for thresholds, environment variables, code index
