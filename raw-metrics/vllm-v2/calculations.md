# vLLM v2 - Calculation Breakdown

**Engine**: vLLM v0.18.1rc1 (rocm/vllm-dev:nightly) | **Date**: 2026-03-25

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
  vllm:generation_tokens_total  = 50.0
  vllm:request_decode_time_seconds_sum   = 0.614251785000306
  vllm:request_decode_time_seconds_count = 1.0

From quick_1u_1k_end.txt:
  vllm:generation_tokens_total  = 4344.0
  vllm:request_decode_time_seconds_sum   = 52.755275389001326
  vllm:request_decode_time_seconds_count = 5.0

Deltas:
  generation_tokens  = 4344.0 - 50.0     = 4294.0 tokens
  decode_time_sum    = 52.755 - 0.614    = 52.141 seconds
  decode_time_count  = 5 - 1             = 4 completed requests

Calculation:
  decode_speed = 4294.0 / 52.141 = 82.35 tokens/sec
```

### TTFT

```
From quick_1u_1k_start.txt:
  vllm:time_to_first_token_seconds_sum   = 0.5815811157226562
  vllm:time_to_first_token_seconds_count = 1.0

From quick_1u_1k_end.txt:
  vllm:time_to_first_token_seconds_sum   = 1.3418867588043213
  vllm:time_to_first_token_seconds_count = 6.0

Average TTFT:
  delta_sum   = 1.342 - 0.582 = 0.760 seconds
  delta_count = 6 - 1         = 5 requests
  ttft_avg    = 0.760 / 5     = 0.152 seconds
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
| 0.08 | 0 | 0 | 0 |
| 0.1 | 0 | 0 | 0 |
| **0.25** | **0** | **5** | **5** |
| 0.5 | 0 | 5 | 5 |
| 0.75 | 1 | 6 | 5 |
| 1.0 | 1 | 6 | 5 |
| 2.5 | 1 | 6 | 5 |
| 5.0 | 1 | 6 | 5 |
| 7.5 | 1 | 6 | 5 |
| 10.0 | 1 | 6 | 5 |
| 20.0 | 1 | 6 | 5 |
| 40.0 | 1 | 6 | 5 |
| 80.0 | 1 | 6 | 5 |
| 160.0 | 1 | 6 | 5 |
| 640.0 | 1 | 6 | 5 |
| 2560.0 | 1 | 6 | 5 |

All 5 requests fell into the le=0.25 bucket (TTFT between 0.1s and 0.25s).

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 5 (from highest finite bucket)
target_count = 0.50 * 5 = 2.5

Walk buckets (skipping zeros):
  le=0.25, cumulative=5 >= 2.5 --> interpolate here
    prev_upper = 0.1, prev_count = 0
    bucket_width = 0.25 - 0.1 = 0.15
    bucket_count = 5 - 0 = 5
    fraction = (2.5 - 0) / 5 = 0.5
    result = 0.1 + (0.5 * 0.15) = 0.175

TTFT p50 = 0.175 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 5 = 4.75

Walk buckets:
  le=0.25, cumulative=5 >= 4.75 --> interpolate here
    fraction = (4.75 - 0) / 5 = 0.95
    result = 0.1 + (0.95 * 0.15) = 0.2425

TTFT p95 = 0.243 seconds
```

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 82.35 tok/s |
| TTFT avg | 0.152s |
| TTFT p50 | 0.175s |
| TTFT p95 | 0.243s |

---

## Scenario 2: 2 Users, 1K Context

**Config**: 2 concurrent users, 1024 input tokens, 1024 max output tokens, 10 requests
**Files**: `quick_2u_1k_start.txt`, `quick_2u_1k_end.txt`

### Decode Speed

```
From quick_2u_1k_start.txt:
  vllm:generation_tokens_total  = 4344.0
  vllm:request_decode_time_seconds_sum   = 52.755275389001326
  vllm:request_decode_time_seconds_count = 5.0

From quick_2u_1k_end.txt:
  vllm:generation_tokens_total  = 20786.0
  vllm:request_decode_time_seconds_sum   = 276.6315607630004
  vllm:request_decode_time_seconds_count = 21.0

