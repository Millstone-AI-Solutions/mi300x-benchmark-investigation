# SGLang - Calculation Breakdown

**Engine**: SGLang v0.5.9 | **Date**: 2026-03-25

Formulas and methodology in [`methodology.md`](../../methodology.md). Raw Prometheus snapshots in the accompanying `.txt` files (`{scenario}_{start|end}.txt`).

### SGLang decode speed derivation

SGLang does not expose `request_decode_time_seconds` like vLLM. Decode time is derived from e2e latency minus TTFT:

```
avg_e2e    = delta(e2e_sum) / delta(e2e_count)
avg_ttft   = delta(ttft_sum) / delta(ttft_count)
avg_decode = avg_e2e - avg_ttft
decode_speed = delta(generation_tokens_total) / (avg_decode * delta(e2e_count))
```

---

## Scenario 1: 1 User, 1K Context

**Config**: 1 concurrent user, 1024 input tokens, 1024 max output tokens, 10 requests
**Files**: `quick_1u_1k_start.txt`, `quick_1u_1k_end.txt`

### Decode Speed

```
From quick_1u_1k_start.txt:
  sglang:generation_tokens_total           = 58.0
  sglang:e2e_request_latency_seconds_sum   = 137.72930788993835
  sglang:e2e_request_latency_seconds_count = 2.0
  sglang:time_to_first_token_seconds_sum   = 137.72935461997986
  sglang:time_to_first_token_seconds_count = 2.0

From quick_1u_1k_end.txt:
  sglang:generation_tokens_total           = 7226.0
  sglang:e2e_request_latency_seconds_sum   = 312.62351536750793
  sglang:e2e_request_latency_seconds_count = 9.0
  sglang:time_to_first_token_seconds_sum   = 150.54411149024963
  sglang:time_to_first_token_seconds_count = 10.0

Deltas:
  generation_tokens = 7226.0 - 58.0       = 7168.0 tokens
  e2e_sum           = 312.6235 - 137.7293 = 174.8942 seconds
  e2e_count         = 9 - 2               = 7 completed requests
  ttft_sum          = 150.5441 - 137.7294 = 12.8148 seconds
  ttft_count        = 10 - 2              = 8 requests (got first token)

Derivation:
  avg_e2e             = 174.8942 / 7 = 24.9849 seconds
  avg_ttft            = 12.8148 / 8  = 1.6018 seconds
  avg_decode          = 24.9849 - 1.6018 = 23.3830 seconds
  derived_decode_time = 23.3830 * 7 = 163.6813 seconds
  decode_speed        = 7168.0 / 163.6813 = 43.79 tokens/sec
```

Note: 8 requests got their first token but only 7 completed end-to-end during the
test window.

### TTFT

```
Average TTFT:
  delta_sum   = 150.5441 - 137.7294 = 12.8148 seconds
  delta_count = 10 - 2              = 8 requests
  ttft_avg    = 12.8148 / 8         = 1.602 seconds
```

**Start** (quick_1u_1k_start.txt):

| Bucket (le) | Cumulative Count |
|-------------|-----------------|
| 0.1 | 0 |
| 0.2 | 0 |
| 0.4 | 0 |
| 0.6 | 0 |
| 0.8 | 0 |
| 1.0 | 0 |
| 2.0 | 0 |
| 4.0 | 1 |
| 6.0 | 1 |
| 8.0 | 1 |
| 10.0 | 1 |
| 20.0 | 1 |
| 40.0 | 1 |
| 60.0 | 1 |
| 80.0 | 1 |
| 100.0 | 1 |
| 200.0 | 2 |
| 400.0 | 2 |
| +Inf | 2 |

**End** (quick_1u_1k_end.txt):

| Bucket (le) | Cumulative Count |
|-------------|-----------------|
| 0.1 | 0 |
| 0.2 | 0 |
| 0.4 | 0 |
| 0.6 | 0 |
| 0.8 | 0 |
| 1.0 | 0 |
| 2.0 | 7 |
| 4.0 | 9 |
| 6.0 | 9 |
| 8.0 | 9 |
| 10.0 | 9 |
| 20.0 | 9 |
| 40.0 | 9 |
| 60.0 | 9 |
| 80.0 | 9 |
| 100.0 | 9 |
| 200.0 | 10 |
| 400.0 | 10 |
| +Inf | 10 |

