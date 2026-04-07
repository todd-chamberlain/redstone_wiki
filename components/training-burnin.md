# Training Burn-in

Preflight checks validate that hardware is functional. Burn-in validates that the full AI stack works end-to-end: model weights load into GPU memory, an inference server starts and serves requests, and the model produces coherent output at acceptable throughput. The difference: "the GPU can do math" and "the GPU can serve a language model."


## What Burn-in Validates

By the time burn-in runs, per-node GPU health, NCCL bandwidth, and storage I/O have already passed. Burn-in answers a different class of question:

- Can the model weights (often 50-140 GB) load into GPU memory without OOM?
- Does the inference server start, bind to a port, and respond to HTTP health checks?
- Does the model produce output (not just empty strings or errors)?
- What throughput and latency does this node achieve?

A node that passes GPU health but fails burn-in usually has a software problem: wrong CUDA version, missing library, model file corruption, or a misconfigured tensor-parallel size.

## Provider Architecture

The burn-in supports three inference engines. Rather than scattering engine-specific logic throughout the code, it uses a simple base class to define the interface:

```python
class InferenceProvider:
    """Base class for inference providers."""

    def __init__(self, model_dir: str):
        self.model_dir = model_dir
        self.process = None

    def start(self) -> subprocess.Popen:
        raise NotImplementedError

    def health_check(self) -> bool:
        try:
            resp = requests.get(f"{API_BASE}/health", timeout=5)
            return resp.status_code == 200
        except Exception:
            return False

    def wait_ready(self, timeout: int = 600) -> bool:
        """Poll until server is ready or timeout."""
        start = time.time()
        while time.time() - start < timeout:
            if self.health_check():
                elapsed = int(time.time() - start)
                print(f"  Server ready after {elapsed}s")
                return True
            time.sleep(5)
        return False

    def generate(self, prompt: str, max_tokens: int = 256) -> dict:
        """Send a completion request via OpenAI-compatible API."""
        resp = requests.post(
            f"{API_BASE}/v1/completions",
            json={
                "model": self.model_name(),
                "prompt": prompt,
                "max_tokens": max_tokens,
                "temperature": 0.0,
            },
            timeout=120,
        )
        resp.raise_for_status()
        data = resp.json()
        return {
            "text": data["choices"][0]["text"],
            "usage": data.get("usage", {}),
            "latency_ms": resp.elapsed.total_seconds() * 1000,
        }

    def stop(self):
        if self.process:
            self.process.send_signal(signal.SIGTERM)
            try:
                self.process.wait(timeout=30)
            except subprocess.TimeoutExpired:
                self.process.kill()
```

`generate()` and `health_check()` have default implementations targeting the OpenAI-compatible API (`/v1/completions` and `/health`). Providers that follow this convention (vLLM, SGLang) inherit them without modification. Providers with different APIs (Ollama) override them. The `stop()` method sends SIGTERM first and escalates to SIGKILL after 30 seconds, giving inference servers time to release GPU memory cleanly.

The 600-second `wait_ready` timeout is generous by design. Large models (70B+ parameters) can take 5-8 minutes to load weights, initialize KV caches, and compile CUDA graphs. Polling every 5 seconds keeps log noise low without missing the ready signal by much.

## VLLMProvider

```python
class VLLMProvider(InferenceProvider):
    def start(self) -> subprocess.Popen:
        import torch
        num_gpus = torch.cuda.device_count()

        cmd = [
            "python", "-m", "vllm.entrypoints.openai.api_server",
            "--model", self.model_dir,
            "--host", SERVER_HOST,
            "--port", str(SERVER_PORT),
            "--tensor-parallel-size", str(num_gpus),
            "--trust-remote-code",
            "--dtype", "auto",
        ]
        self.process = subprocess.Popen(
            cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
        )
        return self.process
```

The `--tensor-parallel-size` is set to the total GPU count on the node (8 for H200 SXM nodes). The model is sharded across all available GPUs automatically. The `--dtype auto` flag lets vLLM pick the optimal dtype from the model config (usually bfloat16 for H200s). Since vLLM exposes an OpenAI-compatible API at `/v1/completions` and a `/health` endpoint, the base class `generate()` and `health_check()` work without modification.

## SGLangProvider

SGLang is mostly identical in structure but has one quirk in health checking:

```python
class SGLangProvider(InferenceProvider):
    def start(self) -> subprocess.Popen:
        import torch
        num_gpus = torch.cuda.device_count()

        cmd = [
            "python", "-m", "sglang.launch_server",
            "--model-path", self.model_dir,
            "--host", SERVER_HOST,
            "--port", str(SERVER_PORT),
            "--tp", str(num_gpus),
            "--trust-remote-code",
        ]
        self.process = subprocess.Popen(
            cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
        )
        return self.process

    def health_check(self) -> bool:
        try:
            resp = requests.get(f"{API_BASE}/health", timeout=5)
            return resp.status_code == 200
        except Exception:
            try:
                resp = requests.get(f"{API_BASE}/v1/models", timeout=5)
                return resp.status_code == 200
            except Exception:
                return False
```

