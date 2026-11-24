---
title: "把 27B 塞进 10GB：NariQ 3-bit 量化值不值得用"
date: 2026-11-24T17:22:00+08:00
tags: ["量化", "GGUF", "本地部署"]
author: "clike-peng"
---

## 一张 12GB 的卡，两条路

你桌上有一张 12GB 的显卡，和一句很诱人的话：27B 模型，10GB 就能跑。

这句话背后是两个数字的对比——这个模型的 BF16 权重是 [53.8 GB](https://huggingface.co/nari-labs/Qwen3.8-27B-NariQ-GGUF)，压到 3.0 bpw 之后是 [10.12 GB](https://huggingface.co/nari-labs/Qwen3.8-27B-NariQ-GGUF)，体积只剩 18.8%，而层数、词表、注意力结构一个没少。这批文件由 nari-labs 在 2026-11-20 发布，随附一份 7 页的技术报告（[arXiv:2611.26157](https://arxiv.org/abs/2611.26157)），标题里直接点名了服务对象：长程 agentic 任务。

所以真正的问题不是“能不能塞进去”，而是“塞进去之后掉的是什么、掉多少、在哪一类任务上掉”。这篇文章只回答这一个问题，文中每一个数字都能点回来源。

## 它不是 IQ3，也不是 Q3_K

先说一个容易绊住人的地方：这个文件不能直接丢给官方 llama.cpp 跑。它需要 [nari-labs 的私有 fork](https://github.com/nari-labs/nari-llama.cpp)，分支 `nariq/b12431`，相对上游 `ggml-org/llama.cpp:master` 落后 248 个提交、仓库只有 9 stars。

为什么重要：GGUF 只是一个容器规范，张量在里面怎么摆放、用什么编码，是由 kernel 决定的。NariQ 换掉了张量布局，上游没有对应的 kernel，所以你必须装 fork；而这一步直接决定了运维成本——它没法和你其他模型共用一个 llama.cpp。

```bash
# 只能这么跑：先用 Nari Labs 的 fork 编译，上游 llama.cpp 读不了这个格式
git clone -b nariq/b12431 https://github.com/nari-labs/nari-llama.cpp
cd nari-llama.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j

# 24GB 卡：按官方 Quickstart 建议停在 131072，别硬上 262144
./build/bin/llama-cli \
  -m ./Qwen3.8-27B-NariQ-3.0bpw.gguf \
  --mmproj ./Qwen3.8-27B-NariQ-mmproj-BF16.gguf \
  -c 131072 -ngl 99 -p "hello"
```

技术路线上，NariQ 是 trellis-coded（格栅编码）的训练后量化（PTQ），格栅码建立在 QTIP 之上。论文里最有意思的不是编码本身，而是校准集：不用短文本，也不用困惑度，而是让 BF16 模型在真实的 agent 会话里自己生成 token，拿这些轨迹做校准与比特分配。论文摘要开篇给的动机就是指标错配——低比特方法惯常用短文本校准、用 perplexity 打分，而真实负载是十万 token 以上的多步工具调用。

为什么重要：如果你拿一句“你好”或者一道选择题去判断 3-bit 能不能用，你测的是它没被优化过的那部分。校准分布和评测分布必须同时够长，这个判断才成立——这同时也解释了后面的分数为什么只能信一半：它们全部出自作者自己的 harness。

还有一个隐性成本要提前算进去：这是一个视觉模型，除权重之外还要挂 `mmproj-BF16` 视觉投影，[0.93 GB，三档共用](https://huggingface.co/nari-labs/Qwen3.8-27B-NariQ-GGUF)。也就是说 3.0 bpw 的真实入场价是 11.05 GB 而不是 10.12 GB。为什么重要：显存预算通常是在“刚好装得下”的边缘上做决策，这 0.93 GB 恰好能把一张 12GB 卡从“勉强能跑”推到“跑不动带视觉的配置”。

## 10.12 GB 摆在哪一格

体积账先摆清楚，全部取自官方模型卡和[同基座的 unsloth GGUF 文件表](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF)：

| 档位 | 体积 | bpw | DeepSWE 解出（/113） | TB 2.1 | AIME | decode tok/s @4K |
|---|---|---|---|---|---|---|
| BF16 | 53.8 GB | 16 | 44 | 62.6 | 97.7 | — |
| **NariQ 3.0** | **10.12 GB** | **3.00** | **47** | **63.8** | **98.3** | **88.9** |
| NariQ 2.5 | 8.44 GB | 2.50 | 46 | 60.2 | 96.5 | 99.2 |
| NariQ 2.0 | 6.75 GB | 2.00 | 33 | 52.4 | 86.0 | 111.7 |
| UD-Q2_K_XL | 9.95 GB | 2.85 | 33 | 53.0 | — | — |
| UD-Q4_K_M | 16.56 GB | ≈4.85 | — | — | — | — |

```python
# 同体积对照：把官方体积和 Table 1 的解出数放在一起，算每档还剩多少能力
files = {
    "BF16":       {"gb": 53.80, "bpw": 16.00, "deepswe": 44},
    "NariQ 3.0":  {"gb": 10.12, "bpw": 3.00,  "deepswe": 47},
    "NariQ 2.5":  {"gb": 8.44,  "bpw": 2.50,  "deepswe": 46},
    "NariQ 2.0":  {"gb": 6.75,  "bpw": 2.00,  "deepswe": 33},
    "UD-Q2_K_XL": {"gb": 9.95,  "bpw": 2.85,  "deepswe": 33},
}
base = files["BF16"]["deepswe"]

for name, f in files.items():
    pct = f["deepswe"] / base * 100
    print(f"{name:11s} {f['gb']:6.2f} GB  {f['bpw']:5.2f} bpw  "
          f"解出 {f['deepswe']:3d} 题  占 BF16 {pct:5.1f}%")
```

这张表里最该盯住的不是第一行，是第二行和第五行。UD-Q2_K_XL 是 9.95 GB、2.85 bpw，NariQ 3.0 是 10.12 GB、3.00 bpw，体积差 1.7%。在 DeepSWE v1.1（113 道工程题、每题跑 3 轮）上，前者解出 33 题，后者解出 47 题，BF16 是 44 题——后者比前者多解出 42%。按 44 题做分母是 75% 对 107%；论文给 UD-Q2_K_XL 标的是 72%，因为第三方那几行用的是 45.7 的 BF16 基线，33 ÷ 45.7 正好约等于 72%。同基座、同方向、体积几乎一样，长任务解出数差 14 道题。

为什么重要：这才是量化的真正对手。不是 FP16——53.8 GB 在消费级卡上根本不在讨论范围——而是“同样 10GB 预算下，你能拿到的另一个文件”。你放弃的不是位宽，是“用普通工具链跑这个模型”这件事本身。

有人会说 47 对 33 是运气，少数题恰好蒙对。同一份数据里还有一个更细的指标可以反驳：DeepSWE 每个任务新增测试的通过率，[BF16 是 83.0%，3.0 bpw 是 84.9%，2.0 bpw 掉到 75.7%](https://github.com/nari-labs/nari-llama.cpp)。为什么重要：解出题数会受个别题影响，测试通过率是逐条累加的，两者同向说明这不是侥幸，而是整体的解题能力确实被保住了一部分。

速度列要小心读。官方实测 [3.0 bpw 88.9 / 2.5 bpw 99.2 / 2.0 bpw 111.7 tok/s @4K](https://github.com/nari-labs/nari-llama.cpp)，位宽越低反而越快，kernel 与带宽双重受益，所以“3-bit 一定更慢”的直觉在这个模型上不成立。但这是 RTX PRO 6000 单卡单会话的数字，而那张卡恰好是唯一被专门调优过的——官方自己承认其他卡上是另一回事，原话放在第四节。

## 掉点在长，不在宽

先看断崖。官方基准表里，3-bit 的 DeepSWE 得分是 41.0，压到 2-bit 变成 28.6；AIME 从 98.3 掉到 86.0，TB 从 63.8 到 52.4。官方模型卡自己写着“2-bit 在数学与 agent 任务上更弱”。所以 3-bit 在这套方法里不是“中间档”，而是下限——再往下压就不成产品了。

再说损失为什么不均匀。论文引言给了一个可直接引用的反例：GSQ-RCO 的 3-bit 量化在 AIME 上与 BF16 只差 1 个点（96.9 对 97.7），但 DeepSWE 只解出 BF16 的 63%。数学题答得和原模型几乎一样，多步工具调用垮掉一半。

为什么重要：这就是平均分的骗局。评测集如果是短的、单轮的、有标准答案的，3-bit 看起来几乎无损；而它真正掉的那一块，恰好是你要拿它做 agent 的那一块。论文用 KL 散度和 agent benchmark 替代困惑度，理由也在这里——附录 A 说在 2.5 与 3.0 bpw 每档，现有最好文件在同位宽或更高位宽下的 KL 是 NariQ 的 1.7–4.9 倍。

读论文 [Table 1](https://arxiv.org/html/2611.26157v1) 时还有一个陷阱：表里 Unsloth 和 EXL3 那几行是第三方用 mini-swe-agent 跑的，harness 不同，只能各自跟自己的 BF16 行比，不能和 NariQ 那几行横着比。为什么重要：横向对比是这类文章最常见的错——把 88% 和 107% 放在一起，然后宣布“A 方法比 B 方法强 19 个点”，其实两边连评测脚本都不是同一份。

第二个掉点位置是长上下文，它与位宽无关、只与显存有关。官方数字：24GB 卡大约能开到 135K 上下文，跑满原生 262,144 需要约 30GB；16GB 的 RTX 5060 Ti 跑 2.5bpw 加视觉投影，在 Windows 上只能到约 47K（桌面要吃掉近 1.5GB）。12GB 卡官方没给数字，按 10.12 GB 权重加 0.93 GB 视觉投影推算——这是我的推断，不是实测——权重已经顶到天花板，上下文空间接近零。

结构上这个模型对长上下文是友好的：64 层里只有 1/4 是 Gated Attention（24 个 Q 头、4 个 KV 头），其余是 Gated DeltaNet 线性注意力，[KV cache 压力比同尺寸传统 Transformer 小得多](https://huggingface.co/Qwen/Qwen3.8-27B)。但再友好，把 10GB 权重塞进 12GB 卡，也只剩不到 2GB 可用。

第三个掉点位置是平台，最容易被忽略。官方 README 的原话是：kernel tuning 只针对 RTX PRO 6000，其他 GPU 上 NariQ 文件目前比同体积的常规 GGUF 慢，coded embedding 查找走 CPU。硬件表更窄：NVIDIA compute capability 8.0+ 配 Linux 才算 qualified，AMD / Intel 完全没有 kernel，Apple Silicon 要自己编译（M3 Pro 上 3.0bpw 约 3.8 tok/s），CPU 路径“对一个 27B 模型来说很慢”。Windows 是 experimental，二进制未签名。

## 分界线

三条可操作的分界。

**显存**：16–24GB 是甜点区。24GB 换约 135K 上下文，16GB 约 47K，这两档够跑长任务；12GB 只能算“能加载”，不算“能干活”。要跑满 262K，准备约 30GB。

**任务**：主要做长程 agent、多步工具调用、代码库级修改，3-bit 的收益是实的——体积几乎不变，长任务多解出 14 道题，这个差距值回票价。主要跑短问答、摘要、翻译，那 3-bit 相对 Q4 的优势没人测过（没有来源给过这两者之间的困惑度差，官方也没测 MMLU / HumanEval），这时选 16.56 GB 的 UD-Q4_K_M，或者直接换个更小的模型，都比折腾 fork 省事。

**平台**：手上是 RTX PRO 6000 或 H100 / L4 / A10G 且跑 Linux，值得试；AMD、Intel、Mac、Windows 桌面，现在不值得——你付出的是被一张卡绑死的代价。

三个必须承认的代价。一是 9 stars 的私有 fork，上游落后 248 个提交，等于把工具链锁在一家公司的一个分支上。二是全部 benchmark 出自作者自己的 harness，第三方尚未复现，而论文 Table 1 自己就展示了 harness 不同会让 BF16 基线从 44 变成 45.7、TB 从 62.6 变成 59.15——这套数字自带约 ±5% 的噪声，107% 不该读成“比 BF16 更强”。三是截至发稿我没检索到公开的失败案例报告（仓库 47 likes、发布 4 天），这既不能证明它稳，也不能证明它不稳。

替代路线给一条：如果你的目标是“在 16GB 卡上跑一个能干活的模型”，现在更稳的选择是 UD-Q3_K_XL（13.15 GB、3.80 bpw）这类常规量化配标准 llama.cpp，代价是长任务解出数从 47 题降到 42.5 题，收益是你能用上游、能换卡、能在 Mac 上跑。

一句话收尾：10GB 的 27B 不是免费午餐，也不是降智玩具。它是一个已被验证有效、但把自己焊死在特定硬件和特定工具链上的方案——值得试，不值得现在就全押。
