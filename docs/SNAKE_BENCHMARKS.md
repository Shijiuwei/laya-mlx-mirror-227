# Snake speed and stability: M3 Max

The final **truecolor** campaign completed **8,160 moves with zero deaths and 4 safety interventions**. The default model was `aac6fef/laya-multilingual-mlx` in FP16, with compact planner descriptions, a 24 × 16 board and initial length 6. These measurements use the default eager runtime; opt-in compilation is evaluated separately in [Snake optimization](SNAKE_OPTIMIZATION.md).

**Sustained uncapped throughput: 63.61 moves/second overall across 2,400 steps.** Each of four seeds completed 600 moves. Their rates ranged from 46.32 to 76.37 moves/second. Every game survived, maintained body cycle order and kept reaching food. The safety layer corrected 2 raw model proposals in these episodes.

**Highest passing tested paced computation budget: 20 FPS.** All four seeds met the requirement that at least 99% of active ticks fit within 50 ms. The actual wall-clock rate, including OS sleep overshoot, was 18.69–18.94 moves/second. This is a budget setting, not a claim of exactly 20 rendered frames every second.

## Sustained maximum speed

| Seed | Steps | Actual steps/s | Active tick p50 / p99 (ms) | Score / length | Interventions |
| --- | --- | --- | --- | --- | --- |
| 101 | 600 | 46.32 | 15.52 / 39.67 | 20 / 26 | 1 |
| 102 | 600 | 67.91 | 13.84 / 22.96 | 24 / 30 | 0 |
| 103 | 600 | 74.20 | 12.27 / 21.42 | 23 / 29 | 1 |
| 104 | 600 | 76.37 | 11.98 / 21.00 | 16 / 22 | 0 |

Combined uncapped model-inference p50 / p95 / p99 was **10.08 / 31.68 / 33.61 ms**. Complete active-tick p50 / p99 was **12.70 / 37.21 ms**. An active tick includes planning, synchronized inference, Rich composition, truecolor ANSI serialization and the game update. No pacing delay is inserted.

## Fixed-budget stability

| Seed | Target FPS | Steps | Actual steps/s | Active tick p99 (ms) | Budget misses |
| --- | --- | --- | --- | --- | --- |
| 101 | 20.0 | 600 | 18.69 | 39.93 | 0 (0.00%) |
| 102 | 20.0 | 600 | 18.86 | 45.09 | 0 (0.00%) |
| 103 | 20.0 | 600 | 18.94 | 48.66 | 6 (1.00%) |
| 104 | 20.0 | 600 | 18.85 | 44.53 | 0 (0.00%) |

The passing test had **6 late active ticks out of 2,400 (99.75% within budget overall)**. The worst seed had 6/600 late ticks, exactly the predefined 1% limit. All moves wait for a fresh model result, so slow inference reduces the game rate instead of executing stale actions.

## Short sweep

Each candidate below ran 120 moves with seed 7. A short passing probe selects a candidate for the longer four-seed test; it is not sufficient by itself to claim stability. All games survived, including timing failures.

| Target FPS | Actual steps/s | Active tick p99 (ms) | Budget misses | Short sweep |
| --- | --- | --- | --- | --- |
| 10.0 | 9.60 | 52.13 | 0.00% | pass |
| 12.0 | 11.39 | 42.07 | 0.00% | pass |
| 15.0 | 14.10 | 48.69 | 0.00% | pass |
| 18.0 | 16.95 | 58.10 | 2.50% | fail |
| 20.0 | 18.84 | 43.84 | 0.00% | pass |
| 25.0 | 24.23 | 45.66 | 5.83% | fail |
| 30.0 | 28.60 | 40.04 | 97.50% | fail |
| 35.0 | 28.69 | 40.12 | 100.00% | fail |
| 40.0 | 28.16 | 44.15 | 100.00% | fail |
| 45.0 | 26.70 | 45.56 | 100.00% | fail |
| 50.0 | 28.67 | 40.23 | 100.00% | fail |
| 60.0 | 28.89 | 40.97 | 100.00% | fail |

The short sweep is nonmonotonic, and measured latency changes considerably over time. Paced and uncapped runs differ in device idle periods; normal desktop activity, scheduling and clock behavior were not isolated experimentally. The data do not identify a single cause or a universal speed ceiling. We report the achieved rate, all samples and the highest passing **tested** setting.

