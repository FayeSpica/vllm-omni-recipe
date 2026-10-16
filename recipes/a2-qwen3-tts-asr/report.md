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

### Qwen3-TTS：MTP full-loop + Talker FULL 补测（2026-10-15）

在前述 MRV2、BSND/native GQA、QKV/Norm/RoPE 融合组合上，进一步启用 MTP 15 轮 full-loop graph 和 Talker 外层 FULL。模型、文本、参考音频、固定 ramp `[4,4,8,16,25]`

以下为**单次启动结果**，C1/C8各16请求、C40为80请求，共112请求通过音频基本校验；与前述三轮中位数分开列示，尚未完成多轮交替A/B及WER/音质验收。

| 并发 | 首包 P95（ms） | RTF P95 | 音频吞吐（秒音频/秒） | 零缓冲连续 |
|---:|---:|---:|---:|---:|
| 1 | 193.2 | 0.223 | 4.52 | 16/16 |
| 8 | 733.0 | 0.400 | 20.26 | 16/16 |
| 40 | 2778.8 | 1.005 | 43.52 | 0/80 |

相对上一轮同worktree、已开启MTP full-loop、Talker仍为PIECEWISE的单轮结果，单独切换Talker FULL的观察收益：

| 并发 | 首包 P95 PIECEWISE→FULL（ms） | 首包降低 | RTF P95 PIECEWISE→FULL | RTF降低 | 音频吞吐 PIECEWISE→FULL |
|---:|---:|---:|---:|---:|---:|
| 1 | 257.0 → 193.2 | 24.8% | 0.391 → 0.223 | 43.0% | 2.59 → 4.52 |
| 8 | 802.9 → 733.0 | 8.7% | 0.488 → 0.400 | 17.9% | 16.04 → 20.26 |
| 40 | 2606.6 → 2778.8 | -6.6% | 0.949 → 1.005 | -5.9% | 44.50 → 43.52 |

**本轮C8满足首包800ms、RTF<1且16/16零缓冲连续；C40仍不满足目标，没有观察到高并发改善。**

C1 profile支持收益来自减少分段图间等待：Stage0跨stream计算区间并集的空泡占比从50.55%降到7.63%；MTP单帧约10.8ms。

复现配置相对前述组合增加 `OMNI_MTP_FULL_LOOP_GRAPH=1`，并将 `platforms.npu.stages` 中stage0的 `compilation_config.cudagraph_mode` 改为 `FULL`；保留 `enforce_eager: false`，stage1不变。

### Qwen3-TTS：vNPU切分补测（2026-10-16）

将一张 A2/910B3 切为两个 `vir10_3c_32g`（各 10 AI Core、标称 32GB），Stage0、Stage1 各占一个 vNPU，与整卡共享对照。两组均开启 QKV/BSND、MTP full-loop、Talker FULL 和 Code2Wav vocoder graph，固定 ramp `[4,4,8,16,25]`。每个 Stage 的显存占用比例由整卡的 0.3 调为半卡的 0.6，保持绝对预算接近。

每组单次启动、每档两轮，以下为两轮指标的中位数，格式为**整卡共享 → 两个半卡**：

- **C1**：首包 P95 188.6 → 219.7ms；RTF P95 0.223 → 0.260；音频吞吐 4.51 → 3.86 秒音频/秒（**下降 14.2%**）；零缓冲连续 32/32 → 32/32。
- **C8**：首包 P95 703.1 → 747.5ms；RTF P95 0.386 → 0.406；音频吞吐 20.59 → 19.70 秒音频/秒（**下降 4.3%**）；零缓冲连续 27/32 → 27/32。
- **C40**：首包 P95 2599.4 → 2792.4ms；RTF P95 1.009 → 1.067；音频吞吐 43.47 → 41.55 秒音频/秒（**下降 4.4%**）；零缓冲连续 0/160 → 0/160。

### Qwen3-ASR-1.7B

采用 Rust 串行 Mel 前端，模型、FULL 图及调度参数保持一致。

