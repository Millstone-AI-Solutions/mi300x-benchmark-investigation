# vLLM v1 - Calculation Breakdown

**Engine**: vLLM v0.18.0 | **Date**: 2026-03-25

Formulas and methodology in [`methodology.md`](../../methodology.md). Raw Prometheus snapshots in the accompanying `.txt` files (`{scenario}_{start|end}.txt`).

---

## Metrics Used

| Metric | Type | Purpose |
|--------|------|---------|
| `vllm:time_to_first_token_seconds` | Histogram | TTFT distribution |
| `vllm:request_decode_time_seconds` | Histogram | Total decode time per request |
| `vllm:generation_tokens_total` | Counter | Total output tokens generated |

## Formulas

### Decode Speed (tokens/sec per user)

```
decode_speed = delta(generation_tokens_total) / delta(request_decode_time_seconds_sum)
```

- `generation_tokens_total`: cumulative counter of all output tokens
- `request_decode_time_seconds_sum`: sum of all individual request decode times

This gives the **per-user** decode throughput because the denominator is the sum of
individual request decode times (not wall-clock time). When multiple users are running
concurrently, each request's decode time is counted separately.

### TTFT Average

```
ttft_avg = delta(time_to_first_token_seconds_sum) / delta(time_to_first_token_seconds_count)
```

### TTFT Percentiles (p50, p75, p90, p95, p99)

Percentiles are calculated from histogram bucket deltas using the same algorithm as
Prometheus `histogram_quantile()`. The algorithm:

1. Compute delta buckets: `delta_bucket[le] = end_bucket[le] - start_bucket[le]`
2. Exclude the `+Inf` bucket
3. Find the bucket containing the target count: `target = quantile * total_count`
4. Linear interpolation within that bucket:

```
bucket_width  = upper_bound - previous_upper_bound
bucket_count  = cumulative_count - previous_cumulative_count
fraction      = (target_count - previous_cumulative_count) / bucket_count
result        = previous_upper_bound + (fraction * bucket_width)
```

This assumes uniform distribution within each bucket.

---

## Scenario 1: 1 User, 1K Context

**Config**: 1 concurrent user, 1024 input tokens, 1024 max output tokens, 10 requests
**Files**: `quick_1u_1k_start.txt`, `quick_1u_1k_end.txt`

### Decode Speed

```
From quick_1u_1k_start.txt:
  vllm:generation_tokens_total  = 808.0
  vllm:request_decode_time_seconds_sum   = 0.5831977089997054
  vllm:request_decode_time_seconds_count = 1.0

From quick_1u_1k_end.txt:
  vllm:generation_tokens_total  = 10307.0
  vllm:request_decode_time_seconds_sum   = 111.07866466100086
  vllm:request_decode_time_seconds_count = 10.0

Deltas:
  generation_tokens  = 10307.0 - 808.0   = 9499.0 tokens
  decode_time_sum    = 111.079 - 0.583   = 110.495 seconds
  decode_time_count  = 10 - 1            = 9 completed requests

Calculation:
  decode_speed = 9499.0 / 110.495 = 85.97 tokens/sec
```

### TTFT

```
From quick_1u_1k_start.txt:
  vllm:time_to_first_token_seconds_sum   = 94.67043089866638
  vllm:time_to_first_token_seconds_count = 2.0

From quick_1u_1k_end.txt:
  vllm:time_to_first_token_seconds_sum   = 96.05383467674255
  vllm:time_to_first_token_seconds_count = 12.0

Average TTFT:
  delta_sum   = 96.054 - 94.670 = 1.3834 seconds
  delta_count = 12 - 2          = 10 requests
  ttft_avg    = 1.3834 / 10     = 0.138 seconds
```

**TTFT Histogram Bucket Deltas** (end - start, excluding +Inf):

| Bucket (le) | Start | End | Delta |
|-------------|-------|-----|-------|
| 0.001 | 0 | 0 | 0 |
| 0.005 | 0 | 0 | 0 |
| 0.01 | 0 | 0 | 0 |
| 0.02 | 0 | 0 | 0 |
| 0.04 | 0 | 0 | 0 |
| 0.06 | 0 | 0 | 0 |
| 0.08 | 1 | 1 | 0 |
| 0.1 | 1 | 1 | 0 |
| **0.25** | **1** | **11** | **10** |
| 0.5 | 1 | 11 | 10 |
| 0.75 | 1 | 11 | 10 |
| 1.0 | 1 | 11 | 10 |
| 2.5 | 1 | 11 | 10 |
| 5.0 | 1 | 11 | 10 |
| 7.5 | 1 | 11 | 10 |
| 10.0 | 1 | 11 | 10 |
| 20.0 | 1 | 11 | 10 |
| 40.0 | 1 | 11 | 10 |
| 80.0 | 1 | 11 | 10 |
| 160.0 | 2 | 12 | 10 |
| 640.0 | 2 | 12 | 10 |
| 2560.0 | 2 | 12 | 10 |