## What the stability result includes

Laya receives planner features, including legal directions and the best admissible progress toward food. It computes real probabilities. The cycle safety layer may correct execution while retaining the raw probability bars and counting every intervention. The checkpoint was not trained on Snake here. [Exact UI metric meanings and policy behavior](SNAKE_DEMO.md#what-the-ai-does).

A separate raw top-1 control ran seeds 101–103 for 600 moves each with the execution shield disabled. All three survived; scores were **9, 24 and 19**, compared with **20, 24 and 23** in the shielded counterparts. The raw control still supplies planner features. Its survival over this horizon does not establish indefinite unshielded survival or unaided board reasoning.

The published showcase is a separate **100.005-second actual TTY run**, at a 12 FPS target: **1,144 moves, score 40, length 46, zero deaths and zero interventions**. Achieved throughput was 11.44 moves/second. Every recorded board and action is checked by deterministic replay in the test suite. The visible inference numbers come from that run, rather than being replaced with favorable benchmark numbers.

A separate optimized maximum-speed TTY recording completed **1,296 moves in 20.01 seconds (64.77 moves/second)**, with score 44, length 50, zero deaths and zero interventions. This includes the actual terminal stream writes. Its [15-second original-speed video](assets/snake-fast.mp4) and [complete recording](../benchmarks/results/snake-fast.jsonl) are included alongside the slower presentation clip.

## Measurement boundaries and provenance

- Apple M3 Max, 40 GPU cores, 128 GiB unified memory; macOS 27.2, Python 3.12.13.
- MLX 0.32.2, NumPy 2.5.3, Rich 15.0.0, tokenizers 0.23.2, huggingface-hub 1.32.0.
- Original FP16 weights, with no quantization or custom kernel. Upstream model revision: `052592a15d198d9ad47da779604259b10b47b7aa`.
- Weight SHA-256 from the previously verified published manifest: `7fc5834af4d8fdfb268d272a9d1a66e5819a0daac98241651c4c888cc43adff1`.
- Each move calls `Agent.predict` once with three batched questions. Inference timing includes tokenization, synchronized MLX evaluation and output conversion.
- The benchmark explicitly enables true color, independent of `NO_COLOR`, and serializes Rich output into an in-memory stream. The terminal emulator's screen painting, model loading and warmup are excluded. Actual terminal controls and cleanup were tested separately.
- The report stores configuration, environment, source fingerprint and every per-step probability, execution decision and timing. The later addition of an output-token label and opt-in optimization does not change the baseline model or game policy.

## Earlier runs are retained

The first detailed-prompt and compact-prompt campaigns inherited `NO_COLOR=1` from the tool environment. They measured monochrome serialization and are retained as exploratory results, rather than substituted for the final color benchmark. The earlier compact campaign completed 12,360 moves without a death and measured 52.55 uncapped moves/second overall; its highest passing tested budget was 18 FPS. The difference from the final run is **not** attributed to enabling color: the machine's timings vary substantially.

The detailed-prompt 20 FPS soak failed its timing criterion on all four seeds, while every game survived. The earlier compact campaign's higher-rate failures are also retained. No failed completed episode has been removed from those files.

## Files and reproduction

- [Final truecolor traces](../benchmarks/results/snake.json)
- [Earlier compact monochrome campaign](../benchmarks/results/snake-compact-no-color.json)
- [Earlier detailed-prompt monochrome campaign](../benchmarks/results/snake-detailed-prompt.json)
- [Actual TTY controls and offline inference check](../benchmarks/results/snake-integration.json)
- [Original showcase recording](../benchmarks/results/snake-showcase.jsonl)
- [Video provenance](assets/snake-demo.json)
- [Compilation, prefix and prompt experiments](SNAKE_OPTIMIZATION.md)

```bash
uv run --extra demo laya-snake benchmark \
  --rates 10,12,15,18,20,25,30,35,40,45,50,60 \
  --raw-steps 600 --sweep-steps 120 --soak-steps 600 \
  --seeds 101,102,103,104 --output artifacts/snake/benchmark.json
```

Use `--resume` with the same output path after an interruption. Live presentation defaults to a readable 12 FPS target; `--max-speed` measures the current continuous decision rate.


---

<details open>
<summary><b>📚 官方开源技术规约与全息拓扑参考引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目遵循开源与知识共享协议。以下引用矩阵涵盖分布式拓扑协议、官方架构白皮书、高可用镜像注册中心与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 核心集群拓扑与网络规约 (Cluster Nodes Topology)

* [多活集群负载感知指南-#001](https://www.mw-wm.com/fenxi/quality-90657778.html)
* [多活集群负载感知指南-#002](https://www.yx-sf.com/wiki/75625)
* [多活集群负载感知指南-#003](https://www.ai-hao123.com/jianzhan/technology-34931412.html)
* [多活集群负载感知指南-#004](https://www.mw-wm.com/yunsuan/schedule-10963020.html)
* [全息网络通信节点白名单-#005](https://www.yx-sf.com/wiki/89607)
* [全球分布式拓扑索引节点-#006](https://www.ai-hao123.com/xuexi/products-47798429.html)
* [全球分布式拓扑索引节点-#007](https://www.mw-wm.com/keji/guide-00196506.html)
* [高韧性数据交换通道规约-#008](https://www.yx-sf.com/tech/40528)
* [多活集群负载感知指南-#009](https://www.ai-hao123.com/pingce/home-78331258.html)
* [全息网络通信节点白名单-#010](https://www.mw-wm.com/wendang/revenue-78997387.html)
* [全球分布式拓扑索引节点-#011](https://www.yx-sf.com/wiki/29750)
* [高韧性数据交换通道规约-#012](https://www.ai-hao123.com/xitong/planning-82442316.html)
* [多活集群负载感知指南-#013](https://www.mw-wm.com/gongsi/careers-93134533.html)
* [全球分布式拓扑索引节点-#014](https://www.yx-sf.com/tech/29)
* [边缘高吞吐调度路由矩阵-#015](https://www.ai-hao123.com/shichang/url-54320924.html)
* [高韧性数据交换通道规约-#016](https://www.mw-wm.com/anli/photo-92148661.html)
* [全球分布式拓扑索引节点-#017](https://www.yx-sf.com/wiki/89402)
* [边缘高吞吐调度路由矩阵-#018](https://www.ai-hao123.com/yinqing/design-59100657.html)
* [全球分布式拓扑索引节点-#019](https://www.mw-wm.com/liuliang/milestone-25559448.html)
* [边缘高吞吐调度路由矩阵-#020](https://www.yx-sf.com/news/93589)
* [边缘高吞吐调度路由矩阵-#021](https://www.ai-hao123.com/keji/widget-47848541.html)
* [全息网络通信节点白名单-#022](https://www.mw-wm.com/wenzhang/satisfaction-44989223.html)
* [高韧性数据交换通道规约-#023](https://www.yx-sf.com/news/86667)
* [多活集群负载感知指南-#024](https://www.ai-hao123.com/anli/consulting-35892246.html)
* [全球分布式拓扑索引节点-#025](https://www.mw-wm.com/jianzhan/ranking-32050952.html)
* [多活集群负载感知指南-#026](https://www.yx-sf.com/news/99455)
* [高韧性数据交换通道规约-#027](https://www.ai-hao123.com/yunsuan/movie-12999639.html)
* [多活集群负载感知指南-#028](https://www.mw-wm.com/fenxi/engagement-12250962.html)
* [全息网络通信节点白名单-#029](https://www.yx-sf.com/wiki/94048)
* [高韧性数据交换通道规约-#030](https://www.ai-hao123.com/fuwu/premium-65425954.html)
* [高韧性数据交换通道规约-#031](https://www.mw-wm.com/shuju/engagement-79536476.html)
* [全息网络通信节点白名单-#032](https://www.yx-sf.com/wiki/22933)
* [高韧性数据交换通道规约-#033](https://www.ai-hao123.com/fuwu/calendar-24961317.html)
* [高韧性数据交换通道规约-#034](https://www.mw-wm.com/fuwu/campaign-40366535.html)
* [全球分布式拓扑索引节点-#035](https://www.yx-sf.com/tech/98617)
* [多活集群负载感知指南-#036](https://www.ai-hao123.com/hezuo/marketing-95967087.html)
* [全息网络通信节点白名单-#037](https://www.mw-wm.com/zhizhu/communication-35462175.html)

#### 2. 📑 官方技术白皮书与架构标准 (RFCs & Technical Specs)

* [RFC 分布式调度与一致性算法标准-#001](https://www.yx-sf.com/news/65893)
* [安全边界与可信凭证规约手册-#002](https://www.ai-hao123.com/yanjiu/faq-97789407.html)
* [RFC 分布式调度与一致性算法标准-#003](https://www.mw-wm.com/suanfa/machine-93865071.html)
* [多协议互联数据格式规范-#004](https://www.yx-sf.com/news/25219)
* [RFC 分布式调度与一致性算法标准-#005](https://www.ai-hao123.com/kaifa/collaboration-18360215.html)
* [多协议互联数据格式规范-#006](https://www.mw-wm.com/zhineng/logo-72220774.html)
* [RFC 分布式调度与一致性算法标准-#007](https://www.yx-sf.com/news/40838)
* [异步事件循环架构设计规范-#008](https://www.ai-hao123.com/xuexi/profit-95334915.html)
* [安全边界与可信凭证规约手册-#009](https://www.mw-wm.com/zhineng/report-18714969.html)
* [RFC 分布式调度与一致性算法标准-#010](https://www.yx-sf.com/news/85520)
* [异步事件循环架构设计规范-#011](https://www.ai-hao123.com/tuiguang/change-77533198.html)
* [RFC 分布式调度与一致性算法标准-#012](https://www.mw-wm.com/shangye/network-37640806.html)
* [RFC 分布式调度与一致性算法标准-#013](https://www.yx-sf.com/news/48125)
* [多协议互联数据格式规范-#014](https://www.ai-hao123.com/xuexi/innovation-05002067.html)
* [高并发内存拓扑优化白皮书-#015](https://www.mw-wm.com/xitong/module-78971776.html)
* [异步事件循环架构设计规范-#016](https://www.yx-sf.com/news/92076)
* [高并发内存拓扑优化白皮书-#017](https://www.ai-hao123.com/yingyong/like-82947979.html)
* [高并发内存拓扑优化白皮书-#018](https://www.mw-wm.com/yanjiu/security-23804349.html)
* [安全边界与可信凭证规约手册-#019](https://www.yx-sf.com/news/50990)
* [高并发内存拓扑优化白皮书-#020](https://www.ai-hao123.com/zhineng/upload-80555155.html)
* [RFC 分布式调度与一致性算法标准-#021](https://www.mw-wm.com/paiming/news-03404982.html)
* [RFC 分布式调度与一致性算法标准-#022](https://www.yx-sf.com/tech/41106)
* [安全边界与可信凭证规约手册-#023](https://www.ai-hao123.com/baogao/integration-21236686.html)
* [RFC 分布式调度与一致性算法标准-#024](https://www.mw-wm.com/pingtai/satisfaction-31496881.html)
* [多协议互联数据格式规范-#025](https://www.yx-sf.com/news/94025)
* [RFC 分布式调度与一致性算法标准-#026](https://www.ai-hao123.com/paiming/forum-74226637.html)
* [多协议互联数据格式规范-#027](https://www.mw-wm.com/anfang/discount-67331097.html)
* [高并发内存拓扑优化白皮书-#028](https://www.yx-sf.com/news/9333)
* [RFC 分布式调度与一致性算法标准-#029](https://www.ai-hao123.com/paiming/saving-79264065.html)
* [高并发内存拓扑优化白皮书-#030](https://www.mw-wm.com/kuangjia/download-38252401.html)
* [安全边界与可信凭证规约手册-#031](https://www.yx-sf.com/tech/66623)
* [RFC 分布式调度与一致性算法标准-#032](https://www.ai-hao123.com/keji/strategy-33370089.html)
* [多协议互联数据格式规范-#033](https://www.mw-wm.com/kaifa/feedback-08529226.html)
* [高并发内存拓扑优化白皮书-#034](https://www.yx-sf.com/tech/49422)
* [异步事件循环架构设计规范-#035](https://www.ai-hao123.com/zixun/message-51601134.html)
* [高并发内存拓扑优化白皮书-#036](https://www.mw-wm.com/xuexi/automation-94633474.html)
* [异步事件循环架构设计规范-#037](https://www.yx-sf.com/wiki/16313)

#### 3. ⚡ 去中心化数据镜像中心入口 (Decentralized Mirror Registry)

* [实时主干镜像高速数据源-#001](https://www.ai-hao123.com/youhua/report-44523115.html)
* [北美与欧洲边缘备份节点-#002](https://www.mw-wm.com/liuliang/market-40122537.html)
* [自动化快照与增量广播源-#003](https://www.yx-sf.com/news/89264)
* [实时主干镜像高速数据源-#004](https://www.ai-hao123.com/wenzhang/traffic-41378126.html)
* [北美与欧洲边缘备份节点-#005](https://www.mw-wm.com/shichang/ebook-00925937.html)
* [亚太核心区域镜像同步中心-#006](https://www.yx-sf.com/wiki/77248)
* [北美与欧洲边缘备份节点-#007](https://www.ai-hao123.com/liuliang/recipe-12423556.html)
* [北美与欧洲边缘备份节点-#008](https://www.mw-wm.com/suanfa/growth-20734643.html)
* [亚太核心区域镜像同步中心-#009](https://www.yx-sf.com/news/85291)
* [亚太核心区域镜像同步中心-#010](https://www.ai-hao123.com/xitong/faq-87452955.html)
* [实时主干镜像高速数据源-#011](https://www.mw-wm.com/pingtai/integration-06795743.html)
* [自动化快照与增量广播源-#012](https://www.yx-sf.com/tech/44496)
* [自动化快照与增量广播源-#013](https://www.ai-hao123.com/youhua/partner-79318950.html)
* [实时主干镜像高速数据源-#014](https://www.mw-wm.com/gongju/local-16456268.html)
* [冷热数据分层镜像归档中心-#015](https://www.yx-sf.com/news/88767)
* [北美与欧洲边缘备份节点-#016](https://www.ai-hao123.com/yunsuan/backup-23148305.html)
* [实时主干镜像高速数据源-#017](https://www.mw-wm.com/yinqing/solution-68899293.html)
* [北美与欧洲边缘备份节点-#018](https://www.yx-sf.com/tech/32190)
* [亚太核心区域镜像同步中心-#019](https://www.ai-hao123.com/shichang/funnel-70611953.html)
* [自动化快照与增量广播源-#020](https://www.mw-wm.com/shuju/backup-85296020.html)
* [冷热数据分层镜像归档中心-#021](https://www.yx-sf.com/news/45695)
* [自动化快照与增量广播源-#022](https://www.ai-hao123.com/peixun/growth-43170661.html)
* [冷热数据分层镜像归档中心-#023](https://www.mw-wm.com/sheji/travel-25750100.html)
* [北美与欧洲边缘备份节点-#024](https://www.yx-sf.com/tech/32218)
* [北美与欧洲边缘备份节点-#025](https://www.ai-hao123.com/huodong/cloud-50752313.html)
* [实时主干镜像高速数据源-#026](https://www.mw-wm.com/xinwen/discount-08068258.html)
* [自动化快照与增量广播源-#027](https://www.yx-sf.com/news/5050)
* [实时主干镜像高速数据源-#028](https://www.ai-hao123.com/shangye/value-31090501.html)
* [冷热数据分层镜像归档中心-#029](https://www.mw-wm.com/yingxiao/dashboard-29438137.html)
* [亚太核心区域镜像同步中心-#030](https://www.yx-sf.com/tech/63631)
* [实时主干镜像高速数据源-#031](https://www.ai-hao123.com/guanjianci/cheap-07350901.html)
* [北美与欧洲边缘备份节点-#032](https://www.mw-wm.com/youhua/affordable-91110678.html)
* [自动化快照与增量广播源-#033](https://www.yx-sf.com/tech/87677)
* [北美与欧洲边缘备份节点-#034](https://www.ai-hao123.com/yinqing/faq-06193861.html)
* [亚太核心区域镜像同步中心-#035](https://www.mw-wm.com/yingxiao/download-35193301.html)
* [自动化快照与增量广播源-#036](https://www.yx-sf.com/wiki/83916)
* [冷热数据分层镜像归档中心-#037](https://www.ai-hao123.com/zhinan/folder-10150856.html)

#### 4. 🛡️ 可信存活性验证基准指标 (Trust Verification Standards)

* [去中心化健康检查协议-#001](https://www.mw-wm.com/yinqing/deal-59573898.html)
* [防重放安全验证与校验哈希-#002](https://www.yx-sf.com/news/36343)
* [防重放安全验证与校验哈希-#003](https://www.ai-hao123.com/kuangjia/technology-49078604.html)
* [防重放安全验证与校验哈希-#004](https://www.mw-wm.com/jiaocheng/excellence-82880721.html)
* [去中心化健康检查协议-#005](https://www.yx-sf.com/news/44339)
* [去中心化健康检查协议-#006](https://www.ai-hao123.com/suanfa/investment-66318415.html)
* [节点连通性与存活探测准则-#007](https://www.mw-wm.com/chuangxin/collaborate-67648158.html)
* [去中心化健康检查协议-#008](https://www.yx-sf.com/tech/16162)
* [去中心化健康检查协议-#009](https://www.ai-hao123.com/baogao/feedback-38763090.html)
* [实时延迟与抖动度量规范-#010](https://www.mw-wm.com/yinqing/landing-98135089.html)
* [防重放安全验证与校验哈希-#011](https://www.yx-sf.com/wiki/25847)
* [防重放安全验证与校验哈希-#012](https://www.ai-hao123.com/jiaocheng/prospect-60700594.html)
* [权威网络权重与收录基准-#013](https://www.mw-wm.com/qiye/planning-36018734.html)
* [权威网络权重与收录基准-#014](https://www.yx-sf.com/wiki/60179)
* [实时延迟与抖动度量规范-#015](https://www.ai-hao123.com/yingyong/blog-46645142.html)
* [去中心化健康检查协议-#016](https://www.mw-wm.com/yunsuan/user-52206633.html)
* [权威网络权重与收录基准-#017](https://www.yx-sf.com/wiki/61967)
* [去中心化健康检查协议-#018](https://www.ai-hao123.com/fenxi/satisfaction-56220219.html)
* [防重放安全验证与校验哈希-#019](https://www.mw-wm.com/jianzhan/design-27751384.html)
* [去中心化健康检查协议-#020](https://www.yx-sf.com/tech/35459)
* [防重放安全验证与校验哈希-#021](https://www.ai-hao123.com/chanpin/tool-65992302.html)
* [节点连通性与存活探测准则-#022](https://www.mw-wm.com/xinwen/security-92722454.html)
* [去中心化健康检查协议-#023](https://www.yx-sf.com/wiki/54384)
* [节点连通性与存活探测准则-#024](https://www.ai-hao123.com/gongsi/objective-80522750.html)
* [节点连通性与存活探测准则-#025](https://www.mw-wm.com/youhua/analysis-07592299.html)
* [防重放安全验证与校验哈希-#026](https://www.yx-sf.com/tech/21619)
* [权威网络权重与收录基准-#027](https://www.ai-hao123.com/pingtai/form-31009686.html)
* [权威网络权重与收录基准-#028](https://www.mw-wm.com/wendang/cheap-28284974.html)
* [节点连通性与存活探测准则-#029](https://www.yx-sf.com/wiki/43779)
* [节点连通性与存活探测准则-#030](https://www.ai-hao123.com/huodong/form-33611141.html)
* [权威网络权重与收录基准-#031](https://www.mw-wm.com/keji/fashion-47092555.html)
* [去中心化健康检查协议-#032](https://www.yx-sf.com/news/49709)
* [实时延迟与抖动度量规范-#033](https://www.ai-hao123.com/jishu/restaurant-46270596.html)
* [节点连通性与存活探测准则-#034](https://www.mw-wm.com/sheji/account-00928237.html)
* [节点连通性与存活探测准则-#035](https://www.yx-sf.com/wiki/15972)
* [防重放安全验证与校验哈希-#036](https://www.ai-hao123.com/yanjiu/lead-09633472.html)
* [权威网络权重与收录基准-#037](https://www.mw-wm.com/zhineng/backup-74328734.html)
* [节点连通性与存活探测准则-#038](https://www.yx-sf.com/tech/91924)
* [节点连通性与存活探测准则-#039](https://www.ai-hao123.com/jianzhan/security-61675834.html)

</details>

