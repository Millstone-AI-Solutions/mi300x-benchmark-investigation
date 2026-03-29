# MI300X Inference Performance Investigation

## Background

Millstone AI publishes independent LLM inference benchmarks measuring user-facing performance. Our benchmark suite is NVIDIA-only, and we're expanding to AMD GPUs starting with MI300X.

During initial testing on a single MI300X VM (Hot Aisle, `enc1-gpuvm009`), the MI300X performed below initial expectations given the specs. This is our first time working with AMD/ROCm, so the gap may well be a configuration issue on our end that we haven't been able to identify.

This repo contains our test data, configurations, startup logs, and raw Prometheus metrics. Some raw metric snapshots from early test runs were not preserved.

---

## Hardware Comparison

| Spec | 1x MI300X | 2x RTX Pro 6000 Blackwell |
|------|-----------|---------------------------|
| VRAM | 192 GB HBM3 | 192 GB GDDR7 (96 GB x 2) |
| Memory Bandwidth | 5.3 TB/s | ~1.8 TB/s per card |
| Interconnect | Single device | PCIe 5.0 TP2 |
| Expected Advantage | MI300X should outperform | |

## Model

**Qwen/Qwen3.5-122B-A10B-FP8:** MoE (122B total, 10B active), FP8.

---

## Results Summary

### Decode Speed (tok/sec per user)

| Scenario | MI300X vLLM v1 | MI300X vLLM v2 | MI300X SGLang | 2x RTX Pro 6000 |
|----------|---------------|----------------|---------------|-----------------|
| 1K ctx / 1 req | 86.0 | 82.0 | 44.0 | **106.0** |
| 1K ctx / 2 req | **79.0** | 73.0 | 40.0 | 76.8 |
| 32K ctx / 1 req | 83.0 | 86.0 | 40.0 | **100.5** |
| 32K ctx / 2 req | **77.0** | 75.0 | 37.0 | 72.6 |
| 128K ctx / 1 req | 76.1 | 74.6 | 49.2 | **89.4** |
| 128K ctx / 5 req | 17.6 | 17.9 | **26.5** | 21.1 |

### TTFT (seconds)

| Scenario | MI300X vLLM v1 | MI300X vLLM v2 | MI300X SGLang | 2x RTX Pro 6000 |
|----------|---------------|----------------|---------------|-----------------|
| 1K ctx / 1 req | 0.1 | 0.1 | 1.6 | 0.1 |
| 1K ctx / 2 req | 0.2 | 0.2 | 1.7 | **0.1** |
| 32K ctx / 1 req | 33.5* | 31.6* | 4.8 | **2.1** |
| 32K ctx / 2 req | 5.3 | 6.2 | 7.3 | **3.2** |
| 128K ctx / 1 req | 17.3 | 16.5 | 25.3 | **12.2** |
| 128K ctx / 5 req | 48.8 | 47.7 | 113.3 | **33.6** |

\* Includes JIT compilation outlier

> **Calculations**: Full derivations (raw counter values, deltas, histogram analysis) in [`vllm-v1`](raw-metrics/vllm-v1/calculations.md) | [`vllm-v2`](raw-metrics/vllm-v2/calculations.md) | [`sglang`](raw-metrics/sglang/calculations.md). 1K and 32K scenarios have complete raw Prometheus snapshots; 128K snapshots were not saved.

---

## Configurations Tested

| # | Engine | Image | Config | Startup Log | Raw Metrics |
|---|--------|-------|--------|-------------|-------------|
| 1 | vLLM v1 | `vllm/vllm-openai-rocm:latest` (v0.18.0) | [yml](configs/mi300x-vllm-v1.yml) | [log](logs/vllm-v1-startup.log) | [metrics](raw-metrics/vllm-v1/) |
| 2 | vLLM v2 | `rocm/vllm-dev:nightly` (v0.18.1rc1) | [yml](configs/mi300x-vllm-v2.yml) | [log](logs/vllm-v2-startup.log) | [metrics](raw-metrics/vllm-v2/) |
| 3 | SGLang | `lmsysorg/sglang:v0.5.9-rocm700-mi30x` | [yml](configs/mi300x-sglang.yml) | [log](logs/sglang-startup.log) | [metrics](raw-metrics/sglang/) |
| 4 | NVIDIA baseline | `vllm/vllm-openai:cu130-nightly` (TP2, FlashInfer) | [yml](configs/rtx-pro-6000-baseline.yml) | - | - |

All three MI300X configs use AITER Flash Attention and AITER FP8 MoE backends (`VLLM_ROCM_USE_AITER=1`). Full docker-compose files with all env vars and flags are in `configs/`.

---

## Notable Log Warnings

We noticed these warnings in the startup logs and wanted to flag them.

### Missing tuned GEMM/MoE configs

