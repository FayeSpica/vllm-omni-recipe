# A2 Qwen3-TTS / Qwen3-ASR 性能报告

硬件：单卡 A2；软件：vLLM 0.29.0、CANN 9.1.0。

## 1. 原始需求

- **TTS**：单卡 40 并发、首包≤800ms、RTF<1，同时关注流式音频连续性。后续优先保障首包和 RTF，评估实际可支持的并发。
- **ASR**：评估单卡识别吞吐和响应时延，比较 Python 与 Rust 前端，并补测 0.6B 模型。

本文以首包 P95 和 RTF P95 判断 TTS 性能，连续性单独统计。TTS 使用 1.7B Base 音色克隆、约 36 字中文；ASR 使用同一条 5 秒音频。数据取三轮对应指标的中位数。

## 2. 基线

### Qwen3-TTS-1.7B Base

原生 0.29.0，固定 ramp `[4,4,8,16,25]`，Talker PIECEWISE。

| 并发 | 首包 P95（ms） | RTF P95 | 音频吞吐（秒音频/秒） | 零缓冲连续 |
|---:|---:|---:|---:|---:|
| 1 | 347.3 | 0.523 | 1.97 | 48/48 |
| 8 | 1094.6 | 0.686 | 11.79 | 35/48 |
| 16 | 1558.3 | 0.928 | 17.56 | 0/96 |
| 32 | 2632.0 | 1.362 | 23.62 | 0/192 |
| 40 | 3296.4 | 1.834 | 23.06 | 0/240 |

### Qwen3-ASR-1.7B

Python 前端，FULL 图模式。

| 并发 | Python 吞吐（请求/秒） | 首 token P95（ms） | 端到端 P95（ms） |
|---:|---:|---:|---:|
| 64 | 51.47 | 1195.8 | 1466.1 |
| 128 | 59.30 | 2255.6 | 3261.5 |

## 3. 优化收益

### Qwen3-TTS

启用 MRV2、BSND/native GQA，以及 QKV 提取、Q/K Norm、RoPE 融合。

| 并发 | 首包 P95 基线→组合（ms） | RTF P95 基线→组合 | 音频吞吐基线→组合 | 吞吐提升 | 组合零缓冲连续 |
|---:|---:|---:|---:|---:|---:|
| 1 | 347.3 → 287.2 | 0.523 → 0.505 | 1.97 → 2.04 | +3.5% | 48/48 |
| 8 | 1094.6 → 782.3 | 0.686 → 0.577 | 11.79 → 13.76 | +16.8% | 45/48 |
| 16 | 1558.3 → 1289.4 | 0.928 → 0.660 | 17.56 → 24.73 | +40.8% | 39/96 |
| 32 | 2632.0 → 2224.9 | 1.362 → 0.880 | 23.62 → 39.22 | +66.0% | 12/192 |
| 40 | 3296.4 → 2808.7 | 1.834 → 1.032 | 23.06 → 42.05 | +82.4% | 0/240 |

**已测 C8 满足首包 800ms、RTF<1，但仍有少量欠载（45/48 连续）；C1 为 48/48 连续。C16/C32/C40 首包超标，目前未达到原始单卡 40 并发目标。**

本组对比同时涉及代码版本变化，以上为组合方案收益，不能拆分为各项优化的独立收益。


C40 补测：基线与组合各三轮、240 个正式请求。组合首包 P95 2809ms、RTF P95 1.032，音频吞吐 42.05 秒音频/秒（提升 82.4%），零缓冲连续 0/240；三轮均未达到首包 800ms、RTF<1 的组合目标。

### Qwen3-ASR-1.7B

采用 Rust 串行 Mel 前端，模型、FULL 图及调度参数保持一致。

| 并发 | Python→Rust 吞吐（请求/秒） | 吞吐提升 | Rust 首 token P95（ms） | Rust 端到端 P95（ms） |
|---:|---:|---:|---:|---:|
| 64 | 51.47 → 75.77 | +47.21% | 826.9 | 1071.3 |
| 128 | 59.30 → 98.13 | +65.48% | 1466.0 | 1922.8 |

**Rust 前端使 1.7B 吞吐提升 47.2%–65.5%。**

### Qwen3-ASR-0.6B 补测

同样使用 Rust 串行 Mel + FULL。

| 并发 | 吞吐（请求/秒） | 三轮范围 | 首 token P95（ms） | 端到端 P95（ms） | 相对历史 1.7B Rust |
|---:|---:|---:|---:|---:|---:|
| 64 | 79.33 | 77.38–82.77 | 799.1 | 1013.3 | +4.7% |
| 128 | 104.13 | 99.72–105.07 | 1257.5 | 1731.2 | +6.1% |

0.6B 吞吐较历史 1.7B Rust 结果提升约 4.7%–6.1%，本次配置下优势有限。

## 4. 启动参数

### TTS 配置

- 模型：`Qwen/Qwen3-TTS-12Hz-1.7B-Base`。
- `model_runner: v2`，Talker PIECEWISE。
- `OMNI_MTP_QKV_FUSION=1`：开启 BSND/native GQA 和 QKV/Norm/RoPE 融合。
- 固定 ramp：`[4,4,8,16,25]`，关闭 adaptive。

### ASR 配置

- 模型：`Qwen/Qwen3-ASR-1.7B` 或 `Qwen/Qwen3-ASR-0.6B`。
- Rust 串行 Mel：`VLLM_USE_RUST_FRONTEND=1`、`ASR_MEL_THREADS=0`。
- FULL 图桶：`[1,2,4,8,16,32,64,128]`。
- `max-num-seqs=128`、`max-num-batched-tokens=2048`、`max-model-len=512`。
- 显存比例：0.7；关闭前缀缓存和 processor cache，开启异步调度。

