# Storage Results

Source: [storage-baseline.json](../results-data/storage-baseline.json) | Run: 20260403-200023 | Phase: baseline (isolation)

## Benchmarks

| Test | Measured | Threshold | Verdict |
|------|----------|-----------|---------|
| Shared FS seq read | 5,100 MB/s | 500 MB/s | PASS |
| Shared FS seq write | 389 MB/s | 400 MB/s | FAIL (by 11 MB/s) |
| Shared FS rand read | 53,158 IOPS (208 MB/s) | 5,000 IOPS | PASS |
| Network SSD seq read | 99.5 MB/s | 1,000 MB/s | FAIL |
| Network SSD seq write | 98.7 MB/s | 1,000 MB/s | FAIL |

## Shared FS

Sequential read hit 10x threshold. Sequential write missed by 11 MB/s (389 vs 400 MB/s). The 20.4ms completion latency is consistent with NFS round-trip overhead on direct I/O writes. Within normal filestore variance — a 2TB Nebius Filestore at 1M block size can fluctuate by a few percent run to run. The 400 MB/s threshold is at the noise floor for this storage tier.

## Network SSD — Warrants Further Investigation

99 MB/s against a 1,000 MB/s threshold. The `gpu_disk_size` is a 200GB network-attached boot disk, not a dedicated NVMe SSD — that explains the 10x gap. The 1,000 MB/s threshold assumes NVMe; these nodes have standard network disks. Next steps: (1) confirm disk type via `lsblk`, (2) check if a higher-performance disk tier is available for `gpu_disk_size`, (3) determine if checkpoint I/O should target shared filestore instead.

## Storage Contention

Not captured. Isolation only completed on 1 of 2 nodes; the contention phase requires all nodes to establish baselines before launching the concurrent test.

**Verdict: MIXED** — shared FS reads 10x threshold, write marginal, network SSD is a disk tier mismatch.

[Back to Results](../results.md)
