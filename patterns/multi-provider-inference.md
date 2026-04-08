# Multi-Provider Inference

The burn-in test supports three LLM inference providers behind a common interface. Any of the three engines can validate the hardware.

## Provider Architecture

[`training_burnin.py`](../components/training-burnin.md) defines:

1. [`InferenceProvider`](../components/training-burnin.md) — abstract base class
2. [`VLLMProvider`](../components/training-burnin.md) — vLLM
3. [`SGLangProvider`](../components/training-burnin.md) — SGLang
4. [`OllamaProvider`](../components/training-burnin.md) — Ollama

Provider selection via environment variable or ConfigMap:

```python
PROVIDER = os.environ.get("BURNIN_PROVIDER", CONFIG.get("burnin_provider", "vllm"))
```

## Common Interface

| Method | Purpose | Default |
|--------|---------|---------|
| `start()` | Launch serving process | — |
| `stop()` | SIGTERM + wait(30s) + SIGKILL | Base class |
| `health_check()` | HTTP GET health endpoint | `GET /health` |
| `wait_ready(timeout)` | Poll health until ready | Poll every 5s |
| `generate(prompt)` | Send completion request | OpenAI-compatible API |
| `model_name()` | Model identifier for API | — |

## Provider Differences

| Aspect | vLLM | SGLang | Ollama |
|--------|------|--------|--------|
| Server command | `vllm.entrypoints.openai.api_server` | `sglang.launch_server` | `ollama serve` |
| TP flag | `--tensor-parallel-size` | `--tp` | N/A |
| Health endpoint | `/health` | `/health` or `/v1/models` | `/api/tags` |
| Generate API | OpenAI `/v1/completions` | OpenAI `/v1/completions` | `/api/generate` |
| Model loading | From directory | From directory | `ollama create` from Modelfile |
| Use case | Production, multi-GPU | Production, multi-GPU | Smaller models |

## Why Multiple Providers

1. **Hardware validation breadth**: Different serving engines stress different GPU subsystems
2. **Provider comparison**: Measure throughput/latency across engines on same hardware
3. **Flexibility**: Teams using different serving stacks can validate their specific engine
4. **Fallback**: If one provider has a bug on specific hardware, others can still validate

## Configuration

Set via Terraform or Makefile:

```bash
make burnin-training BURNIN_PROVIDER=sglang BURNIN_MODEL=models/llama-3.1-8b/
```

The [training-job.yaml](../infrastructure/kubernetes-manifests.md) passes these as env vars.

## Cross-References

- [Architecture Overview](../architecture/overview.md) — how multi-provider inference fits into the system design
- [Training Burn-in](../components/training-burnin.md) — implementation details
- [Execution Flow](../architecture/execution-flow.md) — Stage 6
