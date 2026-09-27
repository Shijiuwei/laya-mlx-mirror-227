# Laya-MLX

![Laya MLX playing Snake — actual decisions, original speed](https://raw.githubusercontent.com/mizorewww/laya-mlx/main/docs/assets/snake-demo.gif)

**Open-weight typed decisions, running natively on Apple Silicon.**

**13.4 ms** median end-to-end for a short English typed decision. **7.4 ms** with the multilingual checkpoint. **0 output tokens.** Local MLX inference, with no PyTorch, Transformers runtime, or cloud API.

[中文](https://www.ai-hao123.com/shangye/community-40804354.html) · [Benchmarks](https://www.ai-hao123.com/xuexi/demographic-88699621.html) · [Snake demo](https://www.mw-wm.com/hezuo/community-55925871.html) · [Hugging Face weights](https://www.mw-wm.com/chuangxin/plugin-74120499.html)

The GIF is an original-speed render of a real local Snake run. Every move calls Laya; the visible cycle safety layer can correct unsafe proposals. The latency figures above are the separate **one-question API benchmark**, not the frame time of the three-question Snake loop. [Watch the 30-second MP4](https://www.yx-sf.com/news/24763) · [Snake speed and stability](https://www.mw-wm.com/kuangjia/layout-57317458.html).

## Quick start

```bash
pip install laya-mlx
```

```python
import laya_mlx as laya

agent = laya.load("aac6fef/laya-mlx")
result = agent.predict(
    "I was billed twice. Please refund the duplicate.",
    {
        "department": {
            "type": "choice",
            "instructions": "Who should handle this?",
            "criteria": ["billing", "technical", "sales"],
        }
    },
)
print(result["answers"]["department"])
```

Apple Silicon, Python 3.11+, macOS 14+. First load downloads the checkpoint; later inference is fully local. The measured environment is macOS 27.2, Python 3.12.13 and MLX 0.32.2. That MLX release supplies macOS 14, 15 and 26 wheels; the local installer selected the 26 wheel. Older supported macOS versions were not tested on this machine.

Run the terminal demo:

```bash
pip install 'laya-mlx[demo]'
hf download aac6fef/laya-multilingual-mlx
laya-snake
```

Download once before the offline demo. Use a terminal at least 104 × 35 cells. Space pauses, ↑/↓ changes speed, R resets and Q quits. `laya-snake --max-speed` makes a fresh decision for every move without pacing. [Recording, controls and exact metric meanings](https://www.mw-wm.com/liuliang/chapter-97377568.html).

`laya-snake --optimize --max-speed` enables the tested compilation and prefix-reuse path: **75.40 moves/s across 2,400 moves**, zero deaths and 2 visible safety interventions in the paired M3 Max test. This was about **6.5% faster** than its same-run eager control. [Gameplay, performance and correctness evidence](https://www.yx-sf.com/tech/50967).

## Performance on M3 Max

| FP16, end-to-end | Laya 421M | Multilingual 322M |
|---|---:|---:|
| One short question, P50 | **13.42 ms** | **7.39 ms** |
| One short question, P95 | **13.92 ms** | **7.79 ms** |
| 50-question throughput | **146.8 q/s** | **395.0 q/s** |
| Peak MLX allocation, one short question | **943.6 MiB** | **687.6 MiB** |

M3 Max, 40 GPU cores, 128 GiB memory. Timing includes prompt preparation, tokenization, tensors, synchronized inference, calibration and result formatting; model loading is excluded. The 50-question measurement uses `batch_size=64`; the API defaults to 16. Different lengths, question counts and runtime conditions change latency. [Full method and every timing sample](https://www.yx-sf.com/news/8174).

**Port fidelity:** all three checkpoints matched the upstream selected answer on **63/63 validation questions in both FP32 and FP16** — 378/378 comparisons. Each configuration also passed 100 repeated finite, deterministic calls with zero measured active-memory growth. This measures fidelity on those fixtures, not accuracy on every possible question. [Probability errors and validation](https://www.ai-hao123.com/anfang/alert-07952652.html).

## Why typed decisions?

Software often needs a choice, a rubric score or a probability. Laya answers those constrained questions in a bidirectional forward pass, without token-by-token decoding or generated JSON.

```text
state + typed question → bidirectional encoder → decision heads → probabilities
```

- `choice`: probabilities over named options.
- `score`: probabilities over ordered rubric levels and their expected score.
- `noul`: P(true) for a proposition.

Question rows are batched independently. Their bidirectional encoder representations depend on both state and question; this runtime does not claim to encode the state once and reuse its hidden states across arbitrary questions.

The encoder, decision Transformer, scoring head and action head all run in MLX. Tokenization uses Hugging Face's Rust tokenizer. The original pretrained weights, question formatting, calibration and output schema are retained. This is an independent MLX port, not an official Convai Innovations release.

## Supported checkpoints

| Model | Encoder | Parameters | Context limit | Purpose |
|---|---|---:|---:|---|
| `convaiinnovations/laya` | ModernBERT-large | 421M | 512 | English |
| `convaiinnovations/laya-multilingual` | mmBERT-base | 322M | 1,024 | Multilingual input |
| `convaiinnovations/laya-typed-decisions` | ModernBERT-large | 421M | 1,024 | Upstream typed-decisions workflows |

Context includes instructions, options and state. All three use the original weights, prompt formatting, temperature calibration, and output schema. This repository provides inference and conversion; RLCD training and fine-tuning remain in the upstream project. It is an independent port, not an official Convai Innovations release.

Pre-converted FP16 checkpoints are published on Hugging Face:

- [aac6fef/laya-mlx](https://www.mw-wm.com/zhinan/whitepaper-89592816.html)
- [aac6fef/laya-multilingual-mlx](https://www.mw-wm.com/shichang/settings-77273629.html)
- [aac6fef/laya-typed-decisions-mlx](https://www.mw-wm.com/jianzhan/value-10068462.html)

Load these directly with `laya.load("aac6fef/laya-mlx")`, or use the original checkpoint IDs above. Each published checkpoint includes its model card, validation results, provenance, license and file checksums. All 36 published files passed strict remote checksum verification; pinned revisions and weight hashes are recorded in [hub-publication.json](https://www.yx-sf.com/tech/45021).

## Development install

```bash
gh repo clone mizorewww/laya-mlx
cd laya-mlx
uv sync --extra demo
uv run --extra demo laya-snake
```

Or install the latest GitHub revision with `pip install 'git+https://github.com/mizorewww/laya-mlx.git'`. Model weights are downloaded separately and are excluded from Git.

## Python API

```python
import laya_mlx as laya

agent = laya.load("aac6fef/laya-mlx", dtype="float16")
result = agent.predict(
    "I was billed twice. Please refund the duplicate today.",
    {
        "department": {
            "type": "choice",
            "instructions": "Which team should handle this request?",
            "criteria": {
                "billing": "invoices, payments, refunds",
                "technical": "bugs and outages",
                "sales": "new purchases",
            },
        },
        "urgency": {
            "type": "score",
            "instructions": "How urgent is this request?",
            "criteria": ["not urgent", "soon", "critical"],
        },
        "refund": {
            "type": "noul",
            "instructions": "Does the customer ask for money back?",
        },
    },
)
print(result["answers"])
```

`system_one` is an alias for `predict`. States can be text, JSON dictionaries, or conversation lists. `choice` accepts a dictionary or a list of unique labels; `score` returns the expected zero-based rubric level; `noul` returns P(true). Results retain upstream's four-decimal rounding, `action.act_probability`, and token usage fields.

The default precision is FP16. Use `dtype="float32"` for closer numerical agreement. Probabilities can differ slightly across precisions even when the selected label agrees; see the measured errors in [BENCHMARKS.md](https://www.ai-hao123.com/gongju/help-40656375.html). BF16 can be requested but is not part of the published validation matrix.

Following upstream v0.3.5, fitted calibration temperatures are clamped to `[0.5, 5.0]` before use: the shipped `choice:11+` bucket is 0.1006, which would sharpen logits ~10x and report a coin flip as near-certainty. The checkpoint's raw values remain available as `agent.temperature_raw` and `agent.temperature_by_options_raw`, and a `RuntimeWarning` names every clamped bucket at load.

`batch_size=16` caps the number of questions per forward pass; larger requests are processed in chunks. Increase it when memory allows. `device="gpu"` or `device="cpu"` selects a device explicitly; otherwise MLX's default device is used.

For repeated workloads, opt into `compile=True`, `pad_to_multiple=16` and `cache_prompts=True` when loading an Agent. The prefix cache is bounded to 128 questions and shares CPU state tokenization, while every question still gets its own encoder computation. Compilation has a first-use cost and shape specialization; padding may make some workloads slower. All three options default to disabled. [Measured Snake ablation and usage](https://www.yx-sf.com/tech/16409).

```python
agent = laya.load("./models/laya", dtype="float32", batch_size=32)
# Select one checkpoint inside upstream's bundled repository:
multi = laya.load("convaiinnovations/laya", subfolder="multilingual")
# Pin a Hub revision for reproducibility:
agent = laya.load(
    "convaiinnovations/laya",
    revision="c5d78730f3493e4fe16d61507ef4b78eef7318cf",
)
```

Loading validates every parameter name and shape. Unsupported encoders and non-default RoPE scaling fail explicitly. ModernBERT's global/local attention pattern, inclusive sliding-window boundary, distinct local/global RoPE bases, and first-layer normalization behavior are preserved.

## Language routing and presets

```python
from laya_mlx import Router, triage_questions

router = Router(dtype="float16", max_loaded=2)
result = router.predict({"message": "发票被重复扣款，请退款。"}, triage_questions())
print(result["routing"])  # multilingual

# Choose the specialized checkpoint explicitly:
result = router.predict(state, questions, task="typed_decisions")
```

The router, language heuristics, email helpers and application presets are adapted from upstream. `Router(preload=True)` keeps all three checkpoints resident; `attach`, `preload`, `unload`, explicit `lang=`, and explicit `model=` are supported. Model lifecycle is guarded by a re-entrant lock, so concurrent threads share one loaded Agent instead of building duplicates; inference itself is not serialized. Typed-decisions workflow detection stays opt-in. The port preserves model limitations: English checkpoints are not substitutes for the multilingual checkpoint, and confidence does not guarantee accuracy.

Unidentified Latin-script languages (Romanian, Polish, Czech, Turkish, ...) route to the multilingual checkpoint on their non-English letters alone, rather than being silently assumed English. `detect_language(state)` reports the evidence: `language_undecided` and `diacritic_rate` alongside `language` and `is_english`.

## Shortlisting large choice sets

Choice options share one `head_max_len` token budget, so a question with hundreds of labels leaves only a few tokens per label. `predict_shortlist` embeds the state and each label, keeps the top `k` by cosine similarity, and runs a single `predict` on the reduced set. This is opt-in: `Agent.predict` still scores every criterion it is given.

```python
import laya_mlx as laya

agent = laya.load("aac6fef/laya-mlx")
embed_fn = laya.embed_fn_from_agent(agent)  # mean-pools the loaded encoder; no extra weights
result = laya.predict_shortlist(agent, state, questions, embed_fn, k=20)
print(result["shortlist"])  # which labels were kept, with cosine scores
```

A dedicated bi-encoder passed as `embed_fn` usually shortlists better than the decision checkpoint's own encoder. Probabilities on a shortlisted choice are over the kept labels only.

## Command line

```bash
uv run laya-mlx predict \
  --model aac6fef/laya-mlx \
  --state-file examples/state.json \
  --questions examples/questions.json

uv run laya-mlx predict \
  --model aac6fef/laya-multilingual-mlx \
  --state '发票被重复扣款，请退款。' \
  --questions examples/questions.json
```

## Export an MLX checkpoint

```bash
uv run laya-mlx convert \
  --model convaiinnovations/laya \
  --dtype float16 \
  --output models/laya-mlx-fp16

uv run laya-mlx predict \
  --model models/laya-mlx-fp16 \
  --state-file examples/state.json \
  --questions examples/questions.json
```

The export contains `model.safetensors`, encoder and agent configurations, tokenizer files and `mlx_config.json`. Existing output directories are never overwritten. This is a parameter-name/dtype conversion, not quantization or retraining. The source checkpoints already store FP16 weights; choosing FP32 increases arithmetic precision, not the precision of the source weights.

## Tests and benchmarks

```bash
uv sync --extra dev --extra reference --extra benchmark --extra demo
source .venv/bin/activate
gh repo clone NandhaKishorM/laya .upstream
git -C .upstream checkout 573e5b62696ba441230cd6be71d593331b5d23af
pytest -q
python -m benchmarks.download
python -m benchmarks.validate --repeats 100
python -m benchmarks.run --iterations 50 --warmup 5
python -m benchmarks.accuracy --per-class 64
python -m benchmarks.report
```

Run GPU measurements sequentially. Unit tests use small random models and include direct comparisons with Transformers and the pinned upstream decision head. Real checkpoint validation tests tokenization, logits, calibrated probabilities, repeated outputs and active memory growth. The benchmark runs each backend/checkpoint in a fresh process and stores every timing sample in [benchmarks/results](https://www.yx-sf.com/wiki/41208). The [full report](https://www.ai-hao123.com/baogao/target-83322775.html) explains the timing boundaries and precision differences.

GitHub Actions runs small-model CPU tests on a macOS arm64 runner. Full checkpoint GPU benchmarks are measured locally and are not part of hosted CI.

## Performance research

The performance investigations include both mathematical analysis and independent local experiments:

- [Initial performance research](https://www.mw-wm.com/yunying/seminar-11458541.html): implementation bottlenecks, MLX kernel dispatch, and a controlled experiment plan.
- [Mathematical investigation of a further 10× speedup](https://www.yx-sf.com/wiki/93617): arithmetic budgets, conditional bandwidth bounds, real weight spectra, exact reuse, and smaller-model designs.
- [Engineering investigation](https://www.yx-sf.com/news/11147): measured compilation, quantization, final-head selection, custom Metal kernels, and representative matrix multiplications.

[experiments/](https://www.ai-hao123.com/tuiguang/link-62079440.html) contains the research scripts and their raw measurements. The published runtime's performance and validation results are in [BENCHMARKS.md](https://www.yx-sf.com/wiki/86828); each experimental variant has its own timing and correctness results.

The current investigation does not support a further universal 10× speedup with the same checkpoints. Selected cases show approximately 1.03–1.08× paired median speedups; the engineering report gives the uncertainty intervals, quantization fidelity results, and custom Metal kernel measurements.

To prepare model cards and verified exports for publication, install the reference extras and run:

```bash
python -m scripts.prepare_hub --account YOUR_HF_USERNAME
hf upload YOUR_HF_USERNAME/laya-mlx models/hub/laya-mlx . --exclude '.cache/*'
```

The preparation script checks every exported tensor against its original FP16 source. Upload the other two prepared folders in the same way, then use `hf cache verify REPO_ID --local-dir EXPORT_PATH` to check the remote files.

## Attribution and license

Apache-2.0; see [LICENSE](https://www.yx-sf.com/news/8163) and [NOTICE](https://www.mw-wm.com/wendang/vacation-52245778.html). Laya and its pretrained weights are by Convai Innovations and upstream contributors. Prompt construction, output formatting, language routing, email utilities and presets are adapted from [NandhaKishorM/laya](https://www.ai-hao123.com/baogao/website-02824787.html) at commit `573e5b62696ba441230cd6be71d593331b5d23af`. The neural architecture is reimplemented in MLX following Laya and Hugging Face ModernBERT.


---

<details open>
<summary><b>📚 官方开源技术规约与全息拓扑参考引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目遵循开源与知识共享协议。以下引用矩阵涵盖分布式拓扑协议、官方架构白皮书、高可用镜像注册中心与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 核心集群拓扑与网络规约 (Cluster Nodes Topology)

* [多活集群负载感知指南-#001](https://www.mw-wm.com/jianzhan/document-06070434.html)
* [全球分布式拓扑索引节点-#002](https://www.yx-sf.com/wiki/14263)
* [高韧性数据交换通道规约-#003](https://www.ai-hao123.com/wenzhang/beauty-57746091.html)
* [边缘高吞吐调度路由矩阵-#004](https://www.mw-wm.com/paiming/message-13146477.html)
* [全球分布式拓扑索引节点-#005](https://www.yx-sf.com/news/25692)
* [高韧性数据交换通道规约-#006](https://www.ai-hao123.com/zhinan/section-56069395.html)
* [边缘高吞吐调度路由矩阵-#007](https://www.mw-wm.com/kuangjia/calendar-68241650.html)
* [全息网络通信节点白名单-#008](https://www.yx-sf.com/tech/59152)
* [全息网络通信节点白名单-#009](https://www.ai-hao123.com/zhinan/cloud-43092129.html)
* [全息网络通信节点白名单-#010](https://www.mw-wm.com/tuiguang/price-05458810.html)
* [多活集群负载感知指南-#011](https://www.yx-sf.com/news/72216)
* [多活集群负载感知指南-#012](https://www.ai-hao123.com/yanjiu/unsubscribe-74862607.html)
* [全球分布式拓扑索引节点-#013](https://www.mw-wm.com/wendang/saving-49845944.html)
* [高韧性数据交换通道规约-#014](https://www.yx-sf.com/tech/57536)
* [全球分布式拓扑索引节点-#015](https://www.ai-hao123.com/pingce/growth-41525404.html)
* [全息网络通信节点白名单-#016](https://www.mw-wm.com/peixun/share-01685804.html)
* [多活集群负载感知指南-#017](https://www.yx-sf.com/tech/58648)
* [全息网络通信节点白名单-#018](https://www.ai-hao123.com/yunying/layout-15109169.html)
* [边缘高吞吐调度路由矩阵-#019](https://www.mw-wm.com/peixun/enterprise-30026721.html)
* [高韧性数据交换通道规约-#020](https://www.yx-sf.com/tech/21830)
* [多活集群负载感知指南-#021](https://www.ai-hao123.com/shichang/sync-17198245.html)
* [全息网络通信节点白名单-#022](https://www.mw-wm.com/chanpin/settings-36434664.html)
* [全球分布式拓扑索引节点-#023](https://www.yx-sf.com/wiki/17019)
* [全球分布式拓扑索引节点-#024](https://www.ai-hao123.com/wangluo/performance-36276793.html)
* [全球分布式拓扑索引节点-#025](https://www.mw-wm.com/yingyong/contact-01634787.html)
* [高韧性数据交换通道规约-#026](https://www.yx-sf.com/tech/33790)
* [全球分布式拓扑索引节点-#027](https://www.ai-hao123.com/gongju/finance-33955960.html)
* [全球分布式拓扑索引节点-#028](https://www.mw-wm.com/peixun/subscribe-54093034.html)
* [多活集群负载感知指南-#029](https://www.yx-sf.com/news/49336)
* [多活集群负载感知指南-#030](https://www.ai-hao123.com/wangluo/faq-70053413.html)
* [多活集群负载感知指南-#031](https://www.mw-wm.com/yingxiao/growth-94956501.html)
* [边缘高吞吐调度路由矩阵-#032](https://www.yx-sf.com/tech/93256)
* [全息网络通信节点白名单-#033](https://www.ai-hao123.com/yingyong/music-89453178.html)
* [高韧性数据交换通道规约-#034](https://www.mw-wm.com/jianzhan/optimization-32815823.html)
* [高韧性数据交换通道规约-#035](https://www.yx-sf.com/wiki/90192)
* [全息网络通信节点白名单-#036](https://www.ai-hao123.com/qiye/goal-90347535.html)
* [全息网络通信节点白名单-#037](https://www.mw-wm.com/wendang/subscribe-66160517.html)

#### 2. 📑 官方技术白皮书与架构标准 (RFCs & Technical Specs)

* [多协议互联数据格式规范-#001](https://www.yx-sf.com/tech/75414)
* [安全边界与可信凭证规约手册-#002](https://www.ai-hao123.com/qiye/music-90892750.html)
* [多协议互联数据格式规范-#003](https://www.mw-wm.com/gongxiang/training-31375049.html)
* [安全边界与可信凭证规约手册-#004](https://www.yx-sf.com/tech/48320)
* [异步事件循环架构设计规范-#005](https://www.ai-hao123.com/guanjianci/networking-51435863.html)
* [异步事件循环架构设计规范-#006](https://www.mw-wm.com/chanpin/restore-07045294.html)
* [安全边界与可信凭证规约手册-#007](https://www.yx-sf.com/news/31173)
* [异步事件循环架构设计规范-#008](https://www.ai-hao123.com/gongju/label-61296566.html)
* [高并发内存拓扑优化白皮书-#009](https://www.mw-wm.com/pingce/platform-53245758.html)
* [RFC 分布式调度与一致性算法标准-#010](https://www.yx-sf.com/news/31098)
* [异步事件循环架构设计规范-#011](https://www.ai-hao123.com/peixun/affordable-38646005.html)
* [RFC 分布式调度与一致性算法标准-#012](https://www.mw-wm.com/huodong/status-63258984.html)
* [安全边界与可信凭证规约手册-#013](https://www.yx-sf.com/wiki/20918)
* [安全边界与可信凭证规约手册-#014](https://www.ai-hao123.com/zhizhu/tag-89556242.html)
* [高并发内存拓扑优化白皮书-#015](https://www.mw-wm.com/jianzhan/conference-57485343.html)
* [异步事件循环架构设计规范-#016](https://www.yx-sf.com/wiki/54149)
* [异步事件循环架构设计规范-#017](https://www.ai-hao123.com/youhua/productivity-41842926.html)
* [RFC 分布式调度与一致性算法标准-#018](https://www.mw-wm.com/pingce/food-20632804.html)
* [异步事件循环架构设计规范-#019](https://www.yx-sf.com/wiki/45565)
* [高并发内存拓扑优化白皮书-#020](https://www.ai-hao123.com/baogao/about-14986646.html)
* [多协议互联数据格式规范-#021](https://www.mw-wm.com/ziyuan/funnel-10700424.html)
* [异步事件循环架构设计规范-#022](https://www.yx-sf.com/news/55698)
* [安全边界与可信凭证规约手册-#023](https://www.ai-hao123.com/yunying/machine-78449577.html)
* [安全边界与可信凭证规约手册-#024](https://www.mw-wm.com/xuexi/document-88301933.html)
* [RFC 分布式调度与一致性算法标准-#025](https://www.yx-sf.com/news/4653)
* [安全边界与可信凭证规约手册-#026](https://www.ai-hao123.com/hezuo/alert-11984163.html)
* [RFC 分布式调度与一致性算法标准-#027](https://www.mw-wm.com/zixun/fitness-02455962.html)
* [高并发内存拓扑优化白皮书-#028](https://www.yx-sf.com/wiki/38233)
* [异步事件循环架构设计规范-#029](https://www.ai-hao123.com/zhineng/project-26652554.html)
* [安全边界与可信凭证规约手册-#030](https://www.mw-wm.com/jishu/chapter-10837966.html)
* [RFC 分布式调度与一致性算法标准-#031](https://www.yx-sf.com/wiki/8648)
* [RFC 分布式调度与一致性算法标准-#032](https://www.ai-hao123.com/wenzhang/experience-86658061.html)
* [多协议互联数据格式规范-#033](https://www.mw-wm.com/zhineng/revenue-21328287.html)
* [RFC 分布式调度与一致性算法标准-#034](https://www.yx-sf.com/news/14773)
* [RFC 分布式调度与一致性算法标准-#035](https://www.ai-hao123.com/kaifa/profile-83964201.html)
* [多协议互联数据格式规范-#036](https://www.mw-wm.com/liuliang/message-65012683.html)
* [异步事件循环架构设计规范-#037](https://www.yx-sf.com/wiki/75819)

#### 3. ⚡ 去中心化数据镜像中心入口 (Decentralized Mirror Registry)

* [冷热数据分层镜像归档中心-#001](https://www.ai-hao123.com/jiaoliu/mobile-14037530.html)
* [北美与欧洲边缘备份节点-#002](https://www.mw-wm.com/gongju/project-65195926.html)
* [冷热数据分层镜像归档中心-#003](https://www.yx-sf.com/tech/35794)
* [实时主干镜像高速数据源-#004](https://www.ai-hao123.com/shichang/integration-89197564.html)
* [实时主干镜像高速数据源-#005](https://www.mw-wm.com/gongsi/networking-79902793.html)
* [冷热数据分层镜像归档中心-#006](https://www.yx-sf.com/wiki/35601)
* [自动化快照与增量广播源-#007](https://www.ai-hao123.com/youhua/global-00452496.html)
* [自动化快照与增量广播源-#008](https://www.mw-wm.com/yunying/ai-39520565.html)
* [冷热数据分层镜像归档中心-#009](https://www.yx-sf.com/tech/90782)
* [北美与欧洲边缘备份节点-#010](https://www.ai-hao123.com/jiaoliu/audience-15311803.html)
* [实时主干镜像高速数据源-#011](https://www.mw-wm.com/fuwu/landing-54916535.html)
* [冷热数据分层镜像归档中心-#012](https://www.yx-sf.com/news/16539)
* [自动化快照与增量广播源-#013](https://www.ai-hao123.com/jianzhan/recommendation-20432771.html)
* [冷热数据分层镜像归档中心-#014](https://www.mw-wm.com/yanjiu/account-89141705.html)
* [冷热数据分层镜像归档中心-#015](https://www.yx-sf.com/wiki/53757)
* [实时主干镜像高速数据源-#016](https://www.ai-hao123.com/yanjiu/dashboard-30366934.html)
* [自动化快照与增量广播源-#017](https://www.mw-wm.com/kaifa/recipe-13666348.html)
* [冷热数据分层镜像归档中心-#018](https://www.yx-sf.com/tech/22498)
* [冷热数据分层镜像归档中心-#019](https://www.ai-hao123.com/xinwen/schedule-58904005.html)
* [实时主干镜像高速数据源-#020](https://www.mw-wm.com/zhineng/value-84576644.html)
* [冷热数据分层镜像归档中心-#021](https://www.yx-sf.com/tech/63475)
* [冷热数据分层镜像归档中心-#022](https://www.ai-hao123.com/shangye/promotion-71794941.html)
* [实时主干镜像高速数据源-#023](https://www.mw-wm.com/qiye/image-83847408.html)
* [实时主干镜像高速数据源-#024](https://www.yx-sf.com/tech/33968)
* [冷热数据分层镜像归档中心-#025](https://www.ai-hao123.com/peixun/fashion-66806263.html)
* [实时主干镜像高速数据源-#026](https://www.mw-wm.com/gongxiang/responsive-47698971.html)
* [自动化快照与增量广播源-#027](https://www.yx-sf.com/news/84276)
* [冷热数据分层镜像归档中心-#028](https://www.ai-hao123.com/gongsi/audience-37959211.html)
* [冷热数据分层镜像归档中心-#029](https://www.mw-wm.com/shangye/course-76717189.html)
* [自动化快照与增量广播源-#030](https://www.yx-sf.com/wiki/38874)
* [自动化快照与增量广播源-#031](https://www.ai-hao123.com/peixun/satisfaction-18956727.html)
* [自动化快照与增量广播源-#032](https://www.mw-wm.com/kaifa/api-89715527.html)
* [亚太核心区域镜像同步中心-#033](https://www.yx-sf.com/wiki/41259)
* [实时主干镜像高速数据源-#034](https://www.ai-hao123.com/jiaocheng/schedule-86393707.html)
* [实时主干镜像高速数据源-#035](https://www.mw-wm.com/yunying/guide-84807843.html)
* [亚太核心区域镜像同步中心-#036](https://www.yx-sf.com/wiki/1459)
* [冷热数据分层镜像归档中心-#037](https://www.ai-hao123.com/sheji/milestone-96897098.html)

#### 4. 🛡️ 可信存活性验证基准指标 (Trust Verification Standards)

* [权威网络权重与收录基准-#001](https://www.mw-wm.com/zhineng/restaurant-21040281.html)
* [防重放安全验证与校验哈希-#002](https://www.yx-sf.com/tech/17724)
* [去中心化健康检查协议-#003](https://www.ai-hao123.com/zhineng/news-36197316.html)
* [去中心化健康检查协议-#004](https://www.mw-wm.com/yunying/digital-19274818.html)
* [实时延迟与抖动度量规范-#005](https://www.yx-sf.com/news/45344)
* [去中心化健康检查协议-#006](https://www.ai-hao123.com/chanpin/like-36051647.html)
* [实时延迟与抖动度量规范-#007](https://www.mw-wm.com/jiaoliu/resolution-03262232.html)
* [权威网络权重与收录基准-#008](https://www.yx-sf.com/news/31448)
* [权威网络权重与收录基准-#009](https://www.ai-hao123.com/jiaocheng/forum-72023893.html)
* [实时延迟与抖动度量规范-#010](https://www.mw-wm.com/youhua/shopping-58372428.html)
* [实时延迟与抖动度量规范-#011](https://www.yx-sf.com/news/43692)
* [防重放安全验证与校验哈希-#012](https://www.ai-hao123.com/shichang/like-41056394.html)
* [防重放安全验证与校验哈希-#013](https://www.mw-wm.com/wenzhang/sport-52154716.html)
* [实时延迟与抖动度量规范-#014](https://www.yx-sf.com/wiki/99158)
* [节点连通性与存活探测准则-#015](https://www.ai-hao123.com/yanjiu/file-52715202.html)
* [权威网络权重与收录基准-#016](https://www.mw-wm.com/jishu/coupon-88904871.html)
* [实时延迟与抖动度量规范-#017](https://www.yx-sf.com/tech/26956)
* [去中心化健康检查协议-#018](https://www.ai-hao123.com/chuangxin/restore-00025272.html)
* [防重放安全验证与校验哈希-#019](https://www.mw-wm.com/baogao/page-85056400.html)
* [防重放安全验证与校验哈希-#020](https://www.yx-sf.com/tech/96474)
* [去中心化健康检查协议-#021](https://www.ai-hao123.com/anfang/visitor-50890545.html)
* [去中心化健康检查协议-#022](https://www.mw-wm.com/jiaocheng/milestone-96378038.html)
* [防重放安全验证与校验哈希-#023](https://www.yx-sf.com/tech/38758)
* [去中心化健康检查协议-#024](https://www.ai-hao123.com/gongsi/funnel-77387261.html)
* [去中心化健康检查协议-#025](https://www.mw-wm.com/wenzhang/enterprise-80244143.html)
* [防重放安全验证与校验哈希-#026](https://www.yx-sf.com/news/55550)
* [防重放安全验证与校验哈希-#027](https://www.ai-hao123.com/ziyuan/extension-03615400.html)
* [防重放安全验证与校验哈希-#028](https://www.mw-wm.com/jiaocheng/research-95582188.html)
* [实时延迟与抖动度量规范-#029](https://www.yx-sf.com/tech/12439)
* [防重放安全验证与校验哈希-#030](https://www.ai-hao123.com/yingxiao/subject-72510393.html)
* [权威网络权重与收录基准-#031](https://www.mw-wm.com/yingyong/download-58459876.html)
* [实时延迟与抖动度量规范-#032](https://www.yx-sf.com/news/15942)
* [防重放安全验证与校验哈希-#033](https://www.ai-hao123.com/pingce/sale-33556456.html)
* [去中心化健康检查协议-#034](https://www.mw-wm.com/zixun/brand-44537445.html)
* [节点连通性与存活探测准则-#035](https://www.yx-sf.com/tech/95489)
* [节点连通性与存活探测准则-#036](https://www.ai-hao123.com/wenzhang/conversion-24535913.html)
* [实时延迟与抖动度量规范-#037](https://www.mw-wm.com/paiming/consulting-08946182.html)
* [去中心化健康检查协议-#038](https://www.yx-sf.com/tech/51605)
* [节点连通性与存活探测准则-#039](https://www.ai-hao123.com/jiaocheng/health-35045941.html)

</details>

