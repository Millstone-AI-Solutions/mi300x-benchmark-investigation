# Test Methodology

Full public methodology: [millstoneai.com/inference-benchmark-methodology](https://www.millstoneai.com/inference-benchmark-methodology)

---

## Test Parameters

| Parameter | Value |
|-----------|-------|
| Model | Qwen/Qwen3.5-122B-A10B-FP8 |
| Input tokens | 1,024 (1K) or 32,768 (32K), calibrated via model tokenizer |
| Output tokens | 1,024, EOS disabled |
| Prompt caching | Disabled & unique prefix per request |
| Concurrency | 1 and 2 concurrent users (128K tests: 1 and 5) |
| Load generator | Locust-based, serial request pattern, ~5s gradual ramp-up |
| Stability | Runs until requests/sec is within 10% for 3 consecutive readings |

## Metrics Collection

All metrics from the engine's `/metrics` endpoint (Prometheus format), not client-side.

For each test scenario:
1. Capture `/metrics` snapshot before the test (baseline)
2. Run the test
3. Capture `/metrics` snapshot after the test
4. Compute deltas (end - start) to isolate that test run

### Metrics Used

**vLLM:**
| Metric | Type | Purpose |
|--------|------|---------|
| `vllm:time_to_first_token_seconds` | Histogram | TTFT distribution |
| `vllm:request_decode_time_seconds` | Histogram | Total decode time per request |
| `vllm:generation_tokens_total` | Counter | Total output tokens generated |

**SGLang:**
| Metric | Type | Purpose |
|--------|------|---------|
| `sglang:time_to_first_token_seconds` | Histogram | TTFT distribution |
| `sglang:generation_tokens_total` | Counter | Total output tokens generated |
| Decode speed derived from token counts and timing histograms |

### Formulas

**Decode Speed (avg tok/sec per user):**
```
decode_speed = delta(generation_tokens_total) / delta(request_decode_time_seconds_sum)
```

**TTFT Average:**
```
ttft_avg = delta(time_to_first_token_seconds_sum) / delta(time_to_first_token_seconds_count)
```

**TTFT Percentiles:** Standard Prometheus `histogram_quantile()` algorithm on bucket deltas.

Full calculation breakdowns with raw counter values in `raw-metrics/*/calculations.md`.

---

## Quick Validation

Our per-user numbers are consistent with the engine's periodic server logs. For example, vLLM v2 reports ~77 tok/s aggregate generation throughput during a single-user run (see [`logs/vllm-v2-requests.log`](logs/vllm-v2-requests.log)). The main difference is our metrics are per-user while server logs report aggregate.

## Full Replication

1. Qwen/Qwen3.5-122B-A10B-FP8
2. Token counts via model tokenizer, EOS disabled
3. Prompt caching disabled
4. Collect `/metrics` before and after each test, compute deltas
5. Use docker-compose files in `configs/` to match server configurations
