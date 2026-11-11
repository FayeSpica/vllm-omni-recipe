# A2 Qwen3-TTS 补测（2026-11-11）

主模型：Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice，快照0c0e3051f131929182e2c023b9537f8b1c68adfe。单卡910B3，两阶段共卡，四API共享单套引擎。输入为四条中文短文本、Vivian/Serena。与2026-10-15的Base音色克隆不是同任务，不给出直接性能倍数。

源码：实验worktree基底ed694eba15a44c5470eff5c9401e880e979f7a79加native-port.patch，补丁SHA256 87ff8986a063c2def5087ad161c1a70a098fece64dec2da1ebff6c101d6776fc。并非原生发行版或#9068最新head。

本轮保持Stage0 max_num_seqs=64，Stage1 max_num_seqs=128（增量codec请求状态槽），decode_batch_max_size=8。状态容量扩展不代表模型计算批量扩大到128。

统计方法：两轮C64预热不计入结果；每档三轮，正式请求C1每轮16条，其他max(32,2*C)。首包/RTF为逐请求合并P95，并记录逐轮P95及范围。边界以三轮全部达标判断。零缓冲从客户端收到第一段PCM立即开始计时，任意正欠载即失败；没有加入可听阈值。300ms缓冲另列，不能替代零缓冲指标。

原64槽配置在C72触发incremental codec has no free request slots并导致Stage1退出；失败数据不用于计算有效性能。新配置单独重测。此文件后续补入完整原始汇总和边界。

## 已完成结果

8,514条正式请求全部成功。边界按三轮全部满足：零缓冲C5/C6、首包500ms C65/C66、RTF<0.8 C71/C72、RTF<1 C73/C74。每对分别表示最后通过/相邻未全通过。首包与RTF共同约束最高已测C65。不是128个整数穷举，也不是跨启动SLA。

- `summary.json`：每档合并指标、三轮明细。
- `deploy.yaml`：本轮完整配置，Stage0并发64、Stage1状态容量128。
- `raw.tar.gz`：逐请求结果、PCM到达时间、测试脚本、启动脚本、配置、输入哈希和服务日志；未包含PCM本体。
- `validation.json`：本轮音频有效性和源码哈希检查。
- 原始PCM保留在A2：`/tmp/a2-tts-rsi-ab-20261109/boundary128-20261111/`。

该配置需安装实测实验worktree，不是下载YAML后即获得所有代码优化。代码补丁与精确SHA记录在self-llm的`perf/qwen3-tts/a2-0330rc1-mtp-frame-20261110/`，尚未提交到上游。

## 后续Stage0上限128

[最新高并发补测](stage0-128/README.md)将Stage0上限也改为128，单独保存配置、逐轮数据和有效性失败；不与本目录原Stage0上限64数据混合。