**TTFT Histogram Bucket Deltas** (end - start):

| Bucket (le) | Start | End | Delta |
|-------------|-------|-----|-------|
| 0.1 | 0 | 0 | 0 |
| 0.2 | 0 | 0 | 0 |
| 0.4 | 0 | 0 | 0 |
| 0.6 | 0 | 0 | 0 |
| 0.8 | 0 | 0 | 0 |
| 1.0 | 0 | 0 | 0 |
| **2.0** | **0** | **7** | **7** |
| **4.0** | **1** | **9** | **8** |
| 6.0 | 1 | 9 | 8 |
| 8.0 | 1 | 9 | 8 |
| 10.0 | 1 | 9 | 8 |
| 20.0 | 1 | 9 | 8 |
| 40.0 | 1 | 9 | 8 |
| 60.0 | 1 | 9 | 8 |
| 80.0 | 1 | 9 | 8 |
| 100.0 | 1 | 9 | 8 |
| 200.0 | 2 | 10 | 8 |
| 400.0 | 2 | 10 | 8 |

7 requests had TTFT <= 2.0s, 1 more between 2.0s and 4.0s.

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 8 (from highest finite bucket le=400.0)
target_count = 0.50 * 8 = 4.0

Walk buckets (skipping zeros):
  le=2.0, cumulative=7 >= 4.0 --> interpolate here
    prev_upper = 1.0, prev_count = 0
    bucket_width = 2.0 - 1.0 = 1.0
    bucket_count = 7 - 0 = 7
    fraction = (4.0 - 0) / 7 = 0.5714
    result = 1.0 + (0.5714 * 1.0) = 1.571

TTFT p50 = 1.571 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 8 = 7.6

Walk buckets:
  le=2.0, cumulative=7 < 7.6 --> continue
  le=4.0, cumulative=8 >= 7.6 --> interpolate here
    prev_upper = 2.0, prev_count = 7
    bucket_width = 4.0 - 2.0 = 2.0
    bucket_count = 8 - 7 = 1
    fraction = (7.6 - 7) / 1 = 0.6
    result = 2.0 + (0.6 * 2.0) = 3.2

TTFT p95 = 3.2 seconds
```

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 43.79 tok/s |
| TTFT avg | 1.602s |
| TTFT p50 | 1.571s |
| TTFT p95 | 3.2s |
| Completed Requests | 7 of 10 (e2e), 8 of 10 (TTFT) |

---

## Scenario 2: 2 Users, 1K Context

**Config**: 2 concurrent users, 1024 input tokens, 1024 max output tokens, 10 requests
**Files**: `quick_2u_1k_start.txt`, `quick_2u_1k_end.txt`

### Decode Speed

```
From quick_2u_1k_start.txt:
  sglang:generation_tokens_total           = 8250.0
  sglang:e2e_request_latency_seconds_sum   = 337.2652313709259
  sglang:e2e_request_latency_seconds_count = 10.0
  sglang:time_to_first_token_seconds_sum   = 150.54411149024963
  sglang:time_to_first_token_seconds_count = 10.0

From quick_2u_1k_end.txt:
  sglang:generation_tokens_total           = 22586.0
  sglang:e2e_request_latency_seconds_sum   = 723.393349647522
  sglang:e2e_request_latency_seconds_count = 24.0
  sglang:time_to_first_token_seconds_sum   = 177.6830997467041
  sglang:time_to_first_token_seconds_count = 26.0

Deltas:
  generation_tokens = 22586.0 - 8250.0    = 14336.0 tokens
  e2e_sum           = 723.3933 - 337.2652 = 386.1281 seconds
  e2e_count         = 24 - 10             = 14 completed requests
  ttft_sum          = 177.6831 - 150.5441 = 27.1390 seconds
  ttft_count        = 26 - 10             = 16 requests (got first token)

Derivation:
  avg_e2e             = 386.1281 / 14 = 27.5806 seconds
  avg_ttft            = 27.1390 / 16  = 1.6962 seconds
  avg_decode          = 27.5806 - 1.6962 = 25.8844 seconds
  derived_decode_time = 25.8844 * 14 = 362.3815 seconds
  decode_speed        = 14336.0 / 362.3815 = 39.56 tokens/sec
```

### TTFT

```
Average TTFT:
  delta_sum   = 177.6831 - 150.5441 = 27.1390 seconds
  delta_count = 26 - 10             = 16 requests
  ttft_avg    = 27.1390 / 16        = 1.696 seconds