All 10 requests fell into the le=0.25 bucket (TTFT between 0.1s and 0.25s).

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 10 (from highest finite bucket)
target_count = 0.50 * 10 = 5.0

Walk buckets (skipping zeros):
  le=0.25, cumulative=10 >= 5.0 --> interpolate here
    prev_upper = 0.1, prev_count = 0
    bucket_width = 0.25 - 0.1 = 0.15
    bucket_count = 10 - 0 = 10
    fraction = (5.0 - 0) / 10 = 0.5
    result = 0.1 + (0.5 * 0.15) = 0.175

TTFT p50 = 0.175 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 10 = 9.5

Walk buckets:
  le=0.25, cumulative=10 >= 9.5 --> interpolate here
    fraction = (9.5 - 0) / 10 = 0.95
    result = 0.1 + (0.95 * 0.15) = 0.2425

TTFT p95 = 0.243 seconds
```

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 85.97 tok/s |
| TTFT avg | 0.138s |
| TTFT p50 | 0.175s |
| TTFT p95 | 0.243s |

---

## Scenario 2: 2 Users, 1K Context

**Config**: 2 concurrent users, 1024 input tokens, 1024 max output tokens, 10 requests
**Files**: `quick_2u_1k_start.txt`, `quick_2u_1k_end.txt`

### Decode Speed

```
From quick_2u_1k_start.txt:
  vllm:generation_tokens_total  = 10307.0
  vllm:request_decode_time_seconds_sum   = 111.07866466100086
  vllm:request_decode_time_seconds_count = 10.0

From quick_2u_1k_end.txt:
  vllm:generation_tokens_total  = 18741.0
  vllm:request_decode_time_seconds_sum   = 217.86847559199987
  vllm:request_decode_time_seconds_count = 18.0

Deltas:
  generation_tokens  = 18741.0 - 10307.0 = 8434.0 tokens
  decode_time_sum    = 217.868 - 111.079 = 106.790 seconds
  decode_time_count  = 18 - 10           = 8 completed requests

Calculation:
  decode_speed = 8434.0 / 106.790 = 78.98 tokens/sec
```

### TTFT

```
From quick_2u_1k_start.txt:
  vllm:time_to_first_token_seconds_sum   = 96.05383467674255
  vllm:time_to_first_token_seconds_count = 12.0

From quick_2u_1k_end.txt:
  vllm:time_to_first_token_seconds_sum   = 98.04432463645935
  vllm:time_to_first_token_seconds_count = 22.0

Average TTFT:
  delta_sum   = 98.044 - 96.054 = 1.990 seconds
  delta_count = 22 - 12         = 10 requests
  ttft_avg    = 1.990 / 10      = 0.199 seconds
