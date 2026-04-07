# Tracing

Every script in the Redstone preflight suite exports OpenTelemetry spans to the same place, configured the same way. Rather than duplicating that boilerplate seven times, `tracing.py` provides two functions and 52 lines of shared setup.

## Why a Shared Module

Each preflight script (GPU health, NCCL, burn-in, etc.) runs as its own Kubernetes Job on its own schedule. Without a shared module, each script would independently configure the OTel SDK, pick an exporter endpoint, set resource attributes, and remember to flush on exit. That is seven chances to get it wrong. By centralizing into one importable module, we guarantee:

- Every span carries the same resource attributes (`service.name`, `k8s.node.name`, `run_id`).
- Every script exports to the same OTLP endpoint.
- The flush-on-exit footgun is documented and exposed as a single function call.

## init_tracer

This function builds the full OTel pipeline (Resource, TracerProvider, Exporter, Processor) and returns a ready-to-use `Tracer`.

```python
import os
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource


def init_tracer(service_name: str) -> trace.Tracer:
    resource = Resource.create({
        "service.name": service_name,
        "k8s.node.name": os.environ.get("NODE_NAME", "unknown"),
        "redstone.run_id": os.environ.get("RUN_ID", "unknown"),
    })

    provider = TracerProvider(resource=resource)

    exporter = OTLPSpanExporter(
        endpoint=os.environ.get(
            "OTEL_EXPORTER_OTLP_ENDPOINT",
            "http://nebius-observability-agent.o11y.svc.cluster.local:4317"
        ),
        insecure=True,
    )

    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    return trace.get_tracer(service_name)
```

### Resource creation

The `Resource` is the identity card for every span. Three attributes are attached:

- **`service.name`** -- the caller passes this in (e.g. `"redstone.gpu_health"`). In Grafana/Tempo, this is how you filter traces to a specific test phase.
- **`k8s.node.name`** -- pulled from the `NODE_NAME` environment variable, which the K8s Job manifest injects via the downward API. Correlates a slow span to a specific bare-metal node.
- **`redstone.run_id`** -- the unique run identifier. Every preflight run gets a UUID, so you can pull all spans from a single validation cycle.

### The OTLP exporter

The default endpoint targets the Nebius observability agent's gRPC port at `nebius-observability-agent.o11y.svc.cluster.local:4317`. A DaemonSet-deployed collector that forwards spans to managed Tempo. The `insecure=True` flag is intentional. The agent runs cluster-internal on a private network, and mTLS is handled at the mesh layer, not the application layer.

The endpoint can be overridden via the standard `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable. Useful for local development (point at a Jaeger instance) or when deploying to clusters with a different observability stack.

### BatchSpanProcessor

The alternative to `BatchSpanProcessor`, `SimpleSpanProcessor`, exports each span synchronously as it completes. That would add network round-trip latency to every `span.end()` call, which is unacceptable in benchmarks where we are timing GPU operations down to the millisecond. The batch processor buffers spans in memory and flushes them in the background on a timer (default: every 5 seconds) or when the buffer fills. The tradeoff is that buffered spans are lost if the process crashes or exits without flushing -- which is why `shutdown_tracer` exists.

### Global provider

The call to `trace.set_tracer_provider(provider)` installs this as the process-wide default. Any library code that calls `trace.get_tracer()` without an explicit provider will inherit this configuration. This matters because some libraries (like `requests` or `boto3` via instrumentation packages) can auto-generate spans, and we want those to flow to the same destination.

## shutdown_tracer

```python
def shutdown_tracer():
    provider = trace.get_tracer_provider()
    if hasattr(provider, "shutdown"):
        provider.shutdown()
```

This function exists because of the `BatchSpanProcessor`. Since spans are buffered in memory, exiting without calling `shutdown()` means the last batch of spans is silently dropped. The final verdict span is typically the one you need to debug a run, so losing it defeats the purpose of tracing.

The `hasattr` guard handles the edge case where the global provider is still the no-op default (e.g., if `init_tracer` was never called or failed). The no-op `TracerProvider` from the API package does not have a `shutdown` method, so calling it directly would raise an `AttributeError`.

Every script follows the same pattern: initialize at module load, shut down at the end of `main()`.

## Usage Pattern

```python
from tracing import init_tracer, shutdown_tracer

tracer = init_tracer("redstone.my_test")

with tracer.start_as_current_span("my_operation") as span:
    span.set_attribute("result", "pass")
    # ... test logic ...

shutdown_tracer()
```

The `start_as_current_span` context manager ensures the span is ended (and queued for export) even if the test logic raises an exception. Attributes set on the span become searchable fields in Tempo/Grafana.

## Service Name Table

Every script that imports this module and the `service.name` it registers:

| Script | Service Name |
|--------|-------------|
| `gpu_health.py` | `redstone.gpu_health` |
| `nccl_test.py` | `redstone.nccl_{TEST_MODE}` (dynamic, e.g. `redstone.nccl_multinode`) |
| `ib_topology.py` | `redstone.ib_topology` |
| `tcp_test.py` | `redstone.tcp_network` |
| `storage_test.py` | `redstone.storage` |
| `training_burnin.py` | `redstone.burnin` |
| `orchestrator.py` | `redstone.orchestrator` |

The `nccl_test.py` entry is dynamic because it interpolates the test mode (e.g. `multinode`, `intranode`) into the service name, allowing you to filter NCCL traces by scope in Grafana.

## Cross-References

- [Architecture Overview](../architecture/overview.md) -- where the tracing module fits in the system
- [Execution Flow](../architecture/execution-flow.md) -- tracing spans the entire preflight flow from orchestrator root span through all phases
- [Observability](../architecture/observability.md) -- the full OTel to Tempo to Grafana pipeline
- [Training Burn-in](training-burnin.md) -- most detailed tracing usage (spans around server startup, eval, and overall burn-in)
- [GPU Health](gpu-health.md) -- per-GPU span attributes for thermal and memory diagnostics