```

**Start** (quick_2u_1k_start.txt = quick_1u_1k end):

| Bucket (le) | Cumulative Count |
|-------------|-----------------|
| 0.1 | 0 |
| 0.2 | 0 |
| 0.4 | 0 |
| 0.6 | 0 |
| 0.8 | 0 |
| 1.0 | 0 |
| 2.0 | 7 |
| 4.0 | 9 |
| 6.0 | 9 |
| 8.0 | 9 |
| 10.0 | 9 |
| 20.0 | 9 |
| 40.0 | 9 |
| 60.0 | 9 |
| 80.0 | 9 |
| 100.0 | 9 |
| 200.0 | 10 |
| 400.0 | 10 |
| +Inf | 10 |

**End** (quick_2u_1k_end.txt):

| Bucket (le) | Cumulative Count |
|-------------|-----------------|
| 0.1 | 0 |
| 0.2 | 0 |
| 0.4 | 0 |
| 0.6 | 0 |
| 0.8 | 0 |
| 1.0 | 0 |
| 2.0 | 21 |
| 4.0 | 25 |
| 6.0 | 25 |
| 8.0 | 25 |
| 10.0 | 25 |
| 20.0 | 25 |
| 40.0 | 25 |
| 60.0 | 25 |
| 80.0 | 25 |
| 100.0 | 25 |
| 200.0 | 26 |
| 400.0 | 26 |
| +Inf | 26 |

**TTFT Histogram Bucket Deltas** (end - start):

| Bucket (le) | Start | End | Delta |
|-------------|-------|-----|-------|
| 0.1 | 0 | 0 | 0 |
| 0.2 | 0 | 0 | 0 |
| 0.4 | 0 | 0 | 0 |
| 0.6 | 0 | 0 | 0 |
| 0.8 | 0 | 0 | 0 |
| 1.0 | 0 | 0 | 0 |
| **2.0** | **7** | **21** | **14** |
| **4.0** | **9** | **25** | **16** |
| 6.0 | 9 | 25 | 16 |
| 8.0 | 9 | 25 | 16 |
| 10.0 | 9 | 25 | 16 |
| 20.0 | 9 | 25 | 16 |
| 40.0 | 9 | 25 | 16 |
| 60.0 | 9 | 25 | 16 |
| 80.0 | 9 | 25 | 16 |
| 100.0 | 9 | 25 | 16 |
| 200.0 | 10 | 26 | 16 |
| 400.0 | 10 | 26 | 16 |

14 requests had TTFT <= 2.0s, 2 more between 2.0s and 4.0s.

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 16 (from highest finite bucket le=400.0)
target_count = 0.50 * 16 = 8.0

Walk buckets (skipping zeros):
  le=2.0, cumulative=14 >= 8.0 --> interpolate here
    prev_upper = 1.0, prev_count = 0
    bucket_width = 2.0 - 1.0 = 1.0
    bucket_count = 14 - 0 = 14
    fraction = (8.0 - 0) / 14 = 0.5714
    result = 1.0 + (0.5714 * 1.0) = 1.571

TTFT p50 = 1.571 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 16 = 15.2

Walk buckets:
  le=2.0, cumulative=14 < 15.2 --> continue
  le=4.0, cumulative=16 >= 15.2 --> interpolate here
    prev_upper = 2.0, prev_count = 14
    bucket_width = 4.0 - 2.0 = 2.0
    bucket_count = 16 - 14 = 2
    fraction = (15.2 - 14) / 2 = 0.6
    result = 2.0 + (0.6 * 2.0) = 3.2

TTFT p95 = 3.2 seconds
```

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 39.56 tok/s |
| TTFT avg | 1.696s |
| TTFT p50 | 1.571s |
| TTFT p95 | 3.2s |
| Completed Requests | 14 of 10 (e2e), 16 of 10 (TTFT) |

Note: More than 10 requests were recorded because cumulative counters include all
requests during the test window. With 2 concurrent users sending 10 requests total,
some overlap with the prior scenario's in-flight requests may contribute.

---

## Scenario 3: 1 User, 32K Context

**Config**: 1 concurrent user, 32768 input tokens, 1024 max output tokens, 10 requests
**Files**: `quick_1u_32k_start.txt`, `quick_1u_32k_end.txt`

