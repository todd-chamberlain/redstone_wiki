# Model and Eval Dataset Staging

## What Was Staged

5 frontier-class models (~3.4 TB total) and 12 eval datasets from S3 to the shared filesystem:

| Model | Architecture | Params | Active | Disk |
|-------|-------------|--------|--------|------|
| Kimi K2.5 | MoE (DeepSeek V3 arch) | 1T | 32B | ~630 GB |
| DeepSeek V3.2 | MoE | 685B | 37B | ~688 GB |
| GLM-5 | MoE (256 experts) | 744B | 40B | ~1,500 GB |
| MiniMax M2.5 | Dense (GQA) | 230B | 10B | ~460 GB |
| Qwen3-Coder-Next | MoE | 80B | 3B | ~160 GB |
| **Total** | | | | **~3.4 TB** |

```
s3://redstone-init/
├── models/
│   ├── kimi-k2.5/           ~630 GB
│   ├── deepseek-v3.2/       ~688 GB
│   ├── qwen3-coder-next/    ~160 GB
│   ├── minimax-m2.5/        ~460 GB
│   └── glm-5/             ~1,500 GB
└── eval-datasets/
    ├── mmlu-pro/        ├── math/
    ├── gsm8k/           ├── bbh/
    ├── humaneval/       ├── ifeval/
    ├── gpqa/            ├── mt-bench/
    ├── swe-bench/       ├── mbpp/
    ├── arena-hard/      └── arena-conversations/
```

## Parallel Staging Pipeline

The staging pipeline (`make stage-models`) parallelizes the S3-to-shared-FS transfer across N pods (default `STAGE_NODES=3`). Each pod runs `s5cmd` for high-throughput parallel downloads, and work is distributed across pods using atomic `mkdir`-based file claiming — each pod attempts to create a directory for a model/dataset, and only the pod that wins the `mkdir` race pulls that artifact. No duplicate downloads and scales linearly: more stager pods = more aggregate S3 bandwidth hitting the shared filesystem.

The approach is necessary because large models (50-200 GB each, GLM-5 at 1.5 TB) would take too long to pull sequentially, and the shared filesystem's write bandwidth is the bottleneck, not S3 read bandwidth.

**Status: VALIDATED** — all models and eval datasets staged successfully.

[Back to Results](../results.md)