```

**TTFT Histogram Bucket Deltas**:

| Bucket (le) | Start | End | Delta |
|-------------|-------|-----|-------|
| 0.001 | 0 | 0 | 0 |
| 0.005 | 0 | 0 | 0 |
| 0.01 | 0 | 0 | 0 |
| 0.02 | 0 | 0 | 0 |
| 0.04 | 0 | 0 | 0 |
| 0.06 | 0 | 0 | 0 |
| 0.08 | 1 | 1 | 0 |
| 0.1 | 1 | 1 | 0 |
| **0.25** | **11** | **18** | **7** |
| **0.5** | **11** | **21** | **10** |
| 0.75 | 11 | 21 | 10 |
| 1.0 | 11 | 21 | 10 |
| 2.5 | 11 | 21 | 10 |
| 5.0 | 11 | 21 | 10 |
| 7.5 | 11 | 21 | 10 |
| 10.0 | 11 | 21 | 10 |
| 20.0 | 11 | 21 | 10 |
| 40.0 | 11 | 21 | 10 |
| 80.0 | 11 | 21 | 10 |
| 160.0 | 12 | 22 | 10 |
| 640.0 | 12 | 22 | 10 |
| 2560.0 | 12 | 22 | 10 |

7 requests had TTFT <= 0.25s, 3 more had TTFT between 0.25s and 0.5s.

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 10
target_count = 0.50 * 10 = 5.0

Walk buckets:
  le=0.25, cumulative=7 >= 5.0 --> interpolate here
    prev_upper = 0.1, prev_count = 0
    bucket_width = 0.25 - 0.1 = 0.15
    bucket_count = 7 - 0 = 7
    fraction = (5.0 - 0) / 7 = 0.7143
    result = 0.1 + (0.7143 * 0.15) = 0.207

TTFT p50 = 0.207 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 10 = 9.5

Walk buckets:
  le=0.25, cumulative=7 < 9.5 --> continue
  le=0.5, cumulative=10 >= 9.5 --> interpolate here
    prev_upper = 0.25, prev_count = 7
    bucket_width = 0.5 - 0.25 = 0.25
    bucket_count = 10 - 7 = 3
    fraction = (9.5 - 7) / 3 = 0.8333
    result = 0.25 + (0.8333 * 0.25) = 0.458

TTFT p95 = 0.458 seconds
```

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 78.98 tok/s |
| TTFT avg | 0.199s |
| TTFT p50 | 0.207s |
| TTFT p95 | 0.458s |

---

## Scenario 3: 1 User, 32K Context

**Config**: 1 concurrent user, 32768 input tokens, 1024 max output tokens, 10 requests
**Files**: `quick_1u_32k_start.txt`, `quick_1u_32k_end.txt`

### Decode Speed

```
From quick_1u_32k_start.txt:
  vllm:generation_tokens_total  = 18741.0
  vllm:request_decode_time_seconds_sum   = 217.86847559199987
  vllm:request_decode_time_seconds_count = 18.0

From quick_1u_32k_end.txt:
  vllm:generation_tokens_total  = 24949.0
  vllm:request_decode_time_seconds_sum   = 292.882867627
  vllm:request_decode_time_seconds_count = 24.0

Deltas:
  generation_tokens  = 24949.0 - 18741.0 = 6208.0 tokens
  decode_time_sum    = 292.883 - 217.868 = 75.014 seconds
  decode_time_count  = 24 - 18           = 6 completed requests

Calculation:
  decode_speed = 6208.0 / 75.014 = 82.76 tokens/sec
```

Note: Only 6 of 10 requests completed during the test window. 4 requests likely
timed out or were still in-flight when the test ended.

### TTFT

```
From quick_1u_32k_start.txt:
  vllm:time_to_first_token_seconds_sum   = 98.04432463645935
  vllm:time_to_first_token_seconds_count = 22.0

From quick_1u_32k_end.txt:
  vllm:time_to_first_token_seconds_sum   = 332.71715331077576
  vllm:time_to_first_token_seconds_count = 29.0

Average TTFT:
  delta_sum   = 332.717 - 98.044 = 234.673 seconds
  delta_count = 29 - 22          = 7 requests
  ttft_avg    = 234.673 / 7      = 33.525 seconds
```

**Deltas** (non-zero only):

| Bucket (le) | Delta |
|-------------|-------|
| 5.0 | 6 |
| 7.5 | 6 |
| 10.0 | 6 |
| 20.0 | 6 |
| 40.0 | 6 |
| 80.0 | 6 |
| 160.0 | 6 |
| 640.0 | 7 |
| 2560.0 | 7 |

Interpretation: 6 requests had TTFT <= 5.0s. 1 additional request had TTFT between
160s and 640s. This is the massive outlier that dominates the average.

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 7 (from highest finite bucket le=2560)
target_count = 0.50 * 7 = 3.5

Walk buckets (skipping zeros):
  le=5.0, cumulative=6 >= 3.5 --> interpolate here
    prev_upper = 2.5, prev_count = 0
    bucket_width = 5.0 - 2.5 = 2.5
    bucket_count = 6 - 0 = 6
    fraction = (3.5 - 0) / 6 = 0.5833
    result = 2.5 + (0.5833 * 2.5) = 3.958

TTFT p50 = 3.958 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 7 = 6.65