### 完整启动命令

以下假设 CANN 环境已初始化、对应优化源码及 Ascend 修复已安装，且 `vllm` 来自测试虚拟环境。三个服务分别运行；TTS 使用完整 YAML，ASR 使用支持音频转写的实验 Rust 二进制。缓存、离线模式和调试环境变量已省略。0.6B 的 Rust tokenizer.json 仍需按已测方式由官方词表准备。使用模型 ID 会解析本地缓存或下载模型；报告数据对应已归档的模型版本。

#### TTS 优化组合

```bash
export ASCEND_RT_VISIBLE_DEVICES=1
export OMNI_MTP_QKV_FUSION=1

vllm serve Qwen/Qwen3-TTS-12Hz-1.7B-Base \
  --omni \
  --trust-remote-code \
  --host 0.0.0.0 \
  --port 18109
```

#### TTS 完整 deploy.yaml

相对原生 0.29.0 默认配置，新增顶层 `model_runner: v2`，并在 connector.extra 中设置 `codec_chunk_adaptive: false`、`codec_chunk_ramp: [4,4,8,16,25]`。相比本报告固定 ramp 基线， `model_runner: v2` 是新增配置。BSND/QKV 融合由环境变量开启。

```yaml
async_chunk: true
connectors:
  connector_of_shared_memory:
    name: SharedMemoryConnector
    extra:
      codec_streaming: true
      connector_get_sleep_s: 0.01
      connector_get_max_wait_first_chunk: 3000
      connector_get_max_wait: 300
      codec_chunk_frames: 25
      codec_left_context_frames: 72
      initial_codec_chunk_frames: 1
      decode_batch_max_size: 4
      codec_chunk_adaptive: false
      codec_chunk_ramp:
      - 4
      - 4
      - 8
      - 16
      - 25
stages:
- stage_id: 0
  max_num_seqs: 64
  gpu_memory_utilization: 0.3
  trust_remote_code: true
  enable_prefix_caching: false
  async_scheduling: true
  max_num_batched_tokens: 32768
  max_model_len: 4096
  devices: '0'
  output_connectors:
    to_stage_1: connector_of_shared_memory
  default_sampling_params:
    temperature: 0.9
    top_k: 50
    max_tokens: 4096
    min_tokens: 2
    repetition_penalty: 1.05
  silence_ban_frames: 0
  subtalker_sampling_params:
    do_sample: true
    temperature: 0.9
    top_k: 50
    top_p: 1.0
- stage_id: 1
  max_num_seqs: 64
  gpu_memory_utilization: 0.3
  enforce_eager: false
  trust_remote_code: true
  enable_prefix_caching: false
  async_scheduling: true
  max_num_batched_tokens: 65536
  max_model_len: 65536
  devices: '0'
  input_connectors:
    from_stage_0: connector_of_shared_memory
  default_sampling_params:
    temperature: 0.0
    top_p: 1.0
    top_k: -1
    max_tokens: 65536
    repetition_penalty: 1.0
platforms:
  rocm:
    stages:
    - stage_id: 1
      enforce_eager: true
  npu:
    stages:
    - stage_id: 0
      enforce_eager: false
      compilation_config:
        cudagraph_mode: PIECEWISE
    - stage_id: 1
      compilation_config:
        cudagraph_mode: PIECEWISE
model_runner: v2
```

#### ASR 1.7B Rust + FULL

```bash
export ASCEND_RT_VISIBLE_DEVICES=1
export VLLM_USE_RUST_FRONTEND=1
export VLLM_RUST_FRONTEND_PATH=/tmp/asr-rust-native-bench-a2-5s-20261008/vllm-rs
export VLLM_MAX_AUDIO_PREPROCESS_WORKERS=8
export ASR_DIAG_PERMITS=8
export ASR_MEL_THREADS=0

vllm serve Qwen/Qwen3-ASR-1.7B \
  --trust-remote-code \
  --max-num-seqs 128 \
  --max-num-batched-tokens 2048 \
  --max-model-len 512 \
  --no-enable-prefix-caching \
  --mm-processor-cache-gb 0 \
  --async-scheduling \
  --gpu-memory-utilization 0.7 \
  --compilation-config '{"cudagraph_capture_sizes":[1,2,4,8,16,32,64,128],"cudagraph_mode":"FULL"}' \
  --host 0.0.0.0 \
  --port 18089
```

#### ASR 0.6B Rust + FULL

```bash
export ASCEND_RT_VISIBLE_DEVICES=1
export VLLM_USE_RUST_FRONTEND=1
export VLLM_RUST_FRONTEND_PATH=/tmp/asr-rust-native-bench-a2-5s-20261008/vllm-rs
export VLLM_MAX_AUDIO_PREPROCESS_WORKERS=8
export ASR_DIAG_PERMITS=8
export ASR_MEL_THREADS=0

vllm serve Qwen/Qwen3-ASR-0.6B \
  --trust-remote-code \
  --max-num-seqs 128 \
  --max-num-batched-tokens 2048 \
  --max-model-len 512 \
  --no-enable-prefix-caching \
  --mm-processor-cache-gb 0 \
  --async-scheduling \
  --gpu-memory-utilization 0.7 \
  --compilation-config '{"cudagraph_capture_sizes":[1,2,4,8,16,32,64,128],"cudagraph_mode":"FULL"}' \
  --host 0.0.0.0 \
  --port 18119
```

[C40 逐轮数据](c40/summary.json) · [C40 补测说明](c40/report.md)
