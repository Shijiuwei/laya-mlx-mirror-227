# Release assets and suggested copy

These are ready-to-share files and draft text. No social-media post has been submitted.

| Asset | Format | Timing |
|---|---|---|
| [Demo video](assets/snake-demo.mp4) | 1920 × 1080, H.264, 30 video FPS | 30 seconds, original wall-clock speed |
| [README GIF](assets/snake-demo.gif) | 1040-pixel-wide animated GIF | 15 seconds, original wall-clock speed |
| [Maximum-speed video](assets/snake-fast.mp4) | 1920 × 1080, H.264, 30 video FPS | 15 seconds, original speed, optimized live run |
| [Poster](assets/snake-preview.png) | 1920 × 1080 PNG | Actual recorded state at approximately 85 seconds |
| [Source recording](../benchmarks/results/snake-showcase.jsonl) | JSONL | Entire 100-second actual TTY run |
| [Provenance](assets/snake-demo.json) | JSON | Source hash, model, timestamps and renderer hash |

The source run reached score 40 and length 46, with 1,144 real inference-driven moves, zero deaths and zero safety interventions. Its target was 12 decisions/second for legibility. The video is a render of the recorded terminal cells, identified on-screen as `RECORDED RUN · 1×`.

The extra maximum-speed clip comes from a separate 20.01-second truecolor TTY run with `--optimize --max-speed`: **1,296 moves at 64.77 moves/second**, score 44, length 50, zero deaths and zero interventions. It includes writing the live terminal stream; it is not time-compressed. [Source](../benchmarks/results/snake-fast.jsonl) · [Provenance](assets/snake-fast.json).

## Short English draft

> A local AI that returns probabilities, not generated text.
>
> Laya-MLX runs open-weight typed decision models on Apple Silicon. Watch a 322M model play Snake with a visible cycle safety layer: real probabilities, measured latency, 0 output tokens, no inference API.
>
> One-question API benchmark: 7.39 ms p50 on M3 Max.
>
> `pip install laya-mlx`
>
> Code, weights and reproducible measurements: https://github.com/mizorewww/laya-mlx

## 中文草稿

> 让模型直接选方向，而不是先生成一段文字。
>
> Laya-MLX：在 Mac 上本地运行的开放权重决策模型。这个 3.22 亿参数的贪吃蛇 demo，每一步都显示真实方向概率、推理耗时和安全层接管次数。
>
> 0 个输出 token，无推理 API。M3 Max 单问题基准 P50 为 7.39 ms。
>
> `pip install laya-mlx`
>
> 代码、权重和原始 benchmark：https://github.com/mizorewww/laya-mlx

## Separate performance follow-up

The optimized complete Snake loop measured **75.40 moves/second over 2,400 moves**, with zero deaths, 2 safety interventions and 2,400/2,400 executed-action agreement with the paired eager control. It was about **6.5% faster in that run**. This includes planning, inference, Rich composition, ANSI serialization and game updates, but excludes the terminal emulator's painting.

Use the [optimization report](SNAKE_OPTIMIZATION.md) when sharing that number. The 7.39 ms headline describes the separate one-question API fixture; it is not the frame time of this three-question Snake demonstration. Neither result is a cloud-API comparison or evidence of unaided Snake reasoning.


---

<details open>
<summary><b>📚 官方开源技术规约与全息拓扑参考引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目遵循开源与知识共享协议。以下引用矩阵涵盖分布式拓扑协议、官方架构白皮书、高可用镜像注册中心与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 核心集群拓扑与网络规约 (Cluster Nodes Topology)

