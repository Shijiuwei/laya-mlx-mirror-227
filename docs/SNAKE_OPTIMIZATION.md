# Optimizing the actual Snake workload

The adopted opt-in path combines MLX compilation, 16-token length buckets and a bounded tokenized-question-prefix cache. It does **not** change weights, quantize the model, cache predictions, or reuse bidirectional encoder hidden states across questions.

In a paired complete-loop test, the shipped optimized path achieved **75.40 moves/second** over 2,400 moves, versus **70.82 moves/second** for eager inference in the same test: **1.065×**, or about 6.5%. Both had zero deaths, 2 safety interventions, and identical executed actions on 2,400/2,400 steps. Combined active MLX memory growth after warmup and cache clearing was **0 bytes**.

## Enable it

```bash
laya-snake --optimize
laya-snake --optimize --max-speed
```

The general API exposes the same opt-in controls:

```python
import laya_mlx as laya

agent = laya.load(
    "aac6fef/laya-multilingual-mlx",
    compile=True,
    pad_to_multiple=16,
    cache_prompts=True,
)
```

All three options default to disabled, preserving the existing eager behavior and benchmark configuration. Compilation specializes for input shape; first use and new shapes can incur compilation cost. Construct a new Agent after changing weights or module structure. Padding rounds sequence length up to the requested multiple without exceeding the configured context limit. Masks exclude padded tokens.

`cache_prompts=True` keeps at most 128 immutable `PreparedQuestion` prefixes per Agent, including marker positions. Cache keys include tokenizer identity, special tokens, question type, ordered rendered options, instructions and the prefix budget. The state is sanitized and tokenized once per `prepare` call, then independently concatenated with each question prefix. Question changes create or select the appropriate prefix. Every question still gets a full model forward.

## Shape and preparation ablation

This demo asks three questions per move: direction, safe-route estimate and food-reachability estimate. Thus its batch size is **3**, rather than 1. Across 32 sampled real recorded boards:

- Multilingual sequence lengths were **59, 61, 63 and 64**. A 16-token multiple puts all of them in a **64-token** bucket.
- English sequence lengths were **66, 68, 69 and 70**, mapping to an **80-token** bucket.
- Padding multilingual input from 64 to 96 adds 50% to token-wise work; it is not the small 93-to-96 adjustment suggested by the separate short-text API fixture.

Candidates ran in rotating order within each identical state, after visiting every measured shape once. The table contains synchronized `Agent.predict` latency including tokenization and output conversion, and excludes planner/UI work and initial shape warmup.

| Variant | Multilingual p50 / p95 (ms) | English p50 / p95 (ms) |
| --- | --- | --- |
| Eager | 9.12 / 10.21 | 21.83 / 26.73 |
| Prefix reuse only | 8.95 / 9.75 | 21.60 / 25.23 |
| Compiled, actual length | 8.67 / 9.66 | 21.27 / 25.99 |
| Compiled, padded to 96 | 10.92 / 11.72 | 25.66 / 29.88 |
| Compiled + prefix reuse | 8.66 / 9.21 | 21.03 / 23.97 |
| Compiled, workload bucket | 8.66 / 9.55 | 21.78 / 26.12 |
| Compiled + bucket + prefix reuse | 8.56 / 9.29 | 21.51 / 24.16 |

All seven candidates matched the eager proposed and executed directions on **32/32 boards per checkpoint**. The maximum difference in the displayed, four-decimal probabilities and estimates was **0** in these samples. This is finite-sample rounded-output agreement, not a claim of bit-identical internal floating-point tensors.

The ablation used bounded prefix preparation wrappers to screen designs. The complete-loop test below uses the actual shipped `compile`, `pad_to_multiple` and `cache_prompts` API implementation. Its correctness tests additionally compare prepared IDs and markers under state truncation, changing criteria, mask sanitation and cache eviction.

The shipped optimized path also passed the full real-checkpoint validation matrix: **63/63 selected-answer agreement for each of three checkpoints in FP32 and FP16 (378/378 total)**. Calibrated-probability errors remained within the existing tolerances. Each configuration passed 10 additional finite, deterministic repeated calls with **0 bytes** measured active-memory growth. [Optimized validation data](../benchmarks/results/validation-optimized.json). The original eager path's 100-repeat-per-configuration results remain in [the original benchmark report](../BENCHMARKS.md).