The dual health check pattern exists because SGLang versions are inconsistent about exposing `/health`. Older versions only have `/v1/models`. Rather than pinning to a specific SGLang version, the provider tries `/health` first (fast path) and falls back to `/v1/models`. Resilient across SGLang upgrades.

## OllamaProvider

Ollama is the outlier. It uses a completely different API surface and has a unique model creation step:

```python
class OllamaProvider(InferenceProvider):
    def start(self) -> subprocess.Popen:
        cmd = ["ollama", "serve"]
        self.process = subprocess.Popen(
            cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
            env={**os.environ, "OLLAMA_HOST": f"{SERVER_HOST}:{SERVER_PORT}"},
        )
        # Create model from local weights
        time.sleep(5)
        model_tag = Path(self.model_dir).name
        result = subprocess.run(
            ["ollama", "create", model_tag, "-f", f"{self.model_dir}/Modelfile"],
            capture_output=True, text=True, timeout=600,
        )
        if result.returncode != 0:
            print(f"  WARNING: ollama create failed: {result.stderr[:200]}")
        return self.process

    def health_check(self) -> bool:
        try:
            resp = requests.get(f"{API_BASE}/api/tags", timeout=5)
            return resp.status_code == 200
        except Exception:
            return False

    def generate(self, prompt: str, max_tokens: int = 256) -> dict:
        model_tag = Path(self.model_dir).name
        resp = requests.post(
            f"{API_BASE}/api/generate",
            json={
                "model": model_tag,
                "prompt": prompt,
                "stream": False,
                "options": {"num_predict": max_tokens, "temperature": 0.0},
            },
            timeout=120,
        )
        resp.raise_for_status()
        data = resp.json()
        return {
            "text": data.get("response", ""),
            "usage": {
                "prompt_tokens": data.get("prompt_eval_count", 0),
                "completion_tokens": data.get("eval_count", 0),
            },
            "latency_ms": resp.elapsed.total_seconds() * 1000,
        }
```

Key differences from vLLM/SGLang:

1. **Model creation from Modelfile.** Unlike vLLM/SGLang which load model weights directly from a directory, Ollama requires an explicit `ollama create` step that reads a `Modelfile` (Ollama's configuration format) from the model directory. The 5-second sleep before the create command is a pragmatic wait for the server to bind its port.

2. **Different API endpoints.** Health goes to `/api/tags` (list available models), and generation goes to `/api/generate` instead of `/v1/completions`. The response format also differs: output text is in `response` instead of `choices[0].text`, and token counts use Ollama-specific field names like `eval_count`.

3. **The `model_name` override.** Ollama references models by tag name (the directory basename), not by path. The `model_name()` method returns `Path(self.model_dir).name` to match what `ollama create` registered.

All three providers are registered in a simple dispatch dict:

```python
PROVIDERS = {
    "vllm": VLLMProvider,
    "sglang": SGLangProvider,
    "ollama": OllamaProvider,
}
```

## Eval Dataset Loading

The eval runner needs to be flexible about input formats because eval datasets come from many sources (HuggingFace repos, internal benchmarks, plain prompt lists):

```python
def load_eval_dataset(dataset_path: str) -> list:
    prompts = []
    path = Path(dataset_path)

    if not path.exists():
        return [{"prompt": "What is 2+2?", "expected": "4"}]

    skip_dirs = {".cache", "model_answer", "model_judgment", "__pycache__"}
    for f in sorted(path.rglob("*")):
        if any(skip in f.parts for skip in skip_dirs):
            continue
        if not f.is_file():
            continue
        if f.suffix == ".jsonl":
            # Each line: {"prompt": "...", "answer": "..."}
            # Also accepts "question" and "input" as prompt field names
            ...
        elif f.suffix == ".json":
            # List of strings or list of dicts
            ...
        elif f.suffix == ".txt":
            # One prompt per line
            ...

    if not prompts:
        prompts = [
            {"prompt": "What is the capital of France?", "expected": "Paris"},
            {"prompt": "Write a Python function to compute fibonacci numbers.", "expected": ""},
            {"prompt": "Explain the difference between TCP and UDP in one paragraph.", "expected": ""},
        ]
    return prompts
```

Design decisions:

- **`rglob("*")`** walks subdirectories recursively because HuggingFace dataset repos often nest data files under `data/` subdirectories.
- **`skip_dirs`** filters out directories that contain reference data, not prompts. The `model_answer` and `model_judgment` directories are MT-Bench artifacts that would pollute the prompt list.
- **JSONL field flexibility** -- the loader accepts `prompt`, `question`, or `input` as the prompt field name, and `answer` or `expected` as the ground truth. This handles both internal datasets and common open-source formats without requiring a schema mapping config.
- **Fallback prompts** -- if the dataset is missing or empty, the loader returns three hardcoded prompts. The burn-in still runs and produces a verdict. A missing dataset is a configuration problem, not a hardware problem, and should not block node validation.

## run_eval

The evaluation loop runs prompts through the provider and collects timing metrics:

```python
def run_eval(provider: InferenceProvider, prompts: list, max_prompts: int = 50) -> dict:
    prompts = prompts[:max_prompts]
    results = []
    total_tokens = 0
    total_latency_ms = 0

    for i, item in enumerate(prompts):
        response = provider.generate(item["prompt"], max_tokens=256)
        elapsed_ms = (time.time() - start) * 1000

        tokens = response.get("usage", {}).get("completion_tokens", 0)
        total_tokens += tokens
        total_latency_ms += elapsed_ms

        # Progress every 10 prompts
        if (i + 1) % 10 == 0:
            tps = total_tokens / (total_latency_ms / 1000)
            print(f"  [{i+1}/{len(prompts)}] avg_lat={avg_lat:.0f}ms tps={tps:.1f} tokens/s")

    # Aggregate
    successful = [r for r in results if not r.get("error")]
    failed = [r for r in results if r.get("error")]
    latencies = [r["latency_ms"] for r in successful]

    return {
        "total_prompts": len(prompts),
        "successful": len(successful),
        "failed": len(failed),
        "total_tokens": total_tokens,
        "avg_latency_ms": round(sum(latencies) / len(latencies), 1),
        "p50_latency_ms": round(sorted(latencies)[len(latencies) // 2], 1),
        "p95_latency_ms": round(sorted(latencies)[int(len(latencies) * 0.95)], 1),
        "tokens_per_second": round(total_tokens / (total_latency_ms / 1000), 1),
        "errors": [r["error"] for r in failed[:5]],  # cap at 5 to avoid huge output
    }
```

The `max_prompts=50` cap prevents the burn-in from running for hours on large datasets. Fifty prompts is enough to get stable latency percentiles while keeping total burn-in time under 15 minutes even for large models. The error list is capped at 5 entries because a server that is failing tends to fail the same way every time, and hundreds of identical error strings waste log space.

The metrics chosen (tokens/s, avg, p50, p95 latency) match what you would monitor in production serving. P50 vs P95 divergence reveals whether the server has a long-tail latency problem (e.g., KV cache eviction, PagedAttention recomputation).

## Verdict Logic

The overall verdict is derived from the success rate of eval prompts:

```python
if eval_results["failed"] == 0 and eval_results["successful"] > 0:
    results["overall"] = "PASS"
elif eval_results["successful"] > eval_results["failed"]:
    results["overall"] = "DEGRADED"
else:
    results["overall"] = "FAIL"
```

Simpler than the threshold-based verdict system in [distributed validation](distributed-validate.md). Burn-in is a binary capability test: either the model serves or it does not. The DEGRADED state catches the edge case where a server starts successfully but occasionally returns errors (e.g., intermittent OOM under load, timeout on long prompts). Investigate DEGRADED nodes before assigning production workloads.

## S3 Upload

After writing results to local disk, the burn-in uploads them to S3 for persistence beyond the Job's lifecycle:

```python
def upload_results_to_s3():
    s3_bucket = CONFIG.get("s3_bucket")
    s3_endpoint = CONFIG.get("s3_endpoint")

    if not s3_bucket:
        return  # S3 not configured

    s3 = boto3.session.Session().client(
        "s3",
        endpoint_url=s3_endpoint,
        config=BotoConfig(retries={"max_attempts": 3, "mode": "adaptive"}),
    )

    prefix = f"results/{RUN_ID}/burnin"
    for root, _dirs, files in os.walk(RESULTS_DIR):
        for fname in files:
            local_path = os.path.join(root, fname)
            rel_path = os.path.relpath(local_path, RESULTS_DIR)
            s3.upload_file(local_path, s3_bucket, f"{prefix}/{rel_path}")
```

The S3 client uses adaptive retry mode, which automatically adjusts retry behavior based on the type of error (throttling vs. connection failure). The `s3_endpoint` is configurable to support S3-compatible stores (MinIO, Nebius Object Storage) in addition to AWS S3. If credentials are not available (`AWS_ACCESS_KEY_ID` not set), the upload is silently skipped. Results are still on local disk and accessible via the shared filesystem.

## Configuration

| Key | Source | Default |
|-----|--------|---------|
| `BURNIN_PROVIDER` | Env / ConfigMap | `vllm` |
| `BURNIN_MODEL` | Env / ConfigMap | (none -- must be set) |
| `BURNIN_EVAL_DATASET` | Env / ConfigMap | (none -- falls back to hardcoded prompts) |
| `s3_bucket` | ConfigMap | (none -- upload skipped if unset) |
| `s3_endpoint` | ConfigMap | (none -- defaults to AWS) |

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- where burn-in fits in the system
- [Tracing](tracing.md) -- burn-in wraps server startup, eval loading, and the eval run in OTel spans
- [Observability](../architecture/observability.md) -- burn-in spans flow through the OTel to Tempo to Grafana pipeline
- [Decision Logic](../architecture/decision-logic.md) -- burn-in has its own PASS/DEGRADED/FAIL verdict logic
- [Multi-Provider Inference](../patterns/multi-provider-inference.md) -- the vLLM/SGLang/Ollama abstraction pattern
- [Execution Flow](../architecture/execution-flow.md) -- Stage 6, where burn-in runs
- [Distributed Validation](distributed-validate.md) -- the complementary cross-node validation that runs before burn-in