| 并发 | Python→Rust 吞吐（请求/秒） | 吞吐提升 | Rust 首 token P95（ms） | Rust 端到端 P95（ms） |
|---:|---:|---:|---:|---:|
| 64 | 51.47 → 75.77 | +47.21% | 826.9 | 1071.3 |
| 128 | 59.30 → 98.13 | +65.48% | 1466.0 | 1922.8 |

**Rust 前端使 1.7B 吞吐提升 47.2%–65.5%。**

### Qwen3-ASR-1.7B：修改参数补测（2026-10-16）

A2 原容器、相同 5 秒音频和 Rust 二进制；预处理 workers/permits 从 8/8 调整为 32/32，保持 `ASR_MEL_THREADS=0`，图模式从 FULL 改为 FULL_AND_PIECEWISE，图桶 `[1,2,4,8,16,32,64,128]`。三次独立启动，C64 每轮 320 请求、C128 每轮 640 请求，2,880 个正式请求全部成功且转写与探针一致。以下为三轮指标中位数。

| 并发 | 原→新吞吐（请求/秒） | 吞吐变化 | 新首 token P50/P95（ms） | 新完整返回 P50/P95（ms） |
|---:|---:|---:|---:|---:|
| 64 | 75.77 → 72.14 | -4.8% | 465.6 / 1050.5 | 766.2 / 1260.7 |
| 128 | 98.13 → 95.03 | -3.2% | 597.7 / 1359.8 | 1189.6 / 1973.1 |

**未观察到吞吐提升，建议保留 8/8 + FULL。**

### Qwen3-ASR-0.6B 补测

同样使用 Rust 串行 Mel + FULL。

| 并发 | 吞吐（请求/秒） | 三轮范围 | 首 token P95（ms） | 端到端 P95（ms） | 相对历史 1.7B Rust |
|---:|---:|---:|---:|---:|---:|
| 64 | 79.33 | 77.38–82.77 | 799.1 | 1013.3 | +4.7% |
| 128 | 104.13 | 99.72–105.07 | 1257.5 | 1731.2 | +6.1% |

0.6B 吞吐较历史 1.7B Rust 结果提升约 4.7%–6.1%，本次配置下优势有限。

### Qwen3-ASR-0.6B：修改参数补测（2026-10-16）

A2 原容器、相同 5 秒音频，参数调整如下；其余服务参数保持原配置。

| 参数 | 原配置 | 补测配置 |
|---|---|---|
| `VLLM_MAX_AUDIO_PREPROCESS_WORKERS` | 8 | 32 |
| `ASR_DIAG_PERMITS` | 8 | 32 |
| `cudagraph_mode` | `FULL` | `FULL_AND_PIECEWISE` |
| `ASR_MEL_THREADS` | 0 | 0（保持串行 Mel） |

| 并发 | 原8/8 + FULL→新32/32 + 混合图吞吐（请求/秒） | 吞吐变化 | 新首token P50/P95（ms） | 新完整返回P50/P95（ms） |
|---:|---:|---:|---:|---:|
| 64 | 79.33 → 75.84 | -4.4% | 439.0 / 843.4 | 731.8 / 1147.0 |
| 128 | 104.13 → 95.28 | -8.5% | 560.2 / 1462.0 | 1130.5 / 2001.9 |

**未观察到收益，保留前述8/8 + FULL配置。** 三次独立启动，共 2,880 个正式请求全部成功。原值为前一天三轮记录，本轮同时修改前端并发和图模式，未做交错重测，不能独立归因。

### Qwen3-ASR：10 秒音频识别耗时补测

A2 单卡，Rust 串行 Mel + FULL。以下为上传完整 10 秒音频后的响应耗时，不包含音频采集等待。

| 模型 | 并发 | 首段文本 P50/P95（ms） | 完整返回 P50/P95（ms） | 吞吐（请求/秒） |
|---|---:|---:|---:|---:|
| Qwen3-ASR-1.7B | 1 | 155.1 / 160.4 | 305.8 / 320.5 | 3.24 |
| Qwen3-ASR-1.7B | 64 | 653.0 / 1066.4 | 1108.2 / 1505.4 | 52.03 |
| Qwen3-ASR-1.7B | 128 | 668.7 / 1943.9 | 1799.7 / 2908.5 | 61.02 |
| Qwen3-ASR-0.6B | 1 | 142.6 / 144.6 | 282.3 / 287.5 | 3.52 |
| Qwen3-ASR-0.6B | 64 | 557.2 / 1053.7 | 1040.9 / 1453.7 | 55.91 |
| Qwen3-ASR-0.6B | 128 | 609.8 / 1842.6 | 1673.7 / 2597.4 | 65.05 |