Deltas:
  generation_tokens  = 20786.0 - 4344.0  = 16442.0 tokens
  decode_time_sum    = 276.632 - 52.755  = 223.876 seconds
  decode_time_count  = 21 - 5            = 16 completed requests

Calculation:
  decode_speed = 16442.0 / 223.876 = 73.44 tokens/sec
```

### TTFT

```
From quick_2u_1k_start.txt:
  vllm:time_to_first_token_seconds_sum   = 1.3418867588043213
  vllm:time_to_first_token_seconds_count = 6.0

From quick_2u_1k_end.txt:
  vllm:time_to_first_token_seconds_sum   = 4.479485273361206
  vllm:time_to_first_token_seconds_count = 23.0

Average TTFT:
  delta_sum   = 4.479 - 1.342 = 3.138 seconds
  delta_count = 23 - 6        = 17 requests
  ttft_avg    = 3.138 / 17    = 0.185 seconds
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
| 0.08 | 0 | 0 | 0 |
| 0.1 | 0 | 0 | 0 |
| **0.25** | **5** | **20** | **15** |
| **0.5** | **5** | **22** | **17** |
| 0.75 | 6 | 23 | 17 |
| 1.0 | 6 | 23 | 17 |
| 2.5 | 6 | 23 | 17 |
| 5.0 | 6 | 23 | 17 |
| 7.5 | 6 | 23 | 17 |
| 10.0 | 6 | 23 | 17 |
| 20.0 | 6 | 23 | 17 |
| 40.0 | 6 | 23 | 17 |
| 80.0 | 6 | 23 | 17 |
| 160.0 | 6 | 23 | 17 |
| 640.0 | 6 | 23 | 17 |
| 2560.0 | 6 | 23 | 17 |

15 requests had TTFT <= 0.25s, 2 more had TTFT between 0.25s and 0.5s.

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 17
target_count = 0.50 * 17 = 8.5

Walk buckets:
  le=0.25, cumulative=15 >= 8.5 --> interpolate here
    prev_upper = 0.1, prev_count = 0
    bucket_width = 0.25 - 0.1 = 0.15
    bucket_count = 15 - 0 = 15
    fraction = (8.5 - 0) / 15 = 0.5667
    result = 0.1 + (0.5667 * 0.15) = 0.185

TTFT p50 = 0.185 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 17 = 16.15

Walk buckets:
  le=0.25, cumulative=15 < 16.15 --> continue
  le=0.5, cumulative=17 >= 16.15 --> interpolate here
    prev_upper = 0.25, prev_count = 15
    bucket_width = 0.5 - 0.25 = 0.25
    bucket_count = 17 - 15 = 2
    fraction = (16.15 - 15) / 2 = 0.575
    result = 0.25 + (0.575 * 0.25) = 0.394

TTFT p95 = 0.394 seconds
```

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 73.44 tok/s |
| TTFT avg | 0.185s |
| TTFT p50 | 0.185s |
| TTFT p95 | 0.394s |

---

## Scenario 3: 1 User, 32K Context

**Config**: 1 concurrent user, 32768 input tokens, 1024 max output tokens, 10 requests
**Files**: `quick_1u_32k_start.txt`, `quick_1u_32k_end.txt`

### Decode Speed

```
From quick_1u_32k_start.txt:
  vllm:generation_tokens_total  = 20786.0
  vllm:request_decode_time_seconds_sum   = 276.6315607630004
  vllm:request_decode_time_seconds_count = 21.0

From quick_1u_32k_end.txt:
  vllm:generation_tokens_total  = 27623.0
  vllm:request_decode_time_seconds_sum   = 356.0909523749997
  vllm:request_decode_time_seconds_count = 27.0

Deltas:
  generation_tokens  = 27623.0 - 20786.0 = 6837.0 tokens
  decode_time_sum    = 356.091 - 276.632 = 79.459 seconds
  decode_time_count  = 27 - 21           = 6 completed requests

Calculation:
  decode_speed = 6837.0 / 79.459 = 86.04 tokens/sec
