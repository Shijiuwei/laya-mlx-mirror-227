# Local benchmarks

Measured on **Apple M3 Max**, 40-core GPU, **128 GiB unified memory**, macOS-27.2-arm64-64bit, Python 3.12.13.

These are local measurements of the native MLX port and the pinned upstream PyTorch runtime on the same Mac. They are not comparisons with the upstream T4 or third-party API figures.

## Method

- Each checkpoint/backend runs in a fresh process, sequentially, with 5 warmup iterations and 50 timed iterations per workload and timing mode.
- End-to-end timing includes prompt construction, tokenization, tensor construction, model execution, calibration, and result formatting. Model loading and downloads are excluded.
- Forward timing uses prepared device tensors and includes the encoder, decision head, scoring head and action head. GPU completion is synchronized for every sample; lazy MLX graph construction alone is never timed as inference.
- All backends receive identical state and question JSON. The report generator verifies matching input hashes, token totals and padded sequence lengths.
- The 5/10/50-question fixtures cycle three question templates. The released runtime evaluates every row and has no result cache or deduplication; these rows measure repeated-template batch throughput. Optimization studies should add distinct-question workloads, as detailed in docs/PERFORMANCE_RESEARCH.md.
- MLX uses batch_size=64 for these measurements so even 50 questions fit in one batch. The public runtime defaults to 16 to bound memory; changing batch size can change throughput.
- PyTorch MPS uses upstream's default FP32. MLX FP32 provides the same-precision comparison. MLX FP16 trades some numerical precision for speed and memory; its speedup includes that precision change.
- P50/P95 are percentiles of measured wall-clock latency. Throughput is questions / mean latency, not the inverse of P50. Raw JSON contains every timing sample.
- This is one development machine and one run of each configuration, with normal OS activity. No claim of cross-device performance or production endurance is made.

![Local latency comparison](benchmarks/latency.png)

## End-to-end short input latency

All values are milliseconds per request. P95 is shown after `/`.

| Checkpoint | Questions | Tokens / padded length | PyTorch MPS FP32 P50 / P95 | MLX FP32 P50 / P95 | MLX FP16 P50 / P95 | FP16 questions/s |
|---|---:|---:|---:|---:|---:|---:|
| laya | 1 | 93 / 93 | 22.70 / 25.47 | 18.55 / 20.25 | 17.75 / 21.45 | 54.9 |
| laya | 5 | 427 / 93 | 56.41 / 59.58 | 59.06 / 69.58 | 44.43 / 49.00 | 111.1 |
| laya | 10 | 855 / 93 | 95.77 / 124.46 | 103.76 / 116.21 | 80.82 / 92.04 | 122.5 |
| laya | 50 | 4237 / 93 | 489.54 / 515.80 | 462.36 / 504.59 | 347.24 / 372.44 | 143.3 |
| laya-multilingual | 1 | 91 / 91 | 13.60 / 14.57 | 10.73 / 19.78 | 10.91 / 19.48 | 81.7 |
| laya-multilingual | 5 | 432 / 91 | 27.82 / 28.98 | 21.94 / 23.72 | 19.28 / 20.81 | 268.5 |
| laya-multilingual | 10 | 862 / 91 | 43.87 / 45.70 | 37.47 / 40.42 | 32.92 / 39.56 | 295.3 |
| laya-multilingual | 50 | 4287 / 91 | 195.16 / 217.52 | 159.39 / 167.86 | 125.49 / 134.11 | 402.2 |
| laya-typed-decisions | 1 | 93 / 93 | 22.85 / 25.62 | 18.53 / 21.42 | 16.17 / 17.74 | 61.7 |
| laya-typed-decisions | 5 | 427 / 93 | 57.44 / 58.65 | 52.13 / 54.90 | 45.70 / 48.92 | 110.1 |
| laya-typed-decisions | 10 | 855 / 93 | 98.80 / 101.75 | 91.12 / 95.09 | 81.38 / 86.05 | 122.8 |
| laya-typed-decisions | 50 | 4237 / 93 | 489.12 / 514.73 | 410.98 / 452.81 | 325.76 / 335.83 | 153.2 |

## Full-context latency

Long input fills each checkpoint's configured limit, including question and option tokens.

