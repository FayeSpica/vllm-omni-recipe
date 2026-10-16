# A3 Qwen3-TTS 性能报告

硬件：单卡 A3；软件：vLLM 0.29.0、CANN 9.1.0。测试日期：2026-10-15。

## 1. 原始需求

单卡40并发、首包≤800ms、RTF<1，同时关注流式音频连续性。

使用 `Qwen/Qwen3-TTS-12Hz-1.7B-Base` 音色克隆、约36字中文和同一条参考音频。首包、RTF均报告P95，数据取各轮对应指标的三轮中位数，连续性汇总三轮请求。

## 2. 基线

同源码已有优化基线：MRV2、QKV/Norm/RoPE融合、BSND/native GQA；Talker PIECEWISE，MTP body graph。固定ramp `[4,4,8,16,25]`。

| 并发 | 首包 P95（ms） | RTF P95 | 音频吞吐（秒音频/秒） | 零缓冲连续 |
|---:|---:|---:|---:|---:|
| 1 | 163.8 | 0.265 | 3.89 | 48/48 |
| 8 | 514.4 | 0.382 | 20.67 | 48/48 |
| 40 | 1783.4 | 0.870 | 48.45 | 92/240 |

## 3. 优化收益

在基线上开启 **Talker FULL + MTP 15轮 full-loop graph**，其余条件不变。

| 并发 | 首包 P95 基线→优化（ms） | RTF P95 基线→优化 | 音频吞吐基线→优化 | 吞吐提升 | 优化后零缓冲连续 |
|---:|---:|---:|---:|---:|---:|
| 1 | 163.8 → 143.8 | 0.265 → 0.218 | 3.89 → 4.61 | +18.4% | 48/48 |
| 8 | 514.4 → 494.8 | 0.382 → 0.380 | 20.67 → 21.19 | +2.5% | 48/48 |
| 40 | 1783.4 → 1629.3 | 0.870 → 0.861 | 48.45 → 48.86 | +0.9% | 62/240 |

**收益主要集中在单并发；C8满足首包800ms、RTF<1且全部连续。C40首包仍超标，连续性未改善，尚未达到单卡40并发目标。**

同机同源码三轮交替对照，共672个正式请求通过HTTP、PCM长度/时长、非静音及连续性复算。基线已包含融合优化，两项graph为组合收益；未加入新增Code2Wav NPU graph，未做WER/长文本音质验证。C40连续通过数轮间波动较大。

## 4. 启动参数

镜像：`quay.nju.edu.cn/fayeomni/vllm-omni:v0.29.0-a3`。需使用本次验证的MRV2/QKV/full-loop源码及Ascend捕获状态修复；原始镜像不包含这些实验补丁。

CANN环境初始化后，在本目录运行：

```bash
export ASCEND_RT_VISIBLE_DEVICES=4
export OMNI_MTP_QKV_FUSION=1
export OMNI_MTP_FULL_LOOP_GRAPH=1

vllm serve Qwen/Qwen3-TTS-12Hz-1.7B-Base \
  --omni \
  --trust-remote-code \
  --deploy-config deploy.yaml \
  --host 0.0.0.0 \
  --port 18169
```

完整配置见 [deploy.yaml](deploy.yaml)。基线使用 [baseline.yaml](baseline.yaml)，并设置 `OMNI_MTP_FULL_LOOP_GRAPH=0`；设备编号可按实际空闲卡调整。Stage1 YAML中的PIECEWISE不表示其外层已经入图，本轮Code2Wav仍走原有NPU执行路径。

逐轮原始汇总见 [results.json](results.json)。这些结果由流式请求脚本采集，不标记为原始 `vllm bench` 数据。