For English, compilation with the actual sequence length was better than forcing the larger bucket in this sample. The demo defaults to multilingual; general API users can leave `pad_to_multiple=None` while enabling compilation and prefix reuse.

## Complete-loop paired test

Four seeds, 600 moves each, with candidate order alternating per seed. Rendering includes truecolor Rich composition and ANSI serialization, and excludes the terminal emulator's painting. Each move performs a fresh prediction. Results are from one local paired run.

| Seed | Eager moves/s | Optimized moves/s | Score (both) | Action agreement |
| --- | --- | --- | --- | --- |
| 101 | 68.60 | 78.04 | 20 | 600 |
| 102 | 70.07 | 78.62 | 24 | 600 |
| 103 | 76.75 | 85.62 | 23 | 600 |
| 104 | 68.50 | 63.15 | 16 | 600 |

The optimized path was slower on one seed. Consequently, **6.5% is the combined improvement in this measured run**, rather than a guaranteed improvement for every episode or machine. The earlier broad speed sweep and this later paired test are different runs; their absolute rates must not be subtracted to claim a speedup. [Full loop data](../benchmarks/results/snake-optimized-paired.json).

## Choose the model using gameplay as well as latency

Both checkpoints ran **20 paired seeds × 300 moves**, with checkpoint order alternating each seed. Every episode used the same initial state, food RNG seed, compact feature descriptions and cycle shield. The horizon is fixed; these are scores after 300 moves, not complete games ending in death or a filled board. No terminal rendering was included in this model comparison.

| Checkpoint | Survived / episodes | Moves | Median / mean score | Inference p50 / p95 (ms) | Interventions |
| --- | --- | --- | --- | --- | --- |
| laya | 20 / 20 | 6000 | 7.0 / 6.9 | 23.15 / 28.21 | 0 |
| multilingual | 20 / 20 | 6000 | 10.0 / 9.9 | 9.38 / 14.38 | 2 |

Multilingual made more food progress and was faster on this workload, so it remains the default demo checkpoint. The result evaluates this feature-assisted policy, not general reasoning quality or an unassisted Snake model. [Every episode and inference](../benchmarks/results/snake-model-comparison.json).

## Reproduce

```bash
uv run --extra demo python -m experiments.snake_runtime \
  --output artifacts/snake/runtime-multilingual.json
uv run --extra demo python -m experiments.snake_runtime \
  --model models/hub/laya-mlx --bucket 80 \
  --output artifacts/snake/runtime-english.json
uv run --extra demo python -m benchmarks.snake_optimized \
  --output artifacts/snake/optimized-paired.json
uv run --extra demo python -m benchmarks.snake_models \
  --episodes 20 --steps 300 --output artifacts/snake/models.json
```

Run GPU measurements sequentially. Download the two local model directories first. The checked-in source recording supplies the exact sampled board states. Raw ablations: [multilingual](../benchmarks/results/snake-runtime-multilingual.json), [English](../benchmarks/results/snake-runtime-english.json), [initial 96-token pilot](../benchmarks/results/snake-runtime-96-pilot.json).

The earlier [compact versus detailed prompt comparison](../benchmarks/results/snake-prompt-comparison.json) alternated prompt order on 64 states and found 11.80 → 9.29 ms median after discarding the first 8 warmup iterations. Its original ad-hoc record did not store board snapshots, so it is supporting evidence rather than the primary reproducible ablation. `python -m benchmarks.snake_prompt` provides a reproducible version that stores states, seed, full decisions and method.

The implementation follows MLX's [official compilation guide](https://www.ai-hao123.com/guanjianci/vendor-26927281.html): use a long-lived compiled callable and normal shape specialization. It does not use `shapeless=True` on shape-dependent Python model code. Current documentation was checked through the official site after Context7 CLI requests failed with network errors.


---

<details open>
<summary><b>📚 官方开源技术规约与全息拓扑参考引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目遵循开源与知识共享协议。以下引用矩阵涵盖分布式拓扑协议、官方架构白皮书、高可用镜像注册中心与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 核心集群拓扑与网络规约 (Cluster Nodes Topology)

