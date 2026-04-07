# ConfigMap-Driven Thresholds

All preflight thresholds are defined in Terraform variables, rendered into a K8s ConfigMap, and consumed by test scripts at runtime. Thresholds change without rebuilding container images.

## Why ConfigMap

1. **Calibration workflow**: Run preflight once on new hardware, observe actual values, adjust thresholds via `terraform apply`
2. **Environment portability**: Different clusters (H100 vs H200, fabric-2 vs fabric-7) need different thresholds
3. **No image rebuild**: Threshold changes take effect immediately on next preflight run
4. **Single source of truth**: Terraform variables document the default, description, and current value

## Data Flow

```
variables.tf (defaults + descriptions)
    ↓
terraform.tfvars / TF_VAR_* (overrides)
    ↓
preflight-config.tf (renders into ConfigMap JSON)
    ↓
K8s ConfigMap "redstone-config"
    ↓
Mounted at /etc/redstone/config.json in every pod
    ↓
Each script reads CONFIG = json.load(f)
```

## Terraform Definition

All thresholds are in [`variables.tf`](../infrastructure/terraform.md):

- Each threshold is a separate Terraform variable with `description` and `default`
- Defaults are calibrated against probe data (2026-04-02, fabric-7, H200, eu-north1)
- The [preflight-config.tf](../infrastructure/terraform.md) renders them into the ConfigMap JSON

## Script Consumption

Every script loads the ConfigMap JSON at startup:

```python
CONFIG_PATH = os.environ.get("REDSTONE_CONFIG", "/etc/redstone/config.json")
with open(CONFIG_PATH) as f:
    CONFIG = json.load(f)
```

Thresholds are accessed via `CONFIG.get("key", default)`, always with a hardcoded fallback.

## Calibration

When deploying to new hardware:

1. Run preflight with permissive thresholds (`matmul_tflops_threshold = 0`)
2. Examine results: `make preflight-results`
3. Set thresholds to ~90% of observed values
4. `terraform apply` to update ConfigMap
5. Re-run preflight to validate

The [preflight-config.tf header](../infrastructure/terraform.md) documents the probe data thresholds were calibrated against.

## Cross-References

- [Architecture Overview](../architecture/overview.md) — how ConfigMap-driven thresholds fit into the overall system design
- [Decision Logic](../architecture/decision-logic.md) — thresholds drive the READY/DEGRADED/NOT_READY verdict computation
- [Observability](../architecture/observability.md) — threshold values and pass/fail results are captured as OTel span attributes in traces
- [Thresholds](../reference/thresholds.md) — complete table of all thresholds with defaults and code locations
- [Environment](../reference/environment.md) — all ConfigMap keys
- [Terraform](../infrastructure/terraform.md) — where thresholds are defined