**单并发完整返回：1.7B P95 约 321ms，0.6B 约 287ms，降低约 10.3%。**

### A100 对照：Qwen3-ASR，10 秒音频

A100-SXM4-80GB 单卡，镜像 `vllm/vllm-omni:v0.31.0rc1`，Python 前端、BF16，使用与 A2 相同的 10 秒音频。

| 模型 | 并发 | 首段文本 P50/P95（ms） | 完整返回 P50/P95（ms） | 吞吐（请求/秒） |
|---|---:|---:|---:|---:|
| Qwen3-ASR-1.7B | 1 | 79.6 / 86.7 | 139.6 / 146.5 | 6.96 |
| Qwen3-ASR-1.7B | 64 | 797.7 / 1301.5 | 1116.0 / 1743.5 | 48.29 |
| Qwen3-ASR-1.7B | 128 | 2125.5 / 2549.4 | 2431.7 / 2989.9 | 48.66 |
| Qwen3-ASR-0.6B | 1 | 65.0 / 72.7 | 101.2 / 112.7 | 9.29 |
| Qwen3-ASR-0.6B | 64 | 843.1 / 1266.2 | 1090.2 / 1659.6 | 51.30 |
| Qwen3-ASR-0.6B | 128 | 2022.6 / 2604.1 | 2300.8 / 2990.0 | 51.14 |

**单并发完整返回：1.7B P50/P95 约 140/146ms，0.6B 约 101/113ms。**

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


## 附录：GLM-TTS / GLM-ASR A2 实测数据

测试日期：2026-10-14，A2 单卡。以下为各轮指标的中位数。

### GLM-TTS

模型 `zai-org/GLM-TTS`，与 Qwen3-TTS 使用相同中文文本和参考音频，流式音色克隆。

| 并发 | 首包 P50（ms） | 首包 P95（ms） | RTF P50 | RTF P95 | 音频吞吐（秒音频/秒） | 零缓冲连续 |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 1420.1 | 12250.9 | 0.440 | 1.647 | 1.411 | 0/48 |
| 2 | 4941.9 | 14480.8 | 0.866 | 1.900 | 2.027 | 0/48 |
| 4 | 6403.7 | 12315.6 | 1.141 | 1.854 | 3.208 | 0/48 |
| 8 | 9686.1 | 10841.4 | 1.875 | 2.093 | 4.150 | 0/48 |

192 个正式请求均通过音频基本校验，但已测各档均未满足首包 800ms、RTF<1 的组合目标，零缓冲连续均未通过。当前 AR 为 BF16 + FULL，DiT/vocoder 为 FP32 eager；结果反映当前适配配置，不代表 GLM-TTS 的模型性能上限。

### GLM-ASR-Nano-2512

模型 `zai-org/GLM-ASR-Nano-2512`，使用相同 5 秒音频，FULL 图桶和主要调度参数与 Qwen3-ASR 对齐。

| 前端 | 并发 | 吞吐（请求/秒） | 首段文本 P95（ms） | 完整返回 P95（ms） |
|---|---:|---:|---:|---:|
| Python | 64 | 20.98 | 2901.29 | 4038.42 |
| Python | 128 | 21.07 | 5634.10 | 6695.89 |
| Rust 串行 Mel | 64 | 31.56 | 2180.51 | 2356.52 |
| Rust 串行 Mel | 128 | 35.14 | 3976.59 | 4247.79 |

Rust 相比 GLM Python 吞吐分别提升 **50.4% / 66.7%**；Rust 三轮共 2,880 个正式请求全部成功。GLM 使用专门适配的 Rust 二进制；输入虽为 5 秒，仍保留模型默认 30 秒音频 padding。与 Qwen3-ASR 的比较属于不同模型的历史同口径服务对照，未做语料级识别精度比较。