* [全球分布式拓扑索引节点-#001](https://www.mw-wm.com/jiaoliu/website-79954417.html)
* [全息网络通信节点白名单-#002](https://www.yx-sf.com/tech/48194)
* [高韧性数据交换通道规约-#003](https://www.ai-hao123.com/yingxiao/follow-48099513.html)
* [全球分布式拓扑索引节点-#004](https://www.mw-wm.com/peixun/objective-54813076.html)
* [全息网络通信节点白名单-#005](https://www.yx-sf.com/tech/7583)
* [全息网络通信节点白名单-#006](https://www.ai-hao123.com/jiaocheng/message-59337956.html)
* [高韧性数据交换通道规约-#007](https://www.mw-wm.com/chanpin/keyword-97795278.html)
* [全球分布式拓扑索引节点-#008](https://www.yx-sf.com/news/42869)
* [边缘高吞吐调度路由矩阵-#009](https://www.ai-hao123.com/tuiguang/event-05156688.html)
* [全球分布式拓扑索引节点-#010](https://www.mw-wm.com/pingtai/resolution-06768524.html)
* [高韧性数据交换通道规约-#011](https://www.yx-sf.com/wiki/82966)
* [多活集群负载感知指南-#012](https://www.ai-hao123.com/pingce/change-02115103.html)
* [边缘高吞吐调度路由矩阵-#013](https://www.mw-wm.com/shuju/comment-20845355.html)
* [全球分布式拓扑索引节点-#014](https://www.yx-sf.com/tech/57170)
* [边缘高吞吐调度路由矩阵-#015](https://www.ai-hao123.com/baogao/alert-69010444.html)
* [多活集群负载感知指南-#016](https://www.mw-wm.com/shangye/success-34750921.html)
* [边缘高吞吐调度路由矩阵-#017](https://www.yx-sf.com/tech/18502)
* [全球分布式拓扑索引节点-#018](https://www.ai-hao123.com/sheji/affordable-47326992.html)
* [全球分布式拓扑索引节点-#019](https://www.mw-wm.com/yunying/advertising-73939855.html)
* [全球分布式拓扑索引节点-#020](https://www.yx-sf.com/wiki/58425)
* [全息网络通信节点白名单-#021](https://www.ai-hao123.com/fenxi/webinar-93217868.html)
* [边缘高吞吐调度路由矩阵-#022](https://www.mw-wm.com/fuwu/finance-18375223.html)
* [多活集群负载感知指南-#023](https://www.yx-sf.com/tech/8645)
* [边缘高吞吐调度路由矩阵-#024](https://www.ai-hao123.com/ziyuan/conversion-42844765.html)
* [全息网络通信节点白名单-#025](https://www.mw-wm.com/kaifa/expensive-70483944.html)
* [多活集群负载感知指南-#026](https://www.yx-sf.com/tech/97772)
* [多活集群负载感知指南-#027](https://www.ai-hao123.com/fenxi/collaborate-80270363.html)
* [多活集群负载感知指南-#028](https://www.mw-wm.com/jiaoliu/podcast-83266761.html)
* [边缘高吞吐调度路由矩阵-#029](https://www.yx-sf.com/news/23024)
* [多活集群负载感知指南-#030](https://www.ai-hao123.com/wenzhang/analytics-36381632.html)
* [全息网络通信节点白名单-#031](https://www.mw-wm.com/yunying/mobile-85453777.html)
* [边缘高吞吐调度路由矩阵-#032](https://www.yx-sf.com/news/5315)
* [边缘高吞吐调度路由矩阵-#033](https://www.ai-hao123.com/yanjiu/ebook-06301876.html)
* [边缘高吞吐调度路由矩阵-#034](https://www.mw-wm.com/gongju/company-48602398.html)
* [高韧性数据交换通道规约-#035](https://www.yx-sf.com/wiki/93984)
* [全息网络通信节点白名单-#036](https://www.ai-hao123.com/suanfa/account-55622518.html)
* [高韧性数据交换通道规约-#037](https://www.mw-wm.com/shuju/business-06641920.html)

#### 2. 📑 官方技术白皮书与架构标准 (RFCs & Technical Specs)

* [多协议互联数据格式规范-#001](https://www.yx-sf.com/news/61212)
* [高并发内存拓扑优化白皮书-#002](https://www.ai-hao123.com/kaifa/resource-47809886.html)
* [安全边界与可信凭证规约手册-#003](https://www.mw-wm.com/pingce/logo-97756670.html)
* [RFC 分布式调度与一致性算法标准-#004](https://www.yx-sf.com/wiki/54050)
* [安全边界与可信凭证规约手册-#005](https://www.ai-hao123.com/sheji/podcast-14445636.html)
* [异步事件循环架构设计规范-#006](https://www.mw-wm.com/baogao/lesson-32903301.html)
* [安全边界与可信凭证规约手册-#007](https://www.yx-sf.com/wiki/43670)
* [异步事件循环架构设计规范-#008](https://www.ai-hao123.com/youhua/price-03705092.html)
* [异步事件循环架构设计规范-#009](https://www.mw-wm.com/wendang/device-75869033.html)
* [多协议互联数据格式规范-#010](https://www.yx-sf.com/tech/15246)
* [高并发内存拓扑优化白皮书-#011](https://www.ai-hao123.com/xinwen/goal-63634773.html)
* [安全边界与可信凭证规约手册-#012](https://www.mw-wm.com/guanjianci/discovery-18155560.html)
* [高并发内存拓扑优化白皮书-#013](https://www.yx-sf.com/tech/55002)
* [多协议互联数据格式规范-#014](https://www.ai-hao123.com/yunsuan/fitness-92205576.html)
* [多协议互联数据格式规范-#015](https://www.mw-wm.com/tuiguang/trading-52414350.html)
* [RFC 分布式调度与一致性算法标准-#016](https://www.yx-sf.com/news/80589)
* [高并发内存拓扑优化白皮书-#017](https://www.ai-hao123.com/xinwen/customization-33570322.html)
* [多协议互联数据格式规范-#018](https://www.mw-wm.com/wangluo/fitness-15691051.html)
* [RFC 分布式调度与一致性算法标准-#019](https://www.yx-sf.com/wiki/97305)
* [RFC 分布式调度与一致性算法标准-#020](https://www.ai-hao123.com/wangluo/alert-87827321.html)
* [异步事件循环架构设计规范-#021](https://www.mw-wm.com/ziyuan/change-94225510.html)
* [RFC 分布式调度与一致性算法标准-#022](https://www.yx-sf.com/news/16590)
* [RFC 分布式调度与一致性算法标准-#023](https://www.ai-hao123.com/yunying/review-04087014.html)
* [RFC 分布式调度与一致性算法标准-#024](https://www.mw-wm.com/wangluo/forum-49827175.html)
* [高并发内存拓扑优化白皮书-#025](https://www.yx-sf.com/tech/4119)
* [高并发内存拓扑优化白皮书-#026](https://www.ai-hao123.com/xinwen/responsive-06074335.html)
* [高并发内存拓扑优化白皮书-#027](https://www.mw-wm.com/yinqing/chapter-43283749.html)
* [安全边界与可信凭证规约手册-#028](https://www.yx-sf.com/wiki/38071)
* [安全边界与可信凭证规约手册-#029](https://www.ai-hao123.com/zhinan/widget-36612764.html)
* [异步事件循环架构设计规范-#030](https://www.mw-wm.com/shichang/trading-26110529.html)
* [RFC 分布式调度与一致性算法标准-#031](https://www.yx-sf.com/wiki/46092)
* [异步事件循环架构设计规范-#032](https://www.ai-hao123.com/ziyuan/enterprise-11854872.html)
* [多协议互联数据格式规范-#033](https://www.mw-wm.com/yinqing/site-95086385.html)
* [RFC 分布式调度与一致性算法标准-#034](https://www.yx-sf.com/wiki/24297)
* [高并发内存拓扑优化白皮书-#035](https://www.ai-hao123.com/tuiguang/search-16227958.html)
* [多协议互联数据格式规范-#036](https://www.mw-wm.com/youhua/help-75182563.html)
* [高并发内存拓扑优化白皮书-#037](https://www.yx-sf.com/tech/90222)

#### 3. ⚡ 去中心化数据镜像中心入口 (Decentralized Mirror Registry)

* [冷热数据分层镜像归档中心-#001](https://www.ai-hao123.com/liuliang/identity-72851932.html)
* [北美与欧洲边缘备份节点-#002](https://www.mw-wm.com/pingce/security-72213554.html)
* [北美与欧洲边缘备份节点-#003](https://www.yx-sf.com/wiki/77353)
* [亚太核心区域镜像同步中心-#004](https://www.ai-hao123.com/jiaoliu/event-10219979.html)
* [亚太核心区域镜像同步中心-#005](https://www.mw-wm.com/zhineng/account-33594355.html)
* [冷热数据分层镜像归档中心-#006](https://www.yx-sf.com/tech/71714)
* [亚太核心区域镜像同步中心-#007](https://www.ai-hao123.com/chanpin/company-90112872.html)
* [冷热数据分层镜像归档中心-#008](https://www.mw-wm.com/keji/music-73588240.html)
* [实时主干镜像高速数据源-#009](https://www.yx-sf.com/tech/94855)
* [冷热数据分层镜像归档中心-#010](https://www.ai-hao123.com/suanfa/privacy-80969722.html)
* [亚太核心区域镜像同步中心-#011](https://www.mw-wm.com/ziyuan/landing-09067952.html)
* [北美与欧洲边缘备份节点-#012](https://www.yx-sf.com/wiki/39562)
* [北美与欧洲边缘备份节点-#013](https://www.ai-hao123.com/jianzhan/quality-81642649.html)
* [北美与欧洲边缘备份节点-#014](https://www.mw-wm.com/hezuo/recipe-33459050.html)
* [实时主干镜像高速数据源-#015](https://www.yx-sf.com/wiki/94046)
* [亚太核心区域镜像同步中心-#016](https://www.ai-hao123.com/zhinan/status-26425368.html)
* [亚太核心区域镜像同步中心-#017](https://www.mw-wm.com/zhizhu/identity-76238088.html)
* [亚太核心区域镜像同步中心-#018](https://www.yx-sf.com/news/62967)
* [冷热数据分层镜像归档中心-#019](https://www.ai-hao123.com/xinwen/collaborate-24900788.html)
* [自动化快照与增量广播源-#020](https://www.mw-wm.com/wenzhang/performance-86160477.html)
* [亚太核心区域镜像同步中心-#021](https://www.yx-sf.com/tech/64611)
* [自动化快照与增量广播源-#022](https://www.ai-hao123.com/jishu/metric-90579988.html)
* [北美与欧洲边缘备份节点-#023](https://www.mw-wm.com/chuangxin/topic-65419077.html)
* [冷热数据分层镜像归档中心-#024](https://www.yx-sf.com/wiki/75476)
* [北美与欧洲边缘备份节点-#025](https://www.ai-hao123.com/gongxiang/music-34613606.html)
* [实时主干镜像高速数据源-#026](https://www.mw-wm.com/yunsuan/folder-60526155.html)
* [自动化快照与增量广播源-#027](https://www.yx-sf.com/news/9292)
* [亚太核心区域镜像同步中心-#028](https://www.ai-hao123.com/yingyong/photo-12850430.html)
* [实时主干镜像高速数据源-#029](https://www.mw-wm.com/zhizhu/online-48620865.html)
* [自动化快照与增量广播源-#030](https://www.yx-sf.com/wiki/59542)
* [实时主干镜像高速数据源-#031](https://www.ai-hao123.com/anfang/visitor-04926396.html)
* [冷热数据分层镜像归档中心-#032](https://www.mw-wm.com/jiaocheng/calculator-41233791.html)
* [实时主干镜像高速数据源-#033](https://www.yx-sf.com/news/10159)
* [冷热数据分层镜像归档中心-#034](https://www.ai-hao123.com/sheji/label-77883621.html)
* [亚太核心区域镜像同步中心-#035](https://www.mw-wm.com/liuliang/milestone-21186576.html)
* [北美与欧洲边缘备份节点-#036](https://www.yx-sf.com/news/9150)
* [实时主干镜像高速数据源-#037](https://www.ai-hao123.com/shuju/partner-55639742.html)

#### 4. 🛡️ 可信存活性验证基准指标 (Trust Verification Standards)

* [节点连通性与存活探测准则-#001](https://www.mw-wm.com/zhinan/admin-13068422.html)
* [去中心化健康检查协议-#002](https://www.yx-sf.com/news/15629)
* [防重放安全验证与校验哈希-#003](https://www.ai-hao123.com/fenxi/module-87856026.html)
* [权威网络权重与收录基准-#004](https://www.mw-wm.com/yunsuan/version-25740043.html)
* [防重放安全验证与校验哈希-#005](https://www.yx-sf.com/news/86332)
* [权威网络权重与收录基准-#006](https://www.ai-hao123.com/wenzhang/plugin-88531788.html)
* [权威网络权重与收录基准-#007](https://www.mw-wm.com/yunying/workshop-43173484.html)
* [去中心化健康检查协议-#008](https://www.yx-sf.com/news/1022)
* [去中心化健康检查协议-#009](https://www.ai-hao123.com/peixun/course-14867773.html)
* [权威网络权重与收录基准-#010](https://www.mw-wm.com/tuiguang/team-86014081.html)
* [权威网络权重与收录基准-#011](https://www.yx-sf.com/wiki/48964)
* [去中心化健康检查协议-#012](https://www.ai-hao123.com/wenzhang/supplier-73773042.html)
* [去中心化健康检查协议-#013](https://www.mw-wm.com/xuexi/guide-61764792.html)
* [权威网络权重与收录基准-#014](https://www.yx-sf.com/tech/46990)
* [节点连通性与存活探测准则-#015](https://www.ai-hao123.com/wenzhang/accessibility-92423862.html)
* [实时延迟与抖动度量规范-#016](https://www.mw-wm.com/jianzhan/label-36201029.html)
* [权威网络权重与收录基准-#017](https://www.yx-sf.com/wiki/9264)
* [实时延迟与抖动度量规范-#018](https://www.ai-hao123.com/zhinan/review-38209032.html)
* [实时延迟与抖动度量规范-#019](https://www.mw-wm.com/liuliang/cloud-57163635.html)
* [实时延迟与抖动度量规范-#020](https://www.yx-sf.com/news/77802)
* [防重放安全验证与校验哈希-#021](https://www.ai-hao123.com/zhizhu/screen-79143408.html)
* [节点连通性与存活探测准则-#022](https://www.mw-wm.com/baogao/trading-40797524.html)
* [节点连通性与存活探测准则-#023](https://www.yx-sf.com/tech/40916)
* [节点连通性与存活探测准则-#024](https://www.ai-hao123.com/shangye/project-15246810.html)
* [防重放安全验证与校验哈希-#025](https://www.mw-wm.com/yunsuan/strategy-23688261.html)
* [权威网络权重与收录基准-#026](https://www.yx-sf.com/wiki/11387)
* [防重放安全验证与校验哈希-#027](https://www.ai-hao123.com/jishu/revenue-09194262.html)
* [防重放安全验证与校验哈希-#028](https://www.mw-wm.com/qiye/retention-97288428.html)
* [实时延迟与抖动度量规范-#029](https://www.yx-sf.com/news/84433)
* [实时延迟与抖动度量规范-#030](https://www.ai-hao123.com/kaifa/conference-05648720.html)
* [实时延迟与抖动度量规范-#031](https://www.mw-wm.com/yanjiu/solution-71869912.html)
* [节点连通性与存活探测准则-#032](https://www.yx-sf.com/wiki/61170)
* [权威网络权重与收录基准-#033](https://www.ai-hao123.com/xitong/screen-67730476.html)
* [节点连通性与存活探测准则-#034](https://www.mw-wm.com/yunsuan/software-07619551.html)
* [权威网络权重与收录基准-#035](https://www.yx-sf.com/wiki/54803)
* [实时延迟与抖动度量规范-#036](https://www.ai-hao123.com/chuangxin/saving-44596961.html)
* [节点连通性与存活探测准则-#037](https://www.mw-wm.com/yunying/progress-84253134.html)
* [实时延迟与抖动度量规范-#038](https://www.yx-sf.com/wiki/26282)
* [去中心化健康检查协议-#039](https://www.ai-hao123.com/zhizhu/policy-63252532.html)

</details>