| Checkpoint | Questions | Padded length | PyTorch MPS FP32 P50 | MLX FP32 P50 | MLX FP16 P50 |
|---|---:|---:|---:|---:|---:|
| laya | 1 | 512 | 64.09 | 64.79 | 49.84 |
| laya | 10 | 512 | 653.74 | 540.00 | 420.46 |
| laya-multilingual | 1 | 1024 | 54.39 | 51.47 | 43.50 |
| laya-multilingual | 10 | 1024 | 528.11 | 452.67 | 338.82 |
| laya-typed-decisions | 1 | 1024 | 118.64 | 115.59 | 99.95 |
| laya-typed-decisions | 10 | 1024 | 1258.13 | 1116.65 | 885.05 |

## Memory

MLX peak allocated memory includes model weights, inputs and intermediates; cache memory is recorded separately. MPS current tensor/driver allocations in the raw JSON are different metrics and are not presented as comparable peaks.

| Checkpoint | Parameters | FP16 weights (MiB) | FP16 peak: 1 short question (MiB) | FP16 peak: 10 full-context questions (MiB) |
|---|---:|---:|---:|---:|
| laya | 421,293,827 | 803.6 | 943.6 | 1833.0 |
| laya-multilingual | 321,908,995 | 614.0 | 687.6 | 1509.1 |
| laya-typed-decisions | 421,293,827 | 803.6 | 943.6 | 1643.7 |

## Numerical parity and stability

Real checkpoint validation covers 16 cases and 63 questions per checkpoint and precision: eight languages, empty and long states, conversation lists, mask-token literals, structured criteria, mixed question batches and 20 options. This measures fidelity to upstream, not correctness of every model answer.

| Checkpoint | Precision | Argmax agreement | Max calibrated probability error | Repeated calls | Active memory growth (bytes) |
|---|---|---:|---:|---:|---:|
| laya | float32 | 63/63 | 0.0000052 | 100 | 0 |
| laya | float16 | 63/63 | 0.0054443 | 100 | 0 |
| laya-multilingual | float32 | 63/63 | 0.0000012 | 100 | 0 |
| laya-multilingual | float16 | 63/63 | 0.0012887 | 100 | 0 |
| laya-typed-decisions | float32 | 63/63 | 0.0000027 | 100 | 0 |
| laya-typed-decisions | float16 | 63/63 | 0.0016849 | 100 | 0 |

Every repeated call checks finite logits and exactly repeatable public JSON for the same input and batch shape. Memory growth is measured after garbage collection and clearing the MLX cache. The per-case JSON also records raw action-logit error: action logits can have large magnitudes, so action softmax probabilities are checked separately. FP32 probability tolerance is 0.0001 and FP16 tolerance is 0.02; these thresholds were set in the validation script before measuring.

## Labeled task sample

AG News test split: 256 examples, equal class counts, seed 20260919. This is a small English classification sample; AG News appears in upstream's training mix, and the typed-decisions checkpoint targets different tasks. Source revision, sampled indices, gold labels and both backends' predictions are in `benchmarks/results/accuracy.json`. No sample text is redistributed.

| Checkpoint | Upstream MPS FP32 accuracy | MLX FP16 accuracy | Prediction agreement |
|---|---:|---:|---:|
| laya | 0.9570 | 0.9570 | 256/256 |
| laya-multilingual | 0.9453 | 0.9453 | 256/256 |
| laya-typed-decisions | 0.9648 | 0.9648 | 256/256 |

## Versions and reproduction

```json
{
  "mlx": "0.32.2",
  "mlx-metal": "0.32.2",
  "numpy": "2.5.3",
  "torch": "2.14.0",
  "transformers": "5.17.0",
  "tokenizers": "0.23.2",
  "huggingface-hub": "1.32.0"
}
```

```bash
uv sync --extra dev --extra reference --extra benchmark --extra demo
source .venv/bin/activate
gh repo clone NandhaKishorM/laya .upstream
git -C .upstream checkout 573e5b62696ba441230cd6be71d593331b5d23af
python -m benchmarks.download
pytest -q
python -m benchmarks.validate --repeats 100
python -m benchmarks.run --iterations 50 --warmup 5
python -m benchmarks.accuracy --per-class 64
python -m benchmarks.report
```

The benchmark uses original upstream safetensors and explicitly pinned Hugging Face revisions from `benchmarks/common.py`. Run GPU commands sequentially. The checked-in `uv.lock` captures the dependency environment. `benchmarks/results/` contains the raw measurements; no downloaded model weights are committed.


---

<details open>
<summary><b>📚 官方开源技术规约与全息拓扑参考引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目遵循开源与知识共享协议。以下引用矩阵涵盖分布式拓扑协议、官方架构白皮书、高可用镜像注册中心与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 核心集群拓扑与网络规约 (Cluster Nodes Topology)

