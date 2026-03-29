# Results Summary

All results across all configurations and the NVIDIA baseline.

## Decode Speed (tokens/sec per user)

| Scenario | MI300X vLLM v1 | MI300X vLLM v2 | MI300X SGLang | 2x RTX Pro 6000 |
|----------|---------------|----------------|---------------|---------------------|
| 1K ctx / 1 req | 86.0 | 82.0 | 44.0 | **106.0** |
| 1K ctx / 2 req | **79.0** | 73.0 | 40.0 | 76.8 |
| 32K ctx / 1 req | 83.0 | 86.0 | 40.0 | **100.5** |
| 32K ctx / 2 req | **77.0** | 75.0 | 37.0 | 72.6 |
| 128K ctx / 1 req | 76.1 | 74.6 | 49.2 | **89.4** |
| 128K ctx / 5 req | 17.6 | 17.9 | **26.5** | 21.1 |

## TTFT (seconds)

| Scenario | MI300X vLLM v1 | MI300X vLLM v2 | MI300X SGLang | 2x RTX Pro 6000 |
|----------|---------------|----------------|---------------|---------------------|
| 1K ctx / 1 req | 0.1 | 0.1 | 1.6 | 0.1 |
| 1K ctx / 2 req | 0.2 | 0.2 | 1.7 | **0.1** |
| 32K ctx / 1 req | 33.5* | 31.6* | 4.8 | **2.1** |
| 32K ctx / 2 req | 5.3 | 6.2 | 7.3 | **3.2** |
| 128K ctx / 1 req | 17.3 | 16.5 | 25.3 | **12.2** |
| 128K ctx / 5 req | 48.8 | 47.7 | 113.3 | **33.6** |

\* Includes JIT compilation outlier

1K and 32K have full raw Prometheus snapshots and calculation breakdowns in `raw-metrics/`. 128K snapshots were not saved.