### Decode Speed

```
From quick_1u_32k_start.txt:
  sglang:generation_tokens_total           = 24634.0
  sglang:e2e_request_latency_seconds_sum   = 778.2061173915863
  sglang:e2e_request_latency_seconds_count = 26.0
  sglang:time_to_first_token_seconds_sum   = 177.6830997467041
  sglang:time_to_first_token_seconds_count = 26.0

From quick_1u_32k_end.txt:
  sglang:generation_tokens_total           = 28730.0
  sglang:e2e_request_latency_seconds_sum   = 900.2372579574585
  sglang:e2e_request_latency_seconds_count = 30.0
  sglang:time_to_first_token_seconds_sum   = 201.66909551620483
  sglang:time_to_first_token_seconds_count = 31.0

Deltas:
  generation_tokens = 28730.0 - 24634.0   = 4096.0 tokens
  e2e_sum           = 900.2373 - 778.2061 = 122.0311 seconds
  e2e_count         = 30 - 26             = 4 completed requests
  ttft_sum          = 201.6691 - 177.6831 = 23.9860 seconds
  ttft_count        = 31 - 26             = 5 requests (got first token)

Derivation:
  avg_e2e             = 122.0311 / 4 = 30.5078 seconds
  avg_ttft            = 23.9860 / 5  = 4.7972 seconds
  avg_decode          = 30.5078 - 4.7972 = 25.7106 seconds
  derived_decode_time = 25.7106 * 4 = 102.8423 seconds
  decode_speed        = 4096.0 / 102.8423 = 39.83 tokens/sec
```

Note: Only 4 of 10 requests completed end-to-end, and 5 got their first token.

### TTFT

```
Average TTFT:
  delta_sum   = 201.6691 - 177.6831 = 23.9860 seconds
  delta_count = 31 - 26             = 5 requests
  ttft_avg    = 23.9860 / 5         = 4.797 seconds
```

**TTFT Histogram Bucket Deltas** (end - start):

| Bucket (le) | Start | End | Delta |
|-------------|-------|-----|-------|
| 0.1 | 0 | 0 | 0 |
| 0.2 | 0 | 0 | 0 |
| 0.4 | 0 | 0 | 0 |
| 0.6 | 0 | 0 | 0 |
| 0.8 | 0 | 0 | 0 |
| 1.0 | 0 | 0 | 0 |
| 2.0 | 21 | 21 | 0 |
| 4.0 | 25 | 25 | 0 |
| **6.0** | **25** | **29** | **4** |
| **8.0** | **25** | **30** | **5** |
| 10.0 | 25 | 30 | 5 |
| 20.0 | 25 | 30 | 5 |
| 40.0 | 25 | 30 | 5 |
| 60.0 | 25 | 30 | 5 |
| 80.0 | 25 | 30 | 5 |
| 100.0 | 25 | 30 | 5 |
| 200.0 | 26 | 31 | 5 |
| 400.0 | 26 | 31 | 5 |

4 requests had TTFT between 4.0s and 6.0s, 1 more between 6.0s and 8.0s.

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 5 (from highest finite bucket le=400.0)
target_count = 0.50 * 5 = 2.5

Walk buckets (skipping zeros):
  le=6.0, cumulative=4 >= 2.5 --> interpolate here
    prev_upper = 4.0, prev_count = 0
    bucket_width = 6.0 - 4.0 = 2.0
    bucket_count = 4 - 0 = 4
    fraction = (2.5 - 0) / 4 = 0.625
    result = 4.0 + (0.625 * 2.0) = 5.25

TTFT p50 = 5.25 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 5 = 4.75

Walk buckets:
  le=6.0, cumulative=4 < 4.75 --> continue
  le=8.0, cumulative=5 >= 4.75 --> interpolate here
    prev_upper = 6.0, prev_count = 4
    bucket_width = 8.0 - 6.0 = 2.0
    bucket_count = 5 - 4 = 1
    fraction = (4.75 - 4) / 1 = 0.75
    result = 6.0 + (0.75 * 2.0) = 7.5