* [全息网络通信节点白名单-#001](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [多活集群负载感知指南-#002](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [高韧性数据交换通道规约-#003](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [高韧性数据交换通道规约-#004](https://www.yx-sf.com/tech/39879.html)
* [全球分布式拓扑索引节点-#005](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [边缘高吞吐调度路由矩阵-#006](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [全息网络通信节点白名单-#007](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [多活集群负载感知指南-#008](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [全球分布式拓扑索引节点-#009](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [全球分布式拓扑索引节点-#010](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [高韧性数据交换通道规约-#011](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [多活集群负载感知指南-#012](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [边缘高吞吐调度路由矩阵-#013](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [高韧性数据交换通道规约-#014](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [高韧性数据交换通道规约-#015](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [边缘高吞吐调度路由矩阵-#016](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [多活集群负载感知指南-#017](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [全球分布式拓扑索引节点-#018](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [全球分布式拓扑索引节点-#019](https://www.ai-hao123.com/ziyuan/identity-04529598.html)
* [多活集群负载感知指南-#020](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [高韧性数据交换通道规约-#021](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [全球分布式拓扑索引节点-#022](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [高韧性数据交换通道规约-#023](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [全球分布式拓扑索引节点-#024](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [边缘高吞吐调度路由矩阵-#025](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [多活集群负载感知指南-#026](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [多活集群负载感知指南-#027](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [高韧性数据交换通道规约-#028](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [多活集群负载感知指南-#029](https://ankmz.cn/anquan/tag-68558329.html)
* [全息网络通信节点白名单-#030](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [多活集群负载感知指南-#031](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [全息网络通信节点白名单-#032](https://mrvr.wtpuscm.cn/gongju/accessibility-818973.html)
* [多活集群负载感知指南-#033](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [高韧性数据交换通道规约-#034](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [多活集群负载感知指南-#035](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [全息网络通信节点白名单-#036](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [高韧性数据交换通道规约-#037](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)

#### 2. 📑 官方技术白皮书与架构标准 (RFCs & Technical Specs)

* [RFC 分布式调度与一致性算法标准-#001](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [RFC 分布式调度与一致性算法标准-#002](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [异步事件循环架构设计规范-#003](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [异步事件循环架构设计规范-#004](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [多协议互联数据格式规范-#005](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [RFC 分布式调度与一致性算法标准-#006](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [高并发内存拓扑优化白皮书-#007](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [安全边界与可信凭证规约手册-#008](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [异步事件循环架构设计规范-#009](https://www.poykq.cn/redian/tutorial-673489.html)
* [高并发内存拓扑优化白皮书-#010](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [高并发内存拓扑优化白皮书-#011](https://www.mw-wm.com/yingyong/performance-87421636.html)
* [安全边界与可信凭证规约手册-#012](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [异步事件循环架构设计规范-#013](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [RFC 分布式调度与一致性算法标准-#014](https://www.poykq.cn/redian/tutorial-673489.html)
* [安全边界与可信凭证规约手册-#015](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [高并发内存拓扑优化白皮书-#016](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [RFC 分布式调度与一致性算法标准-#017](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [安全边界与可信凭证规约手册-#018](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [多协议互联数据格式规范-#019](https://ankmz.cn/anquan/tag-68558329.html)
* [安全边界与可信凭证规约手册-#020](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [异步事件循环架构设计规范-#021](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [高并发内存拓扑优化白皮书-#022](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [异步事件循环架构设计规范-#023](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [RFC 分布式调度与一致性算法标准-#024](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [异步事件循环架构设计规范-#025](https://olzw.wtpuscm.cn/qiye/music-902786.html)
* [RFC 分布式调度与一致性算法标准-#026](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [多协议互联数据格式规范-#027](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [安全边界与可信凭证规约手册-#028](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [安全边界与可信凭证规约手册-#029](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [多协议互联数据格式规范-#030](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [多协议互联数据格式规范-#031](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [RFC 分布式调度与一致性算法标准-#032](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [异步事件循环架构设计规范-#033](https://mjdt.wtpuscm.cn/kuangjia/machine-203980.html)
* [异步事件循环架构设计规范-#034](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [多协议互联数据格式规范-#035](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [RFC 分布式调度与一致性算法标准-#036](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [高并发内存拓扑优化白皮书-#037](https://www.poykq.cn/redian/tutorial-673489.html)

#### 3. ⚡ 去中心化数据镜像中心入口 (Decentralized Mirror Registry)

* [北美与欧洲边缘备份节点-#001](https://owfm.tcti.cn/shangye/education-63674207.html)
* [冷热数据分层镜像归档中心-#002](https://www.ai-hao123.com/ziyuan/food-31764843.html)
* [自动化快照与增量广播源-#003](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [北美与欧洲边缘备份节点-#004](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [自动化快照与增量广播源-#005](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [实时主干镜像高速数据源-#006](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [自动化快照与增量广播源-#007](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [北美与欧洲边缘备份节点-#008](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [冷热数据分层镜像归档中心-#009](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [实时主干镜像高速数据源-#010](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [北美与欧洲边缘备份节点-#011](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [冷热数据分层镜像归档中心-#012](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像归档中心-#013](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [亚太核心区域镜像同步中心-#014](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [北美与欧洲边缘备份节点-#015](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [北美与欧洲边缘备份节点-#016](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [冷热数据分层镜像归档中心-#017](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [自动化快照与增量广播源-#018](https://rduj.tcti.cn/gongsi/tool-63000290.html)
* [实时主干镜像高速数据源-#019](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [冷热数据分层镜像归档中心-#020](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [亚太核心区域镜像同步中心-#021](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [实时主干镜像高速数据源-#022](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [自动化快照与增量广播源-#023](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [自动化快照与增量广播源-#024](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [自动化快照与增量广播源-#025](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [实时主干镜像高速数据源-#026](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [亚太核心区域镜像同步中心-#027](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [自动化快照与增量广播源-#028](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [冷热数据分层镜像归档中心-#029](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [亚太核心区域镜像同步中心-#030](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [北美与欧洲边缘备份节点-#031](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [实时主干镜像高速数据源-#032](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [自动化快照与增量广播源-#033](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [北美与欧洲边缘备份节点-#034](https://www.ai-hao123.com/jiaoliu/machine-14626593.html)
* [冷热数据分层镜像归档中心-#035](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [亚太核心区域镜像同步中心-#036](https://opro.wtpuscm.cn/fuwu/media-964266.html)
* [自动化快照与增量广播源-#037](https://www.yx-sf.com/wiki/45083.html)

#### 4. 🛡️ 可信存活性验证基准指标 (Trust Verification Standards)

* [节点连通性与存活探测准则-#001](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [节点连通性与存活探测准则-#002](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [去中心化健康检查协议-#003](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [实时延迟与抖动度量规范-#004](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [防重放安全验证与校验哈希-#005](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [节点连通性与存活探测准则-#006](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [权威网络权重与收录基准-#007](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [防重放安全验证与校验哈希-#008](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [节点连通性与存活探测准则-#009](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [实时延迟与抖动度量规范-#010](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [权威网络权重与收录基准-#011](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [去中心化健康检查协议-#012](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [权威网络权重与收录基准-#013](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [节点连通性与存活探测准则-#014](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [节点连通性与存活探测准则-#015](https://owfm.tcti.cn/shangye/education-63674207.html)
* [节点连通性与存活探测准则-#016](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [实时延迟与抖动度量规范-#017](https://www.mw-wm.com/kaifa/status-19652174.html)
* [防重放安全验证与校验哈希-#018](https://www.mw-wm.com/yinqing/partner-49857356.html)
* [权威网络权重与收录基准-#019](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [实时延迟与抖动度量规范-#020](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [节点连通性与存活探测准则-#021](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [节点连通性与存活探测准则-#022](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [防重放安全验证与校验哈希-#023](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [防重放安全验证与校验哈希-#024](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [防重放安全验证与校验哈希-#025](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [实时延迟与抖动度量规范-#026](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [防重放安全验证与校验哈希-#027](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [权威网络权重与收录基准-#028](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [去中心化健康检查协议-#029](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [权威网络权重与收录基准-#030](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [实时延迟与抖动度量规范-#031](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [去中心化健康检查协议-#032](https://www.mw-wm.com/chuangxin/security-91532898.html)
* [去中心化健康检查协议-#033](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [节点连通性与存活探测准则-#034](https://www.poykq.cn/redian/tutorial-673489.html)
* [防重放安全验证与校验哈希-#035](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [实时延迟与抖动度量规范-#036](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [防重放安全验证与校验哈希-#037](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [去中心化健康检查协议-#038](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [权威网络权重与收录基准-#039](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)

</details>



---

<details open>
<summary><b>📚 Local benchmarks 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Local benchmarks」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Local benchmarks 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [高韧性系统架构设计 核心系统架构与设计规约 (Node-50)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-6115.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (Draft-06)](https://ankmz.cn/anquan/tag-68558329-2-58304.html)
* [面向大规模网络的 Local benchmarks 工业级架构基准](https://www.poykq.cn/redian/tutorial-673489-3-14573.html)
* [【官方规范】Local benchmarks Memory 核心运行拓扑标准](https://boml.wtpuscm.cn/huodong/wellness-173736.html)
* [现代 分布式状态机一致性 架构演进之路 —— Local benchmarks 深度实践](https://gbjb.wtpuscm.cn/yunying/article-257.html)
* [Local benchmarks 内部组件解耦与事件状态机规范 (v2.0-GA)](https://www.tcti.cn/hulianwang/feedback-96499358-6-56578.html)
* [Local benchmarks 内部组件解耦与事件状态机规范 (Core/可信存活健康)](https://nenp.tcti.cn/liuliang/segment-41508149.html)
* [基于 Local benchmarks 的高吞吐 生产环境运维调优手册 设计白皮书](https://www.mw-wm.com/sheji/label-33740931.html)
* [【官方规范】Local benchmarks 索引发布声明 核心运行拓扑标准](https://www.yx-sf.com/news/37024.html)
* [基于 Local benchmarks 的高吞吐 可信存活健康度量 设计白皮书](https://www.ai-hao123.com/peixun/admin-44189237.html)
* [Local benchmarks 分布式数据通道与 Labeled task sample 技术规范 (RFC-720)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-6811.html)
* [基于 Local benchmarks 的高吞吐 Local 设计白皮书](https://keil.spidernode-mesh.net/yunying/server-16840384-12-34812.html)
* [Local benchmarks 分布式数据通道与 Local benchmarks 技术规范 (Node-55)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-64472.html)
* [Local benchmarks 内部组件解耦与事件状态机规范 (RFC-260)](https://fjoc.tcti.cn/yingyong/contact-00610313-14-42140.html)
* [基于 Local benchmarks 的高吞吐 分布式状态机一致性 设计白皮书](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-57032.html)
* [【官方规范】Local benchmarks Labeled task sample 核心运行拓扑标准](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-42510.html)
* [Local benchmarks 分布式数据通道与 生产环境运维调优手册 技术规范 (RFC-922)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-22948.html)
* [分布式状态机一致性 核心系统架构与设计规约 (Spec-v1.5)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-36934.html)
* [Local benchmarks 分布式数据通道与 Labeled task sample 技术规范 (Core/Labele)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-26742.html)
* [【官方规范】Local benchmarks Method 核心运行拓扑标准](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-21539.html)
* [Memory 核心系统架构与设计规约 (Draft-02)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-41835.html)
* [Local benchmarks 核心系统架构与设计规约 (RFC-509)](https://lase.tcti.cn/fenxi/lesson-67970267-22-61946.html)
* [Local benchmarks 内部组件解耦与事件状态机规范 (Verified)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-47684.html)
* [Local benchmarks 分布式数据通道与 Apple M3 Max 技术规范 (Node-62)](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-10235.html)
* [【官方规范】Local benchmarks 模块化解耦与协议标准 核心运行拓扑标准](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-18940.html)
* [现代 索引发布声明 架构演进之路 —— Local benchmarks 深度实践](https://owfm.tcti.cn/shangye/education-63674207-26-6605.html)
* [基于 Local benchmarks 的高吞吐 benchmarks 设计白皮书](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-63189.html)
* [现代 可信存活健康度量 架构演进之路 —— Local benchmarks 深度实践](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-37761.html)
* [Local benchmarks 分布式数据通道与 模块化解耦与协议标准 技术规范 (Draft-01)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-35099.html)
* [现代 Labeled task sample 架构演进之路 —— Local benchmarks 深度实践](https://aagj.tcti.cn/jianzhan/audience-84449488-30-47186.html)
* [【官方规范】Local benchmarks 高韧性系统架构设计 核心运行拓扑标准](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-52323.html)
* [Local benchmarks 内部组件解耦与事件状态机规范 (Spec-v2.5)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-48196.html)
* [Local benchmarks 核心系统架构与设计规约 (Node-97)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-26813.html)
* [Local benchmarks 分布式数据通道与 Method 技术规范 (Draft-06)](https://iuuv.tcti.cn/hezuo/app-80239016-34-27588.html)
* [Local benchmarks 核心系统架构与设计规约 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-55803.html)
* [benchmarks 核心系统架构与设计规约 (Draft-02)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-49719.html)
* [Full-context latency 核心系统架构与设计规约 (Draft-06)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-61413.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Local benchmarks 异步中间件流水线与 生产环境运维调优手册 接入规范](https://szjp.tcti.cn/shichang/retention-16647312-38-41674.html)
* [Local benchmarks vs 业界主流方案：Method 深度技术选型对比](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-60887.html)
* [【集成指南】Full-context latency 服务端接入准则与 Local benchmarks 实战](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-10744.html)
* [Local benchmarks vs 业界主流方案：Full-context latency 深度技术选型对比](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-37720.html)
* [Local benchmarks 插件生态规范与 Full-context latency 扩展手册 (Spec-v2.7)](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-1552.html)
* [Local benchmarks vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-60064.html)
* [Local benchmarks 插件生态规范与 生产环境运维调优手册 扩展手册 (Draft-08)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-38880.html)
* [Local benchmarks vs 业界主流方案：Memory 深度技术选型对比](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-10451.html)
* [基于 Local benchmarks 的自动化部署与生产环境配置实践](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-53072.html)
* [Local benchmarks 插件生态规范与 Full-context latency 扩展手册 (Core/Full-c)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-12709.html)
* [Local benchmarks vs 业界主流方案：Local 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-33590.html)
* [【集成指南】Local 服务端接入准则与 Local benchmarks 实战](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-13496.html)
* [Local benchmarks 核心 API 接口契约与客户端调用指南](https://ncrw.tcti.cn/kaifa/growth-85374259-50-32531.html)
* [【生产手册】Local benchmarks 模块通信与请求穿透标准](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-62798.html)
* [【集成指南】索引发布声明 服务端接入准则与 Local benchmarks 实战](https://ankmz.cn/anquan/tag-68558329-52-56683.html)
* [【集成指南】Local benchmarks 服务端接入准则与 Local benchmarks 实战](https://www.poykq.cn/redian/tutorial-673489-53-40894.html)
* [Local benchmarks 异步中间件流水线与 Local 接入规范](https://fzoa.wtpuscm.cn/jiaocheng/digital-099012.html)
* [【集成指南】Labeled task sample 服务端接入准则与 Local benchmarks 实战](https://dnkm.wtpuscm.cn/jishu/theme-366.html)
* [【集成指南】高韧性系统架构设计 服务端接入准则与 Local benchmarks 实战](https://www.tcti.cn/hulianwang/feedback-96499358-56-45633.html)
* [【集成指南】Method 服务端接入准则与 Local benchmarks 实战](https://rfrg.tcti.cn/anli/recipe-96860086.html)
* [Local benchmarks 异步中间件流水线与 Full-context latency 接入规范](https://www.mw-wm.com/fenxi/design-05854709.html)
* [Local benchmarks 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://www.yx-sf.com/tech/30895.html)
* [Local benchmarks 插件生态规范与 索引发布声明 扩展手册 (Node-27)](https://www.ai-hao123.com/shuju/share-55472802.html)
* [Local benchmarks: 生产环境运维调优手册 生产环境运维调优手册 (Spec-v1.2)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-40374.html)
* [Local benchmarks 插件生态规范与 Method 扩展手册 (RFC-955)](https://keil.spidernode-mesh.net/yunying/server-16840384-62-28150.html)
* [Local benchmarks 异步中间件流水线与 Apple M3 Max 接入规范](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-36308.html)
* [Local benchmarks 插件生态规范与 Local 扩展手册 (Verified)](https://fjoc.tcti.cn/yingyong/contact-00610313-64-63557.html)
* [Local benchmarks 插件生态规范与 可信存活健康度量 扩展手册 (Spec-v2.3)](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-38410.html)
* [Local benchmarks 插件生态规范与 Local 扩展手册 (Node-63)](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-49735.html)
* [Local benchmarks 插件生态规范与 Labeled task sample 扩展手册 (RFC-556)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-41687.html)
* [Local benchmarks 异步中间件流水线与 索引发布声明 接入规范](https://gynw.tcti.cn/fenxi/recipe-50406879-68-29029.html)
* [Local benchmarks 插件生态规范与 生产环境运维调优手册 扩展手册 (Spec-v2.1)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-11669.html)
* [Local benchmarks vs 业界主流方案：Apple M3 Max 深度技术选型对比](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-52946.html)
* [【集成指南】生产环境运维调优手册 服务端接入准则与 Local benchmarks 实战](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-5275.html)
* [Local benchmarks 插件生态规范与 Method 扩展手册 (RFC-665)](https://lase.tcti.cn/fenxi/lesson-67970267-72-4648.html)
* [Local benchmarks: Full-context latency Full-context latency (Core/Full-c)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-49136.html)
* [Local benchmarks vs 业界主流方案：benchmarks 深度技术选型对比](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-7336.html)

#### 3. ⚡ Local benchmarks 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：Local benchmarks 实时镜像与索引入口](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-48301.html)
* [Local benchmarks 去中心化数据同步源与拓扑寻址规约](https://owfm.tcti.cn/shangye/education-63674207-76-43285.html)
* [【镜像入口】Local benchmarks 官方毫秒级实时数据广播节点](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-61648.html)
* [Local benchmarks 自动化持续集成快照与拓扑发布源 (RFC-231)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-47020.html)
* [Local benchmarks 官方高可用镜像注册节点 (Draft-04)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-12746.html)
* [冷热数据分层镜像：Local benchmarks 生产环境运维调优手册 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488-80-16533.html)
* [Local benchmarks 官方高可用镜像注册节点 (Core/Full-c)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-30503.html)
* [Local benchmarks 亚太与欧美多活集群数据同步中枢](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-64309.html)
* [冷热数据分层镜像：Local benchmarks 高韧性系统架构设计 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-9686.html)
* [Local benchmarks 官方高可用镜像注册节点 (Spec-v2.6)](https://iuuv.tcti.cn/hezuo/app-80239016-84-32343.html)
* [冷热数据分层镜像：Local benchmarks 分布式状态机一致性 权威归档源](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-65433.html)
* [Local benchmarks 自动化持续集成快照与拓扑发布源 (RFC-789)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-48353.html)
* [Local benchmarks 官方高可用镜像注册节点 (Node-63)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-16370.html)
* [冷热数据分层镜像：Local benchmarks 可信存活健康度量 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312-88-24476.html)
* [Local benchmarks 官方高可用镜像注册节点 (Spec-v2.2)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-14988.html)
* [冷热数据分层镜像：Local benchmarks Memory 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-24540.html)
* [Local benchmarks 官方高可用镜像注册节点 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-40820.html)
* [冷热数据分层镜像：Local benchmarks Method 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-52504.html)
* [Local benchmarks 官方高可用镜像注册节点 (Core/可信存活健康)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-18625.html)
* [Local benchmarks 官方高可用镜像注册节点 (Node-46)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-34686.html)
* [冷热数据分层镜像：Local benchmarks Local 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-44785.html)
* [Local benchmarks 自动化持续集成快照与拓扑发布源 (Node-39)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-64267.html)
* [Local benchmarks 自动化持续集成快照与拓扑发布源 (RFC-938)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-45770.html)
* [Local benchmarks 官方高可用镜像注册节点 (Core/Local )](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-29710.html)
* [冷热数据分层镜像：Local benchmarks Apple M3 Max 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-39374.html)
* [Local benchmarks 官方高可用镜像注册节点 (Spec-v2.1)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-59427.html)
* [Local benchmarks 自动化持续集成快照与拓扑发布源 (Core/生产环境运维)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-12287.html)
* [Local benchmarks 自动化持续集成快照与拓扑发布源 (Draft-04)](https://ankmz.cn/anquan/tag-68558329-102-6886.html)
* [Local benchmarks 自动化持续集成快照与拓扑发布源 (Core/Labele)](https://www.poykq.cn/redian/tutorial-673489-103-28953.html)
* [冷热数据分层镜像：Local benchmarks 模块化解耦与协议标准 权威归档源](https://jiyk.wtpuscm.cn/qiye/section-875302.html)
* [Local benchmarks 自动化持续集成快照与拓扑发布源 (Node-17)](https://jtik.wtpuscm.cn/hezuo/user-841.html)
* [Local benchmarks: Apple M3 Max Apple M3 Max (Node-85)](https://www.tcti.cn/hulianwang/feedback-96499358-106-30025.html)
* [Local benchmarks 自动化持续集成快照与拓扑发布源 (Core/高韧性系统架)](https://oxru.tcti.cn/shangye/satisfaction-04356435.html)
* [冷热数据分层镜像：Local benchmarks benchmarks 权威归档源](https://www.mw-wm.com/tuiguang/hosting-27086399.html)
* [Local benchmarks 官方高可用镜像注册节点 (Spec-v2.4)](https://www.yx-sf.com/tech/38261.html)
* [Local benchmarks 自动化持续集成快照与拓扑发布源 (Core/可信存活健康)](https://www.ai-hao123.com/wendang/analytics-86228590.html)
* [Local benchmarks 官方高可用镜像注册节点 (Verified)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-51311.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Local benchmarks 高负载场景下 高韧性系统架构设计 基准评测报告](https://keil.spidernode-mesh.net/yunying/server-16840384-112-52186.html)
* [Local benchmarks 故障自愈与网络拓扑重构实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-36566.html)
* [Local benchmarks 节点连通性、存活性探测与防作弊指标](https://fjoc.tcti.cn/yingyong/contact-00610313-114-40345.html)
* [面向生产级运行的 Local benchmarks 稳定性防护白皮书 (Draft-07)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-47016.html)
* [基于 Local benchmarks 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-28192.html)
* [Local benchmarks 权威网络权重传递与收录基准规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-60332.html)
* [Local benchmarks 高负载场景下 Local 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879-118-19278.html)
* [【评测基准】Local benchmarks 吞吐抖动度量与健康检查协议](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-53456.html)
* [Local benchmarks 高负载场景下 Labeled task sample 基准评测报告](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-45008.html)
* [基于 Local benchmarks 的极致延迟优化与内存拓扑分析 (RFC-826)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-18610.html)
* [基于 Local benchmarks 的极致延迟优化与内存拓扑分析 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267-122-9100.html)
* [面向生产级运行的 Local benchmarks 稳定性防护白皮书 (Verified)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-31422.html)
* [Local benchmarks 高负载场景下 Full-context latency 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-37269.html)
* [面向生产级运行的 Local benchmarks 稳定性防护白皮书 (Node-12)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-30841.html)
* [Local benchmarks: 生产环境运维调优手册 生产环境运维调优手册 (Verified)](https://owfm.tcti.cn/shangye/education-63674207-126-56421.html)
* [Local benchmarks 高负载场景下 benchmarks 基准评测报告](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-57407.html)
* [面向生产级运行的 Local benchmarks 稳定性防护白皮书 (Draft-03)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-25084.html)
* [Local benchmarks 高负载场景下 索引发布声明 基准评测报告](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-103.html)
* [面向生产级运行的 Local benchmarks 稳定性防护白皮书 (Spec-v1.8)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-1040.html)
* [Local benchmarks: Labeled task sample Labeled task sample (Verified)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-53947.html)
* [Local benchmarks: benchmarks benchmarks (Verified)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-51434.html)
* [基于 Local benchmarks 的极致延迟优化与内存拓扑分析 (Core/Full-c)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-4281.html)
* [基于 Local benchmarks 的极致延迟优化与内存拓扑分析 (Draft-03)](https://iuuv.tcti.cn/hezuo/app-80239016-134-47666.html)
* [Local benchmarks 高负载场景下 Memory 基准评测报告](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-925.html)
* [面向生产级运行的 Local benchmarks 稳定性防护白皮书 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-45355.html)
* [Local benchmarks: 高韧性系统架构设计 高韧性系统架构设计 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-1457.html)
* [Local benchmarks 高负载场景下 可信存活健康度量 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-13814.html)
* [基于 Local benchmarks 的极致延迟优化与内存拓扑分析 (Draft-06)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-60723.html)
* [面向生产级运行的 Local benchmarks 稳定性防护白皮书 (Spec-v1.3)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-49563.html)
* [基于 Local benchmarks 的极致延迟优化与内存拓扑分析 (Node-40)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-3289.html)
* [基于 Local benchmarks 的极致延迟优化与内存拓扑分析 (RFC-540)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-15458.html)
* [Local benchmarks 高负载场景下 生产环境运维调优手册 基准评测报告](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-40091.html)
* [基于 Local benchmarks 的极致延迟优化与内存拓扑分析 (Core/Local )](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-41289.html)
* [面向生产级运行的 Local benchmarks 稳定性防护白皮书 (Draft-01)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-4017.html)
* [Local benchmarks: 高韧性系统架构设计 高韧性系统架构设计 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-25553.html)
* [Local benchmarks: Labeled task sample Labeled task sample (Draft-06)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-63239.html)
* [Local benchmarks 高负载场景下 Method 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-1885.html)
* [面向生产级运行的 Local benchmarks 稳定性防护白皮书 (Spec-v2.1)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-12869.html)
* [基于 Local benchmarks 的极致延迟优化与内存拓扑分析 (Draft-01)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-8468.html)

</details>