```

Note: Only 6 of 10 requests completed during the test window. 4 requests likely
timed out or were still in-flight when the test ended.

### TTFT

```
From quick_1u_32k_start.txt:
  vllm:time_to_first_token_seconds_sum   = 4.479485273361206
  vllm:time_to_first_token_seconds_count = 23.0

From quick_1u_32k_end.txt:
  vllm:time_to_first_token_seconds_sum   = 225.66890788078308
  vllm:time_to_first_token_seconds_count = 30.0

Average TTFT:
  delta_sum   = 225.669 - 4.479 = 221.189 seconds
  delta_count = 30 - 23         = 7 requests
  ttft_avg    = 221.189 / 7     = 31.60 seconds
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
160s and 640s. This is the massive outlier that dominates the average -- the same
pattern observed in the first vLLM run (`vllm_20260325_162221`).

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
| Decode Speed | 86.04 tok/s |
| TTFT avg | 31.60s |
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
  vllm:generation_tokens_total  = 27623.0
  vllm:request_decode_time_seconds_sum   = 356.0909523749997
  vllm:request_decode_time_seconds_count = 27.0

From quick_2u_32k_end.txt:
  vllm:generation_tokens_total  = 34112.0
  vllm:request_decode_time_seconds_sum   = 443.037301336999
  vllm:request_decode_time_seconds_count = 33.0

Deltas:
  generation_tokens  = 34112.0 - 27623.0 = 6489.0 tokens
  decode_time_sum    = 443.037 - 356.091 = 86.946 seconds
  decode_time_count  = 33 - 27           = 6 completed requests

Calculation:
  decode_speed = 6489.0 / 86.946 = 74.63 tokens/sec
```

### TTFT

```
From quick_2u_32k_start.txt:
  vllm:time_to_first_token_seconds_sum   = 225.66890788078308
  vllm:time_to_first_token_seconds_count = 30.0

From quick_2u_32k_end.txt:
  vllm:time_to_first_token_seconds_sum   = 275.5278730392456
  vllm:time_to_first_token_seconds_count = 38.0

Average TTFT:
  delta_sum   = 275.528 - 225.669 = 49.859 seconds
  delta_count = 38 - 30           = 8 requests
  ttft_avg    = 49.859 / 8        = 6.232 seconds
```

**Deltas** (non-zero only):

| Bucket (le) | Delta |
|-------------|-------|
| 7.5 | 6 |
| 10.0 | 8 |
| 20.0 | 8 |
| 40.0 | 8 |
| 80.0 | 8 |
| 160.0 | 8 |
| 640.0 | 8 |
| 2560.0 | 8 |

6 requests had TTFT <= 7.5s, 2 more between 7.5s and 10.0s.

**TTFT p50 Calculation** (quantile=0.50):
```
total_count  = 8
target_count = 0.50 * 8 = 4.0

Walk buckets:
  le=7.5, cumulative=6 >= 4.0 --> interpolate here
    prev_upper = 5.0, prev_count = 0
    bucket_width = 7.5 - 5.0 = 2.5
    bucket_count = 6 - 0 = 6
    fraction = (4.0 - 0) / 6 = 0.6667
    result = 5.0 + (0.6667 * 2.5) = 6.667

TTFT p50 = 6.667 seconds
```

**TTFT p95 Calculation** (quantile=0.95):
```
target_count = 0.95 * 8 = 7.6

Walk buckets:
  le=7.5, cumulative=6 < 7.6 --> continue
  le=10.0, cumulative=8 >= 7.6 --> interpolate here
    prev_upper = 7.5, prev_count = 6
    bucket_width = 10.0 - 7.5 = 2.5
    bucket_count = 8 - 6 = 2
    fraction = (7.6 - 6) / 2 = 0.8
    result = 7.5 + (0.8 * 2.5) = 9.5

TTFT p95 = 9.5 seconds
```

### Summary

| Metric | Value |
|--------|-------|
| Decode Speed | 74.63 tok/s |
| TTFT avg | 6.232s |
| TTFT p50 | 6.667s |
| TTFT p95 | 9.5s |
| Completed Requests | 6 of 10 (decode), 8 of 10 (TTFT) |