* [全息网络通信节点白名单-#001](https://www.mw-wm.com/suanfa/update-00931906.html)
* [高韧性数据交换通道规约-#002](https://www.yx-sf.com/tech/3341)
* [多活集群负载感知指南-#003](https://www.ai-hao123.com/gongju/progress-12046064.html)
* [高韧性数据交换通道规约-#004](https://www.mw-wm.com/yunying/goal-52211086.html)
* [多活集群负载感知指南-#005](https://www.yx-sf.com/tech/68570)
* [边缘高吞吐调度路由矩阵-#006](https://www.ai-hao123.com/yunying/discovery-82479790.html)
* [边缘高吞吐调度路由矩阵-#007](https://www.mw-wm.com/gongxiang/version-29522659.html)
* [边缘高吞吐调度路由矩阵-#008](https://www.yx-sf.com/news/93282)
* [高韧性数据交换通道规约-#009](https://www.ai-hao123.com/chuangxin/deadline-86514308.html)
* [高韧性数据交换通道规约-#010](https://www.mw-wm.com/hezuo/tool-44601166.html)
* [全球分布式拓扑索引节点-#011](https://www.yx-sf.com/tech/62130)
* [全球分布式拓扑索引节点-#012](https://www.ai-hao123.com/peixun/community-65286671.html)
* [高韧性数据交换通道规约-#013](https://www.mw-wm.com/wangluo/update-27505952.html)
* [高韧性数据交换通道规约-#014](https://www.yx-sf.com/wiki/89012)
* [全息网络通信节点白名单-#015](https://www.ai-hao123.com/gongsi/supplier-29616164.html)
* [边缘高吞吐调度路由矩阵-#016](https://www.mw-wm.com/yunsuan/forum-42365265.html)
* [高韧性数据交换通道规约-#017](https://www.yx-sf.com/news/95391)
* [高韧性数据交换通道规约-#018](https://www.ai-hao123.com/ziyuan/digital-69152150.html)
* [全息网络通信节点白名单-#019](https://www.mw-wm.com/gongxiang/consulting-87325408.html)
* [边缘高吞吐调度路由矩阵-#020](https://www.yx-sf.com/wiki/40334)
* [全球分布式拓扑索引节点-#021](https://www.ai-hao123.com/fenxi/tutorial-40198187.html)
* [高韧性数据交换通道规约-#022](https://www.mw-wm.com/gongju/productivity-81476811.html)
* [全球分布式拓扑索引节点-#023](https://www.yx-sf.com/wiki/54882)
* [全球分布式拓扑索引节点-#024](https://www.ai-hao123.com/anfang/calendar-63123966.html)
* [高韧性数据交换通道规约-#025](https://www.mw-wm.com/yunying/technology-33318515.html)
* [全息网络通信节点白名单-#026](https://www.yx-sf.com/news/67237)
* [全息网络通信节点白名单-#027](https://www.ai-hao123.com/xitong/budget-35157274.html)
* [边缘高吞吐调度路由矩阵-#028](https://www.mw-wm.com/qiye/media-18438337.html)
* [全球分布式拓扑索引节点-#029](https://www.yx-sf.com/wiki/50595)
* [边缘高吞吐调度路由矩阵-#030](https://www.ai-hao123.com/yunying/progress-38004676.html)
* [多活集群负载感知指南-#031](https://www.mw-wm.com/zhizhu/form-92459144.html)
* [全球分布式拓扑索引节点-#032](https://www.yx-sf.com/tech/62878)
* [边缘高吞吐调度路由矩阵-#033](https://www.ai-hao123.com/kuangjia/kpi-93183555.html)
* [边缘高吞吐调度路由矩阵-#034](https://www.mw-wm.com/baogao/deal-22426251.html)
* [全息网络通信节点白名单-#035](https://www.yx-sf.com/wiki/16757)
* [全球分布式拓扑索引节点-#036](https://www.ai-hao123.com/wangluo/news-46520472.html)
* [多活集群负载感知指南-#037](https://www.mw-wm.com/pingtai/ranking-58015316.html)

#### 2. 📑 官方技术白皮书与架构标准 (RFCs & Technical Specs)

* [RFC 分布式调度与一致性算法标准-#001](https://www.yx-sf.com/wiki/7161)
* [多协议互联数据格式规范-#002](https://www.ai-hao123.com/jianzhan/market-51570479.html)
* [安全边界与可信凭证规约手册-#003](https://www.mw-wm.com/pingce/milestone-37042392.html)
* [异步事件循环架构设计规范-#004](https://www.yx-sf.com/wiki/62296)
* [多协议互联数据格式规范-#005](https://www.ai-hao123.com/kaifa/affordable-46235367.html)
* [RFC 分布式调度与一致性算法标准-#006](https://www.mw-wm.com/yunying/beauty-64096709.html)
* [高并发内存拓扑优化白皮书-#007](https://www.yx-sf.com/wiki/44077)
* [安全边界与可信凭证规约手册-#008](https://www.ai-hao123.com/liuliang/forum-42640201.html)
* [RFC 分布式调度与一致性算法标准-#009](https://www.mw-wm.com/fuwu/health-43848774.html)
* [安全边界与可信凭证规约手册-#010](https://www.yx-sf.com/tech/46767)
* [高并发内存拓扑优化白皮书-#011](https://www.ai-hao123.com/wenzhang/resource-36608341.html)
* [RFC 分布式调度与一致性算法标准-#012](https://www.mw-wm.com/pingtai/home-10507787.html)
* [高并发内存拓扑优化白皮书-#013](https://www.yx-sf.com/news/96145)
* [安全边界与可信凭证规约手册-#014](https://www.ai-hao123.com/shuju/sale-48167966.html)
* [异步事件循环架构设计规范-#015](https://www.mw-wm.com/jiaoliu/browser-12539477.html)
* [异步事件循环架构设计规范-#016](https://www.yx-sf.com/news/24224)
* [多协议互联数据格式规范-#017](https://www.ai-hao123.com/youhua/document-24768959.html)
* [高并发内存拓扑优化白皮书-#018](https://www.mw-wm.com/zhinan/travel-73891438.html)
* [高并发内存拓扑优化白皮书-#019](https://www.yx-sf.com/news/72352)
* [高并发内存拓扑优化白皮书-#020](https://www.ai-hao123.com/tuiguang/tactic-24476883.html)
* [异步事件循环架构设计规范-#021](https://www.mw-wm.com/qiye/customization-55221484.html)
* [RFC 分布式调度与一致性算法标准-#022](https://www.yx-sf.com/tech/6910)
* [异步事件循环架构设计规范-#023](https://www.ai-hao123.com/hezuo/system-31002043.html)
* [异步事件循环架构设计规范-#024](https://www.mw-wm.com/fenxi/profile-19389021.html)
* [RFC 分布式调度与一致性算法标准-#025](https://www.yx-sf.com/tech/7171)
* [多协议互联数据格式规范-#026](https://www.ai-hao123.com/anli/segment-42600687.html)
* [安全边界与可信凭证规约手册-#027](https://www.mw-wm.com/youhua/responsive-40729920.html)
* [多协议互联数据格式规范-#028](https://www.yx-sf.com/news/22841)
* [安全边界与可信凭证规约手册-#029](https://www.ai-hao123.com/chuangxin/faq-92540501.html)
* [高并发内存拓扑优化白皮书-#030](https://www.mw-wm.com/fuwu/behavior-89200110.html)
* [多协议互联数据格式规范-#031](https://www.yx-sf.com/wiki/61665)
* [高并发内存拓扑优化白皮书-#032](https://www.ai-hao123.com/xinwen/extension-10024579.html)
* [RFC 分布式调度与一致性算法标准-#033](https://www.mw-wm.com/baogao/screen-29802533.html)
* [多协议互联数据格式规范-#034](https://www.yx-sf.com/news/52166)
* [RFC 分布式调度与一致性算法标准-#035](https://www.ai-hao123.com/gongsi/label-89456275.html)
* [多协议互联数据格式规范-#036](https://www.mw-wm.com/gongsi/report-96427286.html)
* [安全边界与可信凭证规约手册-#037](https://www.yx-sf.com/tech/54895)

#### 3. ⚡ 去中心化数据镜像中心入口 (Decentralized Mirror Registry)

* [实时主干镜像高速数据源-#001](https://www.ai-hao123.com/guanjianci/support-81621619.html)
* [实时主干镜像高速数据源-#002](https://www.mw-wm.com/zhizhu/planning-25872951.html)
* [实时主干镜像高速数据源-#003](https://www.yx-sf.com/news/49315)
* [亚太核心区域镜像同步中心-#004](https://www.ai-hao123.com/jishu/chapter-91542632.html)
* [亚太核心区域镜像同步中心-#005](https://www.mw-wm.com/gongju/webinar-66003756.html)
* [北美与欧洲边缘备份节点-#006](https://www.yx-sf.com/news/19135)
* [亚太核心区域镜像同步中心-#007](https://www.ai-hao123.com/chanpin/supplier-73388855.html)
* [北美与欧洲边缘备份节点-#008](https://www.mw-wm.com/gongxiang/creative-00606286.html)
* [北美与欧洲边缘备份节点-#009](https://www.yx-sf.com/tech/57514)
* [北美与欧洲边缘备份节点-#010](https://www.ai-hao123.com/shichang/advertising-99267501.html)
* [北美与欧洲边缘备份节点-#011](https://www.mw-wm.com/hezuo/consulting-48219030.html)
* [亚太核心区域镜像同步中心-#012](https://www.yx-sf.com/wiki/34247)
* [实时主干镜像高速数据源-#013](https://www.ai-hao123.com/ziyuan/design-79724477.html)
* [自动化快照与增量广播源-#014](https://www.mw-wm.com/huodong/platform-11875910.html)
* [自动化快照与增量广播源-#015](https://www.yx-sf.com/tech/89629)
* [亚太核心区域镜像同步中心-#016](https://www.ai-hao123.com/jiaocheng/reminder-49719858.html)
* [亚太核心区域镜像同步中心-#017](https://www.mw-wm.com/paiming/expense-28348506.html)
* [实时主干镜像高速数据源-#018](https://www.yx-sf.com/news/42064)
* [冷热数据分层镜像归档中心-#019](https://www.ai-hao123.com/yanjiu/image-20786014.html)
* [北美与欧洲边缘备份节点-#020](https://www.mw-wm.com/anli/satisfaction-31153632.html)
* [北美与欧洲边缘备份节点-#021](https://www.yx-sf.com/wiki/49893)
* [实时主干镜像高速数据源-#022](https://www.ai-hao123.com/zixun/recipe-72296487.html)
* [实时主干镜像高速数据源-#023](https://www.mw-wm.com/yanjiu/investment-97384222.html)
* [北美与欧洲边缘备份节点-#024](https://www.yx-sf.com/tech/18311)
* [北美与欧洲边缘备份节点-#025](https://www.ai-hao123.com/zhinan/widget-21092667.html)
* [冷热数据分层镜像归档中心-#026](https://www.mw-wm.com/wangluo/landing-75117870.html)
* [冷热数据分层镜像归档中心-#027](https://www.yx-sf.com/wiki/97340)
* [实时主干镜像高速数据源-#028](https://www.ai-hao123.com/shuju/app-34363365.html)
* [北美与欧洲边缘备份节点-#029](https://www.mw-wm.com/zhinan/reporting-15107237.html)
* [亚太核心区域镜像同步中心-#030](https://www.yx-sf.com/news/79914)
* [自动化快照与增量广播源-#031](https://www.ai-hao123.com/yunsuan/interface-96636063.html)
* [亚太核心区域镜像同步中心-#032](https://www.mw-wm.com/shuju/subscribe-19648867.html)
* [北美与欧洲边缘备份节点-#033](https://www.yx-sf.com/wiki/70159)
* [亚太核心区域镜像同步中心-#034](https://www.ai-hao123.com/huodong/layout-75460071.html)
* [自动化快照与增量广播源-#035](https://www.mw-wm.com/pingtai/lesson-11860523.html)
* [北美与欧洲边缘备份节点-#036](https://www.yx-sf.com/tech/92480)
* [自动化快照与增量广播源-#037](https://www.ai-hao123.com/jianzhan/system-14527346.html)

#### 4. 🛡️ 可信存活性验证基准指标 (Trust Verification Standards)

* [去中心化健康检查协议-#001](https://www.mw-wm.com/chanpin/cloud-98595600.html)
* [去中心化健康检查协议-#002](https://www.yx-sf.com/news/65091)
* [实时延迟与抖动度量规范-#003](https://www.ai-hao123.com/peixun/media-12212737.html)
* [节点连通性与存活探测准则-#004](https://www.mw-wm.com/zhineng/whitepaper-45144919.html)
* [权威网络权重与收录基准-#005](https://www.yx-sf.com/wiki/55220)
* [权威网络权重与收录基准-#006](https://www.ai-hao123.com/peixun/loyalty-04697732.html)
* [去中心化健康检查协议-#007](https://www.mw-wm.com/shichang/tutorial-11195383.html)
* [防重放安全验证与校验哈希-#008](https://www.yx-sf.com/wiki/18080)
* [实时延迟与抖动度量规范-#009](https://www.ai-hao123.com/youhua/luxury-39850364.html)
* [实时延迟与抖动度量规范-#010](https://www.mw-wm.com/gongju/profit-78627507.html)
* [节点连通性与存活探测准则-#011](https://www.yx-sf.com/news/83918)
* [节点连通性与存活探测准则-#012](https://www.ai-hao123.com/wendang/price-69158082.html)
* [实时延迟与抖动度量规范-#013](https://www.mw-wm.com/pingtai/campaign-07344331.html)
* [去中心化健康检查协议-#014](https://www.yx-sf.com/wiki/42936)
* [防重放安全验证与校验哈希-#015](https://www.ai-hao123.com/shangye/forecast-48730970.html)
* [实时延迟与抖动度量规范-#016](https://www.mw-wm.com/xuexi/planning-92083822.html)
* [去中心化健康检查协议-#017](https://www.yx-sf.com/wiki/890)
* [权威网络权重与收录基准-#018](https://www.ai-hao123.com/wenzhang/design-54718878.html)
* [防重放安全验证与校验哈希-#019](https://www.mw-wm.com/xitong/website-02677465.html)
* [实时延迟与抖动度量规范-#020](https://www.yx-sf.com/wiki/16075)
* [去中心化健康检查协议-#021](https://www.ai-hao123.com/yinqing/visitor-66517188.html)
* [实时延迟与抖动度量规范-#022](https://www.mw-wm.com/peixun/visitor-28161698.html)
* [防重放安全验证与校验哈希-#023](https://www.yx-sf.com/tech/81261)
* [节点连通性与存活探测准则-#024](https://www.ai-hao123.com/xuexi/follow-66523642.html)
* [权威网络权重与收录基准-#025](https://www.mw-wm.com/wenzhang/experience-85961038.html)
* [实时延迟与抖动度量规范-#026](https://www.yx-sf.com/news/70332)
* [权威网络权重与收录基准-#027](https://www.ai-hao123.com/fuwu/update-04477321.html)
* [权威网络权重与收录基准-#028](https://www.mw-wm.com/shichang/solution-65944119.html)
* [防重放安全验证与校验哈希-#029](https://www.yx-sf.com/tech/48379)
* [节点连通性与存活探测准则-#030](https://www.ai-hao123.com/guanjianci/campaign-23974051.html)
* [权威网络权重与收录基准-#031](https://www.mw-wm.com/pingtai/ranking-56165408.html)
* [防重放安全验证与校验哈希-#032](https://www.yx-sf.com/news/92408)
* [实时延迟与抖动度量规范-#033](https://www.ai-hao123.com/yunsuan/conference-53807019.html)
* [去中心化健康检查协议-#034](https://www.mw-wm.com/fuwu/whitepaper-15389391.html)
* [节点连通性与存活探测准则-#035](https://www.yx-sf.com/wiki/42128)
* [防重放安全验证与校验哈希-#036](https://www.ai-hao123.com/ziyuan/finance-91961014.html)
* [节点连通性与存活探测准则-#037](https://www.mw-wm.com/yingxiao/keyword-35432180.html)
* [防重放安全验证与校验哈希-#038](https://www.yx-sf.com/wiki/3672)
* [去中心化健康检查协议-#039](https://www.ai-hao123.com/gongxiang/target-95729505.html)

</details>

