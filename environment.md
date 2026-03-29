# Test Environment

## MI300X Host

| Component | Value |
|-----------|-------|
| Provider | Hot Aisle |
| Hostname | enc1-gpuvm009 |
| GPU | 1x AMD Instinct MI300X (192 GB HBM3) |
| OS | Ubuntu 22.04 |
| ROCm Version | 7.2.0 |
| ROCm SMI Version | 4.0.0+fc0010cf6a |
| ROCm SMI LIB Version | 7.8.0 |
| rocm-libs Package | 7.2.0.70200-43~22.04 |
| NUMA Balancing | Disabled (0) |
| Transformers (in container) | 4.57.6 |

### Software Versions by Configuration

| Config | Container Image | Engine Version |
|--------|----------------|----------------|
| vLLM v1 | `vllm/vllm-openai-rocm:latest` | v0.18.0 |
| vLLM v2 | `rocm/vllm-dev:nightly` | v0.18.1rc1.dev93+gb73b5b062 |
| SGLang | `lmsysorg/sglang:v0.5.9-rocm700-mi30x` | v0.5.9 |

## NVIDIA Baseline Host

| Component | Value |
|-----------|-------|
| GPU | 2x NVIDIA RTX Pro 6000 Blackwell (96 GB GDDR7 each) |
| Container Image | `vllm/vllm-openai:cu130-nightly` |
| Attention Backend | FlashInfer |
| Tensor Parallel | 2 (across PCIe 5.0) |