Both vLLM configurations log warnings about missing tuned configs for the 122B model's matrix shapes (from [`logs/vllm-v2-startup.log`](logs/vllm-v2-startup.log)):
```
[aiter] shape is M:32768, N:20480, K:3072, not found tuned config in a8w8_blockscale_tuned_gemm.csv, will use default config!
[aiter] shape is M:32768, N:3072, K:8192, not found tuned config ...
[aiter] shape is M:32768, N:2048, K:3072, not found tuned config ...
[aiter] shape is M:32768, N:3072, K:1024, not found tuned config ...
[aiter] shape is M:1034, N:20480, K:3072, not found tuned config ...
[aiter] shape is M:1034, N:17408, K:3072, not found tuned config ...
```

The tuned config files that are loaded reference DeepSeek V3 and Qwen3 235B but not the 122B:
```
[aiter] merge tuned file under model_configs/ and configs/
  .../a8w8_blockscale_tuned_gemm.csv
  .../model_configs/a8w8_blockscale_tuned_gemm_ds_v3.csv
  .../model_configs/a8w8_blockscale_tuned_gemm_qwen3_235b.csv
```

Same pattern for MoE configs (only `tuned_fmoe_qwen3_235b.csv`, no 122B equivalent).

### JIT compilation times

The v2 startup log shows JIT compilation during startup:
- Graph compilation: 87.88s
- AITER module builds: module_quant (41.7s), module_gemm_a8w8_blockscale (116.0s), module_moe_asm (48.8s)

We also saw one large TTFT outlier in the 32K single-user test JIT compilation.

---

## Environment

```
Host:       enc1-gpuvm009 (Hot Aisle)
GPU:        1x AMD Instinct MI300X (192 GB HBM3)
ROCm:       7.2.0
OS:         Ubuntu 22.04
NUMA:       Balancing disabled (0)
rocm-libs:  7.2.0.70200-43~22.04
```

Full details: [`environment.md`](environment.md)

---

## What We Tried

1. **Followed AMD's Day 0 deployment guide** for Qwen 3.5 on MI300X
2. **Followed AMD's Developer Cloud guide (same model)**: [OpenClaw on AMD Developer Cloud: Qwen 3.5 and SGLang](https://www.amd.com/en/developer/resources/technical-articles/2026/openclaw-on-amd-developer-cloud-qwen-3-5-and-sglang.html)
3. **Tested three images** (vLLM official ROCm, vLLM ROCm nightly, SGLang ROCm)
4. **Confirmed AITER backends are active** (ROCM_AITER_FA for attention, AITER for FP8 MoE)
5. **Disabled NUMA balancing** as recommended
6. **Set recommended environment variables** (VLLM_ROCM_USE_AITER, TORCH_BLAS_PREFER_HIPBLASLT, etc.)

---

## Methodology

See [`methodology.md`](methodology.md) for the condensed test methodology, or view the [full public methodology](https://www.millstoneai.com/inference-benchmark-methodology) for complete details.

Quick summary for replication:

| Parameter | Value |
|-----------|-------|
| Input tokens | 1,024 (1K) or 32,768 (32K), calibrated via model tokenizer |
| Output tokens | 1,024, EOS disabled |
| Prompt caching | Disabled & unique prefix per request |
| Concurrency | 1 and 2 concurrent users (locust, serial requests) |
| Stability | Runs until requests/sec is within 10% for 3 consecutive readings |
| Metrics source | Engine `/metrics` endpoint (Prometheus), before/after delta snapshots |
| Load generator | Locust-based, gradual ramp-up (~5s to full concurrency) |

---

## Repo Structure

```
mi300x-benchmark-investigation/
├── README.md                    # This file
├── methodology.md               # Condensed test methodology and replication guide
├── environment.md               # System/software versions
├── configs/
│   ├── mi300x-vllm-v1.yml      # vllm/vllm-openai-rocm:latest (v0.18.0)
│   ├── mi300x-vllm-v2.yml      # rocm/vllm-dev:nightly (v0.18.1rc1)
│   ├── mi300x-sglang.yml       # lmsysorg/sglang:v0.5.9-rocm700-mi30x
│   └── rtx-pro-6000-baseline.yml  # NVIDIA baseline config
├── results-summary.md           # All results in one comparison view
├── logs/
│   ├── vllm-v1-startup.log     # Shows AITER backend selection
│   ├── vllm-v2-startup.log     # Shows missing tuned GEMM configs
│   ├── vllm-v2-requests.log    # Runtime request/throughput logs
│   └── sglang-startup.log      # SGLang initialization
└── raw-metrics/
    ├── vllm-v1/                 # Prometheus snapshots + calculations
    │   ├── calculations.md      # Full math showing how results were derived
    │   ├── quick_1u_1k_start.txt
    │   ├── quick_1u_1k_end.txt
    │   └── ...
    ├── vllm-v2/
    └── sglang/
```

---

## Published Benchmark Reference

For context on what a finished benchmark looks like from us:
- [Qwen3.5-122B-A10B-FP8 on 2x RTX Pro 6000 Blackwell](https://www.millstoneai.com/inference-benchmark/qwen3-5-122b-a10b-fp8-2x-rtx-pro-6000-blackwell)

## Contact

Jacob Mills
Millstone AI Solutions - [millstoneai.com](https://www.millstoneai.com)