Walk buckets:
  le=5.0, cumulative=6 < 6.65 --> continue
  le=7.5, cumulative=6 < 6.65 --> continue
  ...
  le=160.0, cumulative=6 < 6.65 --> continue
  le=640.0, cumulative=7 >= 6.65 --> interpolate here
    prev_upper = 160.0, prev_count = 6
    bucket_width = 640.0 - 160.0 = 480.0
    bucket_count = 7 - 6 = 1
    fraction = (6.65 - 6) / 1 = 0.65
    result = 160.0 + (0.65 * 480.0) = 472.0

TTFT p95 = 472.0 seconds
```

Note: The p95 is extremely high because of one outlier request with TTFT somewhere
between 160s and 640s. With only 7 requests total, a single outlier dominates the
upper percentiles. The linear interpolation within the 160-640 bucket places p95
at 472s, but the actual single outlier could be anywhere in that range.

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 82.76 tok/s |
| TTFT avg | 33.525s |
| TTFT p50 | 3.958s |
| TTFT p95 | 472.0s |
| Completed Requests | 6 of 10 (decode), 7 of 10 (TTFT) |

---

## Scenario 4: 2 Users, 32K Context

**Config**: 2 concurrent users, 32768 input tokens, 1024 max output tokens, 10 requests
**Files**: `quick_2u_32k_start.txt`, `quick_2u_32k_end.txt`

### Decode Speed

```
From quick_2u_32k_start.txt:
  vllm:generation_tokens_total  = 24949.0
  vllm:request_decode_time_seconds_sum   = 292.882867627
  vllm:request_decode_time_seconds_count = 24.0

From quick_2u_32k_end.txt:
  vllm:generation_tokens_total  = 31789.0
  vllm:request_decode_time_seconds_sum   = 382.138409387
  vllm:request_decode_time_seconds_count = 30.0

Deltas:
  generation_tokens  = 31789.0 - 24949.0 = 6840.0 tokens
  decode_time_sum    = 382.138 - 292.883 = 89.256 seconds
  decode_time_count  = 30 - 24           = 6 completed requests

Calculation:
  decode_speed = 6840.0 / 89.256 = 76.63 tokens/sec
```

### TTFT

```
From quick_2u_32k_start.txt:
  vllm:time_to_first_token_seconds_sum   = 332.71715331077576
  vllm:time_to_first_token_seconds_count = 29.0

From quick_2u_32k_end.txt:
  vllm:time_to_first_token_seconds_sum   = 374.87476563453674
  vllm:time_to_first_token_seconds_count = 37.0

Average TTFT:
  delta_sum   = 374.875 - 332.717 = 42.158 seconds
  delta_count = 37 - 29           = 8 requests
  ttft_avg    = 42.158 / 8        = 5.270 seconds
```

**Deltas** (non-zero only):

| Bucket (le) | Delta |
|-------------|-------|
| 5.0 | 3 |
| 7.5 | 7 |
| 10.0 | 8 |
| 20.0 | 8 |
| 40.0 | 8 |
| 80.0 | 8 |
| 160.0 | 8 |
| 640.0 | 8 |
| 2560.0 | 8 |

3 requests had TTFT <= 5.0s, 4 more between 5.0s and 7.5s, 1 more between 7.5s and 10.0s.

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 8
target_count = 0.50 * 8 = 4.0

Walk buckets:
  le=5.0, cumulative=3 < 4.0 --> continue
  le=7.5, cumulative=7 >= 4.0 --> interpolate here
    prev_upper = 5.0, prev_count = 3
    bucket_width = 7.5 - 5.0 = 2.5
    bucket_count = 7 - 3 = 4
    fraction = (4.0 - 3) / 4 = 0.25
    result = 5.0 + (0.25 * 2.5) = 5.625

TTFT p50 = 5.625 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 8 = 7.6

Walk buckets:
  le=5.0, cumulative=3 < 7.6 --> continue
  le=7.5, cumulative=7 < 7.6 --> continue
  le=10.0, cumulative=8 >= 7.6 --> interpolate here
    prev_upper = 7.5, prev_count = 7
    bucket_width = 10.0 - 7.5 = 2.5
    bucket_count = 8 - 7 = 1
    fraction = (7.6 - 7) / 1 = 0.6
    result = 7.5 + (0.6 * 2.5) = 9.0

TTFT p95 = 9.0 seconds
```

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 76.63 tok/s |
| TTFT avg | 5.270s |
| TTFT p50 | 5.625s |
| TTFT p95 | 9.0s |
| Completed Requests | 6 of 10 (decode), 8 of 10 (TTFT) |