TTFT p95 = 7.5 seconds
```

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 39.83 tok/s |
| TTFT avg | 4.797s |
| TTFT p50 | 5.25s |
| TTFT p95 | 7.5s |
| Completed Requests | 4 of 10 (e2e), 5 of 10 (TTFT) |

---

## Scenario 4: 2 Users, 32K Context

**Config**: 2 concurrent users, 32768 input tokens, 1024 max output tokens, 10 requests
**Files**: `quick_2u_32k_start.txt`, `quick_2u_32k_end.txt`

### Decode Speed

```
From quick_2u_32k_start.txt:
  sglang:generation_tokens_total           = 29754.0
  sglang:e2e_request_latency_seconds_sum   = 930.2652142047882
  sglang:e2e_request_latency_seconds_count = 31.0
  sglang:time_to_first_token_seconds_sum   = 201.66909551620483
  sglang:time_to_first_token_seconds_count = 31.0

From quick_2u_32k_end.txt:
  sglang:generation_tokens_total           = 42042.0
  sglang:e2e_request_latency_seconds_sum   = 1354.400405883789
  sglang:e2e_request_latency_seconds_count = 43.0
  sglang:time_to_first_token_seconds_sum   = 304.3774116039276
  sglang:time_to_first_token_seconds_count = 45.0

Deltas:
  generation_tokens = 42042.0 - 29754.0   = 12288.0 tokens
  e2e_sum           = 1354.4004 - 930.2652 = 424.1352 seconds
  e2e_count         = 43 - 31              = 12 completed requests
  ttft_sum          = 304.3774 - 201.6691  = 102.7083 seconds
  ttft_count        = 45 - 31              = 14 requests (got first token)

Derivation:
  avg_e2e             = 424.1352 / 12 = 35.3446 seconds
  avg_ttft            = 102.7083 / 14 = 7.3363 seconds
  avg_decode          = 35.3446 - 7.3363 = 28.0083 seconds
  derived_decode_time = 28.0083 * 12 = 336.1000 seconds
  decode_speed        = 12288.0 / 336.1000 = 36.56 tokens/sec
```

### TTFT

```
Average TTFT:
  delta_sum   = 304.3774 - 201.6691 = 102.7083 seconds
  delta_count = 45 - 31             = 14 requests
  ttft_avg    = 102.7083 / 14       = 7.336 seconds
```

**TTFT Histogram Bucket Deltas** (end - start):

| Bucket (le) | Start | End | Delta |
|-------------|-------|-----|-------|
| 0.1 | 0 | 0 | 0 |
| 0.2 | 0 | 0 | 0 |
| 0.4 | 0 | 0 | 0 |
| 0.6 | 0 | 0 | 0 |
| 0.8 | 0 | 0 | 0 |
| 1.0 | 0 | 0 | 0 |
| 2.0 | 21 | 21 | 0 |
| 4.0 | 25 | 25 | 0 |
| 6.0 | 29 | 29 | 0 |
| **8.0** | **30** | **44** | **14** |
| 10.0 | 30 | 44 | 14 |
| 20.0 | 30 | 44 | 14 |
| 40.0 | 30 | 44 | 14 |
| 60.0 | 30 | 44 | 14 |
| 80.0 | 30 | 44 | 14 |
| 100.0 | 30 | 44 | 14 |
| 200.0 | 31 | 45 | 14 |
| 400.0 | 31 | 45 | 14 |

All 14 requests had TTFT between 6.0s and 8.0s.

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 14 (from highest finite bucket le=400.0)
target_count = 0.50 * 14 = 7.0

Walk buckets (skipping zeros):
  le=8.0, cumulative=14 >= 7.0 --> interpolate here
    prev_upper = 6.0, prev_count = 0
    bucket_width = 8.0 - 6.0 = 2.0
    bucket_count = 14 - 0 = 14
    fraction = (7.0 - 0) / 14 = 0.5
    result = 6.0 + (0.5 * 2.0) = 7.0

TTFT p50 = 7.0 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 14 = 13.3

Walk buckets:
  le=8.0, cumulative=14 >= 13.3 --> interpolate here
    prev_upper = 6.0, prev_count = 0
    bucket_width = 8.0 - 6.0 = 2.0
    bucket_count = 14 - 0 = 14
    fraction = (13.3 - 0) / 14 = 0.95
    result = 6.0 + (0.95 * 2.0) = 7.9

TTFT p95 = 7.9 seconds
```

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 36.56 tok/s |
| TTFT avg | 7.336s |
| TTFT p50 | 7.0s |
| TTFT p95 | 7.9s |
| Completed Requests | 12 of 10 (e2e), 14 of 10 (TTFT) |

---
