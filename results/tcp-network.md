# TCP Network Results

Source: [tcp-test.json](../results-data/tcp-test.json) | Run: 20260403-200023 | 4 parallel streams, 20s duration

## Bandwidth

| Metric | Measured | Threshold | Verdict |
|--------|----------|-----------|---------|
| Bandwidth | 11.39 Gb/s | 10 Gb/s | PASS |
| Retransmits | 57,199 | (informational) | High |

TCP bandwidth passed. The high retransmit count (57K in 20s) indicates buffer pressure. Not blocking for control plane traffic, but TCP buffer sizes and MTU settings should be checked for control plane traffic.

**Verdict: PASS**

[Back to Results](../results.md)
