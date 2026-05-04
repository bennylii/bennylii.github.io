---
title: DeepSeek V4-Pro 到底跑在什么芯片上？
date: 2026-05-05 04:00:00
tags: [deepseek, gpu, nvidia, ascend, huawei, ai, training, hardware]
---

# DeepSeek V4-Pro 到底跑在什么芯片上？——基于公开信息的推演与还原

> 本文所有推论基于公开信息交叉比对。不做 insider 爆料，不渲染阴谋论。每一条标注可靠程度，承认知识盲区。

---

## 一、"B200 NVL144"并不存在——先厘清几个常见混淆

在讨论 DeepSeek V4 的训练硬件前，需要先澄清一个被反复提及但实际不存在的配置名。

**B200/Blackwell 系列只有 NVL72。** NVL72 是一个整机架级系统：72 个 B200 GPU（通过 NVLink 5 全互联）+ 36 个 Grace CPU，封装在一个高约 2 米、重 1.2-1.5 吨的液冷机柜中。单柜 GPU 显存总计约 13.5 TB HBM3e，FP8 峰值算力约 324 PFLOPS，功耗约 120kW。行业报价约 300 万美元/柜。

**NVL144 是下一代 Vera Rubin 架构的配置**，预计 2026 年下半年才开始量产出货，目前没有公开定价。

与 NVL72 整机架平行的，还有 **HGX B200 8 卡服务器**——标准 8U 机架式，8 张 B200 GPU 通过 NVLink Switch 全互联，可风冷，单节点功耗约 10-12kW。但 NVLink 域仅限于这 8 张卡，跨节点必须走 InfiniBand。

两者的关键差异，后文会反复用到：

| | HGX B200 | GB200 NVL72 |
|---|---|---|
| GPU 数 | 8 | 72 |
| NVLink 域 | 8 GPU | 72 GPU |
| 域内带宽 | 1.8 TB/s/GPU | 1.8 TB/s/GPU |
| 跨节点方式 | InfiniBand | InfiniBand（域间） |
| 物理形态 | 标准服务器 | 整机架（1.5 吨） |
| 冷却 | 风冷/液冷 | 必须液冷 |

**已确认事实** | **来源**：[NVIDIA GB200 NVL72 官方规格](https://www.nvidia.com/en-us/data-center/gb200-nvl72/)、[Modal: Blackwell 产品线解析（HGX vs GB200 vs DGX）](https://modal.com/blog/nvidia-blackwell)、[Spheron 架构与定价指南](https://www.spheron.network/blog/nvidia-gb200-nvl72-guide/)、[SemiAnalysis Blackwell Perf/TCO 分析](https://semianalysis.com/2025/08/20/h100-vs-gb200-nvl72-training-benchmarks/)

---

## 二、V3 说清楚了，V4 为什么沉默了？

2024 年底发布的 DeepSeek-V3 技术报告，[正文第一页](https://arxiv.org/abs/2412.19437)就写着："DeepSeek-V3 is trained on a cluster of 2048 NVIDIA H800 GPUs." 这是 V3 最具传播力的故事之一——只用 2048 张被阉割了互联带宽的 H800，以约 550 万美元的算力成本，训出了对标 GPT-4o 的模型。

**V4 的技术报告打破了这一传统。**

2026 年 4 月 24 日随 V4 发布的[技术报告](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/DeepSeek_V4.pdf)，仔细读过的人会发现一个反常点：报告提到了"华为"1 次、"NVIDIA"3 次、"GPU"14 次，但**没有一句话说明训练用的是什么 GPU**。

唯一与硬件相关的句子是第 3.1 节中的：

> "We validated the fine-grained EP scheme on both NVIDIA GPUs and Huawei NPUs platforms."

这里是"验证"（validated），不是"训练"（trained）。验证细粒度 EP 方案在两个平台上都能跑通，和不披露训练集群的规模和型号，是两回事。

虎嗅在 V4 发布当天的深度分析中注意到了这一点：

> "DeepSeek V4 技术文件披露了很多训练细节，但是不包括训练硬件（显卡）。……相比之下，V3 技术文件在一开始就宣布是由英伟达 H800 和 A100 训练出来的。"

对于一个素以"低成本训练"为技术品牌的公司来说，放弃这个叙事显然不是一个随意的选择。**如果训练硬件是合规采购的，没有理由隐藏。刻意的沉默本身就是信号。**

**可靠程度：高** | **来源**：[DeepSeek-V3 技术报告](https://arxiv.org/abs/2412.19437)、[DeepSeek-V4 技术报告](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/DeepSeek_V4.pdf)、[InfoQ 分析](https://www.infoq.cn/article/wUUPEzvNajcaVN0k7HPF)、[观察者网报道](https://www.guancha.cn/economy/2026_04_24_814819.shtml)

---

## 三、美方指控与美国执法行动

2026 年 4 月 10 日，V4 发布前两周，一位特朗普政府高级官员在[记者吹风会上](https://www.reuters.com/world/china/chinas-deepseek-trained-ai-model-nvidias-best-chip-despite-us-ban-official-says-2026-02-24/)明确表示：

> "数千块 NVIDIA Blackwell GPU 在内蒙古的一个数据中心运行了数月，被用于训练 DeepSeek V4。"

The Information 随后发表了更详细的调查报道，指出走私者偏好 **HGX B200 8 卡服务器**而非 GB200 NVL72，因为前者尺寸更小，可以拆箱、伪装运输。报道称这些服务器通过第三国（新加坡、马来西亚等）转口，最终进入中国境内。

这不是孤立的指控。在此之前，美国及其盟友已经在供应链节点上进行了多次执法：

**2025 年 12 月——Operation Gatekeeper（美国 DOJ）：**
- 德州男子 Alan Hao Hsu 及其公司承认走私价值 **1.6 亿美元**的 NVIDIA H100/H200 GPU 到中国
- 同伙龚凡跃（纽约布鲁克林，中国公民）被控撕掉 NVIDIA 标签、贴上假标签"SANDKYAN"、伪报为"适配器模块"
- 同伙袁本林（弗吉尼亚州 IT 公司 CEO）向 FBI 卧底支付 100 万美元"赎金"试图取回被扣芯片，面临最高 **20 年监禁**
- 这是美国司法部首例 AI 芯片走私定罪

**2025 年 2-3 月——新加坡警方：**
- 突击搜查 22 个地点，逮捕 9 人，指控 3 人欺诈服务器供应商
- 涉及金额约 **3.9 亿美元**（约 28 亿人民币）的 NVIDIA 服务器
- 芯片路线：新加坡 → 马来西亚 → 中国大陆
- 新加坡律政部长公开承认：NVIDIA 在新加坡的销售额中仅 **约 1%** 真正留在当地

DeepSeek 对此的回应是：**既不承认，也不否认。** 这种沉默本身值得注意——如果指控不实，一个正在寻求国际融资（据 The Information 报道，腾讯和阿里巴巴正洽谈投资，估值超 200 亿美元）的公司通常会有动力澄清。

**可靠程度：中高**（指控来自美方官员，有明确细节但无直接物证公开；过往执法案例为独立核实） | **来源**：[Reuters 独家报道（2026.02.24）](https://www.reuters.com/world/china/chinas-deepseek-trained-ai-model-nvidias-best-chip-despite-us-ban-official-says-2026-02-24/)、[Reuters: DeepSeek 拒绝向 NVIDIA/AMD 提供 V4 早期访问权限（2026.02.25）](https://www.reuters.com/world/china/deepseek-withholds-latest-ai-model-us-chipmakers-including-nvidia-sources-say-2026-02-25/)、[The Information 走私调查](https://x.com/theinformation/status/1999150974221337056)、[Tom's Hardware: NVIDIA 否认走私报道](https://www.tomshardware.com/tech-industry/artificial-intelligence/nvidia-decries-far-fetched-reports-of-smuggling-in-face-of-deepseek-training-reports-unnamed-sources-claim-chinese-company-is-involved-in-blackwell-smuggling-ring)、[美国 DOJ Operation Gatekeeper 新闻稿](https://www.justice.gov/opa/pr/us-authorities-shut-down-major-china-linked-ai-tech-smuggling-network)、[CNBC 走私调查深度报道](https://www.cnbc.com/2025/12/31/160-million-worth-export-controlled-nvidia-gpus-allegedly-smuggled-to-china.html)、[新加坡警方/CNA 报道](https://www.reuters.com/technology/singapore-charges-three-with-fraud-that-media-link-nvidia-chips-2025-02-28/)、[新加坡 3.9 亿美元案](https://www.reuters.com/technology/servers-used-singapore-fraud-case-may-contain-nvidia-chips-minister-says-2025-03-03/)

---

## 四、为什么走私目标是 HGX B200 而非 NVL72？

把 NVL72 整机架从第三国偷运入境，在操作层面几乎不成立：

- **体积**：高 2 米，重 1.5 吨，需要专业吊装和货运
- **功耗**：120kW，相当于约 100 户家庭用电，普通机房完全无法启动
- **冷却**：液冷，需要配套 CDU（冷量分配单元）、冷板、管路
- **安装**：内部 5000+ 根铜缆，需要 NVIDIA 认证工程师部署调试

这不是"贵一点的显卡"，而是数据中心级的重型设备。

HGX B200 则完全不同。标准 8U 机架式服务器，可风冷，可以装箱运输，外形与普通企业级服务器无异。8 卡是一台完整服务器的最小可行单元，而每台服务器内部有独立的 NVLink 域——不需要 NVL72 那种整柜 NVLink 背板。

**但代价是互联带宽。** 这是下一章要讨论的核心问题。

**可靠程度：高** | **来源**：[NVIDIA GB200 NVL72 白皮书](https://www.nvidia.com/en-us/data-center/gb200-nvl72/)、[NVIDIA Developer Blog: GB200 NVL72](https://developer.nvidia.com/blog/nvidia-gb200-nvl72-delivers-trillion-parameter-llm-training-and-real-time-inference/)、[Tom's Hardware: 走私分析](https://www.tomshardware.com/tech-industry/artificial-intelligence/nvidia-decries-far-fetched-reports-of-smuggling-in-face-of-deepseek-training-reports-unnamed-sources-claim-chinese-company-is-involved-in-blackwell-smuggling-ring)、[The Information 走私调查](https://x.com/theinformation/status/1999150974221337056)

---

## 五、8 个 GPU 的孤岛——HGX B200 的互联困境

HGX B200 节点内有 8 个 GPU，通过 NVLink 5 全互联，每 GPU 1.8 TB/s 双向带宽，聚合 14.4 TB/s。这是它的"舒适区"。

一旦跨出这 8 个 GPU 的 NVLink 域，通信就必须走 InfiniBand。即使用 8 张 ConnectX-7 NDR 网卡（每 GPU 独占一个 NIC，GPUDirect RDMA），单链路也只有 50 GB/s。对应 NVLink 域内的 1800 GB/s，**跨节点带宽是域内的 1/36**。

```
┌─────────────────────────────────────────────────────────┐
│  HGX B200 节点内（8 GPU，NVLink 5）                      │
│  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐ │
│  │GPU├──┤GPU├──┤GPU├──┤GPU├──┤GPU├──┤GPU├──┤GPU├──┤GPU│ │
│  │ 0 │  │ 1 │  │ 2 │  │ 3 │  │ 4 │  │ 5 │  │ 6 │  │ 7 │ │
│  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘ │
│  全互联：每 GPU 1.8 TB/s，聚合 14.4 TB/s                │
└─────────────────────────────────────────────────────────┘
                         │
                   ┌─────┴─────┐
                   │ InfiniBand │  ← 每条链路 50 GB/s
                   │  NDR400    │     聚合 400 GB/s（8 NIC）
                   └─────┬─────┘
                         │
┌─────────────────────────────────────────────────────────┐
│  另一个 HGX B200 节点（同样 8 GPU）                       │
│  跨节点带宽仅为域内的 1/36                               │
└─────────────────────────────────────────────────────────┘
```

对于 MoE 模型训练，这意味着什么？核心通信模式是 **all-to-all**（专家并行中的 token dispatch 和 combine）。在 8 GPU 域内，all-to-all 走 NVLink，延迟在亚微秒级。一旦 EP 扩展到跨节点，每一轮 all-to-all 都要经过 InfiniBand——微秒级延迟，带宽缩水 36 倍。

DeepSeek 之所以能在这套硬件上训练 MoE，靠的是开源项目 **[DeepEP](https://github.com/deepseek-ai/DeepEP)**——专门为 MoE 跨节点通信优化的库：

- **高吞吐模式**：利用 NVSHMEM 通过 InfiniBand 做跨节点 all-to-all，FP8 压缩通信量减半
- **低延迟模式**：节点内利用 NVLink 做极低延迟 dispatch/combine
- **零 SM 占用**：通信由专用硬件处理，不占用 GPU 计算单元 → 可以与矩阵乘法**完全重叠**
- **V2**：统一 ElasticBuffer 接口，框架层面不需要区分通信走的是 NVLink 还是 InfiniBand

配合分层并行策略（TP 不跨节点、尽量 EP 保持在节点内、DP/ZeRO 的梯度同步可与反向传播重叠），HGX B200 集群可以做到 MoE 训练的 MFU 在 **30-40%** 左右。作为对比，GB200 NVL72（72 GPU 同域）的 MFU 可达 50%+。

**这 10-20 个百分点的 MFU 差距，根源就是那 36 倍的跨节点带宽鸿沟。** DeepSeek 选择 HGX B200 不是因为它好，而是因为它能搞到——NVL72 既搞不到，也运不进来。这是典型的"有什么用什么"，不是"什么好用用什么"。

旁注：作为反衬，xAI 的 Colossus 超级计算机坐拥约 55 万张合法采购的 NVIDIA GPU（全球最大 AI 集群），但 2026 年 4 月内部备忘录显示其 MFU 仅约 11%——xAI 总裁亲口形容为 "embarrassingly low"。DeepSeek 在被制裁、走灰色渠道的受限硬件上做到 30-40% MFU，本身就说明了工程能力的差距。硬件规模不等于训练效率。[[Business Insider](https://www.businessinsider.com/elon-musk-xai-compute-cursor-ai-model-training-2026-4)]

**可靠程度：高**（架构部分是公开规格，MFU 范围来自 SemiAnalysis 和 LMSYS 的独立基准测试） | **来源**：[DeepEP GitHub](https://github.com/deepseek-ai/DeepEP)、[SemiAnalysis: H100 vs GB200 NVL72 训练基准](https://semianalysis.com/2025/08/20/h100-vs-gb200-nvl72-training-benchmarks/)、[LMSYS: DeepSeek on GB200 NVL72 Part II](https://lmsys.org/blog/2025-09-25-gb200-part-2/)、[Spheron 架构分析](https://www.spheron.network/blog/nvidia-gb200-nvl72-guide/)、[LMSYS: SGLang + NVIDIA InferenceMAX on GB200](https://lmsys.org/blog/2025-10-14-sa-inference-max/)

---

## 六、中国海关维度：抓不抓 GPU 走私？

一个重要的法律区分贯穿了整个讨论：**中国海关执行的是《海关法》和关税制度，而非美国 EAR 出口管制。** 两者是完全不同的法律框架，由完全不同的主体执行。

如果一批 GPU/服务器：
- 如实申报品名、规格、价格
- 正常缴纳进口关税和增值税
- 持有符合中国国内法规的进口批文

那么中国海关没有法律依据仅仅因为"这是美国出口管制清单上的物品"而拒绝放行。中国的执法部门没有义务替美国执行单边出口管制。这也是为什么走私案件的主要执法节点出现在供应链的**上游**（新加坡、美国的中间商、物流商和金融机构），而不是中国境内的终端用户。

但这不等于"无风险"。美国 EAR 具有长臂管辖效力，可以追究交易链条上任何涉及美国技术、美国实体或美元结算的节点。具体的法律后果取决于目标企业在不在实体清单上：

| 目标企业类型 | 核心风险 |
|---|---|
| **已在实体清单上**（如华为） | 供应链节点被逐个切断、海外资金通道被 SWIFT/CHIPS 排除、第三方合作伙伴因担心连带制裁而终止合作 |
| **不在实体清单上**（如多数民营 AI 公司） | 被 BIS 列入实体清单，失去所有美国技术来源；中间商和物流方面临美国 DOJ 刑事起诉 |

对于 DeepSeek 而言，如果 V4 确实使用了通过灰色渠道获取的 NVIDIA 硬件，它面临的核心风险是第二个——被加入实体清单。同时，如果 DeepSeek 正在寻求国际融资和海外云服务合作，这种风险会更复杂。

需要澄清一个常见的误解：**如果大厂有国家意志的支持、有特批进口手续，那就已经不是"走私"了。** "走私"的法律定义是逃避海关监管、偷逃关税。如果正常报关、持证进口，在中国的法律框架下它就是合规进口——即使这批货物在美国法律下属于非法出口。两个主权国家的法律冲突，不等于进口方在两个国家都违法。

**可靠程度：高**（法律框架为公开信息；具体执法案例来自公开报道） | **来源**：《中华人民共和国海关法》、[美国 BIS EAR 指南](https://www.bis.doc.gov/index.php/regulations/export-administration-regulations-ear)、[美国 DOJ Operation Gatekeeper 案卷](https://www.justice.gov/opa/pr/us-authorities-shut-down-major-china-linked-ai-tech-smuggling-network)、[CNBC 走私调查](https://www.cnbc.com/2025/12/31/160-million-worth-export-controlled-nvidia-gpus-allegedly-smuggled-to-china.html)

---

## 七、昇腾 950 在 V4 项目中的真实角色

这是最容易被误读的部分。昇腾 950 在 DeepSeek V4 项目中的参与是真实的，但其范围被很多人夸大了。

### 先说硬件：950 系列的两种芯片

昇腾 950 系列基于同一块 Ascend 950 Die，分成两个变体，面向不同场景：

| | 950PR | 950DT |
|---|---|---|
| **定位** | 推理（Prefill）+ 推荐 | 推理（Decode）+ **训练** |
| **HBM** | 112GB（HiBL 1.0） | 144GB（HiZQ 2.0） |
| **HBM 带宽** | ~3.2 TB/s | ~4 TB/s |
| **互联** | — | 2 TB/s |
| **FP8 算力** | — | 1 PFLOPS |
| **FP4 算力** | 1.56 PFLOPS | 2 PFLOPS |
| **量产时间** | 2026 年 3 月 | **2026 年 Q4** |
| **定价** | ~7 万人民币 | 未公布 |

### 时间线是铁证

DeepSeek V4-Pro 发布于 **2026 年 4 月 24 日**。按照 V4 技术报告披露的预训练数据量（33T tokens）和 V3 的训练速度外推，V4 的 Pretrain 至少需要 2-3 个月连续运行。这意味着训练必须在 **2026 年 1 月之前**开始。

而 950DT（训练芯片）的预期量产时间是 **2026 年 Q4**——在 V4 完成训练之后约半年。所以：

> **V4-Pro 的完整 Pretrain 不可能在 Ascend 950DT 上完成。时间线对不上。**

### 升腾在 V4 中已确认的参与

以下是有公开证据的：

1. **推理部署**：华为和 DeepSeek 双方均确认，昇腾 950 超节点 Day-0 支持 V4 推理。实测 V4-Pro TPOT ~20ms，V4-Flash TPOT ~10ms（8K 输入场景）
2. **续训练/微调**：V4-Flash 已在 Atlas A3 和 950DT 上跑通续训练流程。华为开源了参考实现
3. **EP 方案验证**：V4 技术报告明确写了 EP 方案在两个平台上都做了验证
4. **算子适配**：CANN 对 V4 的 CSA（压缩稀疏注意力）、HCA（混合上下文注意力）、MoE 路由等模块做了原生适配

### 昇腾目前不能做的

- **完整 Pretrain**：没有公开证据。950DT 还没量产
- **RL/后训练**：没有公开信息。RL 对显存和通信的要求比 Pretrain 更高

DeepSeek 官方在 [API 定价页面](https://api-docs.deepseek.com/news/news260424)写的"预计下半年昇腾 950 超节点批量上市后价格下调"，上下文是**推理 API 的价格和吞吐**——说的是用更多国产芯片扩容推理服务，降低单位成本。这句话常被误解为"DeepSeek 在用昇腾训练"，但它说的其实是推理。

CSDN 昇腾社区的一篇文章总结得很准确：

> "先跑通推理，再完成续训练适配，最后攻克最难的完整预训练。"

目前国产算力在 V4 项目中的进度是**第二步**。关于 CM384 和盘古 Ultra 在 Ascend 上跑通完整 MoE 训练的意义（以及它们对 V4 的适用边界），详见下一章。

**可靠程度：高**（时间线为公开信息；规格为官方发布） | **来源**：华为全联接大会 2025、[上海证券报（2026.04.24）](https://www.stcn.com/article/detail/3803257.html)、[每经网昇腾分析](https://www.nbd.com.cn/articles/2026-04-24/4358607.html)、[DeepSeek API 定价页面](https://api-docs.deepseek.com/news/news260424)、[V4 技术报告](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/DeepSeek_V4.pdf)

---

## 八、CM384 与昇腾训练生态：已经跑通了什么？

上一节讨论的是昇腾 950 系列在 V4 项目中的角色——推理和续训练已确认，Pretrain 不可能。但可能有人会问：**华为的 CloudMatrix 384（CM384）不是早就发布了？盘古大模型不是在 Ascend 上训练的吗？** 这些成就和 V4 的关系需要仔细区分，既不能夸大，也不应低估。

### CM384 不是推理专用机

华为在 2025 WAIC 上发布的 CM384 是一个关键参考点。384 张 Ascend 910 NPU + 192 颗鲲鹏 CPU，通过 UB（Unified Bus）非阻塞全互联组网，16 个机柜（12 计算 + 4 通信）。[CM384 论文](https://arxiv.org/abs/2506.12708)明确写了系统的定位是：

> "enables converged execution of LLM inference, training, simulation, and analytics workloads"

**训练和推理融合平台**，不是推理专用。论文进一步指出 "support for distributed training and inference using RDMA-compliant frameworks"——训练能力在架构设计阶段就已内置。

CM384 基于的芯片是 **Ascend 910**（不是 950 系列），单卡 HBM 约 128 GB，FP16/BF16/INT8（无原生 FP8），算力明显低于 B200。但它在互联上有独特的架构选择——UB 非阻塞交换：

- **跨节点带宽衰减 <3%**：不同于 HGX B200 跨节点走 InfiniBand 带宽缩水 36 倍，CM384 的 UB 交换芯片使跨节点 all-to-all 几乎不衰减
- **延迟增加 <1μs**：跨节点通信延迟在亚微秒级

这和前文讨论的 HGX B200 形成鲜明对比。CM384 的 UB 互联在架构思路上更接近 NVLink 域内通信，而非 InfiniBand。对 MoE 的 all-to-all 通信模式而言，这种互联拓扑有天然优势。

### 盘古 Ultra：在 Ascend 上跑完完整训练的存在性证明

关于"昇腾能不能做训练"，最硬的证据不是 PPT 参数，而是实际跑出来的训练结果：

**盘古 Ultra MoE**（718B 总参 / 39B active / 256 experts）——在 **6,000 张 Ascend NPU** 上完成完整训练，[技术报告发表于 arxiv（2505.04519）](https://arxiv.org/abs/2505.04519)。关键数据：
- **MFU 30.0%**，训练吞吐 1.46M tokens/s（6K 卡配置）
- 使用 MindSpeed 平台 + Megatron 框架
- 跑通全部训练阶段：**Pretrain → SFT → RLHF/DPO**
- 技术栈：CANN + HCCL（集合通信，对标 NCCL）+ torch_npu adapter

**盘古 Ultra dense**：12.3T tokens，在 **8,192 张 Ascend NPU** 上完成，[技术报告发表于 arxiv（2504.07866）](https://arxiv.org/abs/2504.07866)。

**盘古 Embedded**：华为官方明确写了 "reinforcement learning on Ascend clusters, optimized by a latency-tolerant scheduler that combines stale synchronous parallelism with prioritized data queues"——RL 后训练在 Ascend 集群上**生产级跑通**，不是"理论可行"。

这意味着什么？**CANN 的训练软件栈（HCCL 集合通信、MindSpeed 分布式训练框架、Megatron 3D 并行适配、RL 调度器）已经在 6K-8K 卡规模上经过了生产验证。** 这不是 Demo，不是 POC，是产出过完整 MoE 模型的工程实践。

### 算力与显存：CM384 能跑什么，不能跑什么

前面定性说了 CM384/Ascend 910 对 V4-Pro 不够，这里给出定量计算。

#### V4-Pro Pretrain on CM384：算力是硬墙

Ascend 910 单卡 FP16 算力约 320 TFLOPS。CM384 共 384 卡，理论 FP16 总算力约 **123 PFLOPS**。无原生 FP8——所有训练必须在 FP16/BF16 下进行。

V4-Pro Pretrain 总计算量（基于 MoE active params 估算）：
- 49B active × 6 FLOPs/token（前向+反向）≈ 294 GFLOPs/token
- 33T tokens → 33 × 10¹² × 294 × 10⁹ ≈ **9.7 × 10²⁴ FLOPs**

在 CM384 上的理论训练时间（假设 MFU 30%，实际上 Ascend 910 跑 MoE 的 MFU 可能更低）：

| MFU | 有效算力 | 训练时间 |
|---|---|---|
| 30%（乐观） | 37 PFLOPS | ~8.3 年 |
| 15%（实际可能） | 18.5 PFLOPS | ~16.6 年 |

即使扩展到盘古 Ultra 的 6,000 卡配置（~1,920 PFLOPS FP16），MFU 30% 下仍需约 **5-6 个月**——前提是 1.6T 参数跨 6K 卡的 all-to-all 通信能跑通，这在 Ascend 910 + CANN 上远未经过验证。盘古 Ultra 自己只有 718B/39B active，已经把 6K 卡推到 MFU 30% 了。

**结论：CM384 跑 V4-Pro Pretrain 完全不可行。** 瓶颈首先在算力（差两个数量级），其次在通信规模。

#### V4-Pro RL on CM384：长上下文的 KV cache 是杀手

RL 的显存需求和 Pretrain 不同。RL 需要：
- 所有 expert 常驻显存（rollout 阶段任意 token 可能路由到任意 expert）
- 参考模型（用于 KL 散度约束，通常和训练模型同等大小）
- **长序列 rollout 的 KV cache**（PPO/GRPO 需要完整的注意力状态）

V4-Pro 在 FP16 + Adam + ZeRO-3 下的基础显存（不含 KV cache）：

| 组件 | FP16 占用 |
|---|---|
| 模型权重（1.6T params） | 1.6T × 2B = 3.2 TB |
| 优化器状态（fp32 master + momentum + variance） | 1.6T × 12B = 19.2 TB |
| 梯度 | 1.6T × 2B = 3.2 TB |
| 参考模型（可卸载至 CPU） | 0–3.2 TB |
| **合计（不含参考模型）** | **25.6 TB** |

CM384 总 HBM = 384 × 128 GB = **49 TB**。ZeRO-3 分片后单卡 ~67 GB，还剩下约 60 GB headroom。到这里似乎还行——但接下来才是真正的瓶颈。

**KV cache 爆炸。** V4-Pro 主打 1M token 上下文。单条 1M token 序列的 KV cache：
- 假设 60 层、128 KV heads、128 head_dim：2 × 60 × 128 × 128 × 1M × 2 bytes ≈ **3.9 TB/序列**
- GRPO 通常需要 4-8 条并行 rollout + 对应的参考模型前向 → 保守估计 **15-60 TB 额外显存**
- 即使启用 MLA 压缩（DeepSeek 的 latent attention 将 KV 压缩到 latent space），对 1M 上下文，压缩后的 KV cache 仍在 **数百 GB 到 TB 级别**

加上 KV cache 后，总显存需求远超 49 TB。缩小 rollout batch 或序列长度会降低 RL 的样本效率，但这只是折中方案——不是从根本上解决问题。

**结论：CM384 不支持 V4-Pro（1M 上下文）的 RL。** 瓶颈主要在长上下文 KV cache，其次在全量 1.6T 参数的优化器状态。

#### V4-Flash RL on CM384：在可行范围内

把问题缩小到 V4-Flash，情况不同：

V4-Flash：284B 总参 / 13B active / 256 experts，上下文窗口较短，定位是轻量级模型。

基础显存（FP16 + Adam + ZeRO-3）：

| 组件 | FP16 占用 |
|---|---|
| 模型权重 | 284B × 2B = 568 GB |
| 优化器状态 | 284B × 12B = 3.4 TB |
| 梯度 | 284B × 2B = 568 GB |
| **合计（不含参考模型/KV cache）** | **~4.5 TB** |

CM384 总 HBM 49 TB。ZeRO-3 分片后单卡 ~12 GB。加上 KV cache 和参考模型（Flash 的上下文窗口远小于 Pro，KV cache 需求小一个数量级），单卡 128 GB 有充足的裕量。

**384 卡对 V4-Flash RL 来说可行。** 扩展到 1,024 卡以上可支撑更大 batch 和更长的上下文实验。这与盘古 Ultra（718B/39B active，6K 卡）的经验一致。

### 这解释了为什么 DeepSeek 和华为在合作

把 CM384/盘古的视角加回整体画面，国产芯片在 V4 项目中的实际角色分层：

- **推理部署**（昇腾 950PR）——已确认，Day-0 上线
- **续训练/微调**（昇腾 950DT + Ascend 910 集群）——V4-Flash 已跑通
- **V4-Flash 级别的 RL 后训练**——CM384 级别硬件在理论上有可行性（盘古 Ultra 提供了软件栈的存在性证明），V4 是否实操无公开证据
- **V4-Pro 级别的 RL 后训练**——CM384/Ascend 910 级别不可行（长上下文 KV cache 爆炸 + 全量 1.6T 优化器状态太大）。需 950DT 或更高
- **V4-Pro 完整 Pretrain**——CM384 完全不可行（算力差两个数量级）。需 950DT/960 SuperPoD，2027 年前不太可能

CSDN 昇腾社区的那句话总结得很准："先跑通推理，再完成续训练适配，最后攻克最难的完整预训练。"国产算力目前在**第二步末期到第三步初期**。盘古 Ultra 证明了完整训练在 Ascend 上是可能的——但那是盘古，不是 DeepSeek。两个模型的架构差异（attention 类型、MoE 路由策略、训练超参）意味着 DeepSeek V4 的 Ascend 适配仍需大量工程工作。

**可靠程度：中高** | **来源**：[CM384 论文（arxiv 2506.12708）](https://arxiv.org/abs/2506.12708)、盘古 Ultra 技术报告、华为全联接大会 2025、[华为 Cloud ModelArts 文档](https://www.huaweicloud.com/intl/en-us/solution/modelarts.html)

---

## 九、如果用 Ascend 950DT 做 Pretrain，需要多少？

虽然 V4 没用 950DT 做训练，但从技术角度看，未来是否可行？

对比三款芯片的训练规格：

| | H800（V3 使用） | B200 | Ascend 950DT |
|---|---|---|---|
| HBM | 80 GB | 192 GB | 144 GB |
| HBM 带宽 | 3.35 TB/s | 8 TB/s | ~4 TB/s |
| 互联带宽 | ~0.4 TB/s | 1.8 TB/s | 2 TB/s |
| FP8 算力 | ~4 PFLOPS | ~4.5 PFLOPS | 1 PFLOPS |

950DT 的画像很清楚：**显存和互联是强项（相对 H800），算力是短板（仅 H800 的 1/4）。** 这对 MoE 训练来说是一个有趣的组合——MoE 是通信和显存密集型的，纯算力反而没那么瓶颈。

以 V3 的公开数据为基准做 FLOPs 外推：

- V3：2048 张 H800，约 55 天 Pretrain（14.8T tokens）
- V4-Pro：1.6T/49B active，33T tokens
- 激活参数比 1.32×，数据量比 2.23×，**总计算量约 V3 的 3 倍**（考虑到总参数增加带来的额外通信和收敛难度，实际在 3-4×）
- 950DT 单卡有效速度约为 H800 的 **35-50%**（算力差距大但显存和互联有补偿）

反推不同时间目标下的集群规模：

| 目标训练时间 | 所需 950DT 数量 | 总 HBM |
|---|---|---|
| 6-9 个月（保守） | 4,000–6,000 张 | 0.6–0.9 PB |
| 3-4 个月（实用） | 6,000–10,000 张 | 0.9–1.4 PB |
| 1.5-2 个月（激进） | 12,000–20,000 张 | 1.7–2.9 PB |

Atlas 950 SuperPoD 标配 8192 卡，总 HBM 约 1.15 PB。对于 3-4 个月的训练目标，一个满配 SuperPoD 基本足够。

**但真正的瓶颈不在硬件容量。** 最大的不确定性是 **CANN 软件栈在大规模分布式训练中的成熟度**——CANN Next 虽然在推理侧表现出色（90%+ CUDA 代码可一键迁移），但数千卡的 3D 并行训练涉及大量底层通信原语（all-to-all、all-reduce、all-gather）、断点续训（checkpoint/restart）、故障恢复（MTBF 管理），这些在 Ascend 生态中仍处于快速迭代但尚未大规模验证的阶段。

合理预期：**2026 Q4 起 950DT 开始支撑中等规模的 MoE 训练，2027 年 960 系列（FP8 2 PFLOPS）有望实现万亿参数级模型的完整国产 Pretrain。**

**可靠程度：中**（硬件规格为公开信息；训练时间推算为理论外推，实际 MFU 和软件成熟度是核心未知变量） | **来源**：行业规格汇总、基于 V3 FLOPs 的工程估算

---

## 十、综合还原：最可能的技术路线图

将以上所有信息拼在一起，以下是对 DeepSeek V4-Pro 训练硬件的最可能还原：

```
2024 H2  │  DeepSeek 开始设计 V4 架构
         │  目标：1.6T 总参 / 49B 激活 / 1M 上下文的 MoE 模型
         │
2025 H1  │  V4 架构冻结。合规渠道只能买到 H20（训练 1.6T 模型性能不足）
         │  开始通过第三国渠道获取 HGX B200（8 卡服务器）
         │
2025 H2  │  HGX B200 集群在内蒙等地部署到位
         │  配套 DeepEP + InfiniBand 解决跨节点通信
         │  开始 V4-Pro Pretrain（33T tokens，预计 3-4 个月连续运行）
         │  同期与华为合作 CANN 适配准备
         │
2026.01  │  V3.2-Speciale 发布（可能是 V4 训练过程中的中间产物/能力验证）
         │
2026.03  │  昇腾 950PR 量产。DeepSeek 拿到早期样片做推理适配
         │
2026.04  │  美方官员公开指控 V4 使用走私 Blackwell 训练（4 月 10 日）
         │  V4-Pro / V4-Flash 发布并开源（4 月 24 日）
         │  技术报告中训练硬件刻意留白
         │  昇腾 950 系列 Day-0 支持推理部署
         │
2026 H2  │  昇腾 950DT 量产
         │  Atlas 950 SuperPoD 批量交付
         │  DeepSeek 将 V4 推理服务大规模扩容到国产芯片
         │  续训练/微调工作流向 Ascend 平台
         │
2027     │  昇腾 960 系列（FP8 2 PFLOPS，Atlas 960 SuperPoD 15488 卡）
         │  有望实现万亿参数级模型的完整国产 Pretrain
```

---

## 结语：关于方法论的说明

这篇文章的核心工作不是"爆料"，而是**交叉比对**：

- DeepSeek V4 技术报告的**沉默**（隐藏训练硬件）vs V3 的**透明**（主动公开 2048 H800）
- 美方官员的**明确指控**（内蒙、Blackwell、数月）vs DeepSeek 的**不回应**
- 美国 DOJ 和新加坡警方的**实际行动**（Gatekeeper、3.9 亿美元案件）vs 走私网络的**客观存在**
- 昇腾 950DT 的**量产时间**（2026 Q4）vs V4 的**发布时间**（2026.04）
- HGX B200 的**物理可运输性**（8U 服务器）vs NVL72 的**物理不可运输性**（1.5 吨整机架）

每一条单独看，都可以有别的解释。但当这些线索指向同一个方向时——**V4-Pro 的 Pretrain 运行在通过非正规渠道获取的 HGX B200 集群上**——这个推论就具备了相当的可信度。

同时也必须承认：**没有确凿的直接证据。** DeepSeek 没有承认，训练日志没有公开，硬件采购记录不可获取。一切仍停留在"合理推演"层面。如果未来有新的公开信息推翻了这一推论，也完全正常——这正是基于公开信息做分析的固有局限。

### 附：关键来源一览

| 类型 | 来源 |
|---|---|
| V4 技术报告 | [DeepSeek, "DeepSeek-V4 Technical Report", April 2026](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/DeepSeek_V4.pdf) |
| V3 技术报告 | [DeepSeek, "DeepSeek-V3 Technical Report", December 2024](https://arxiv.org/abs/2412.19437) |
| 美方指控 | [Reuters, Feb 24, 2026](https://www.reuters.com/world/china/chinas-deepseek-trained-ai-model-nvidias-best-chip-despite-us-ban-official-says-2026-02-24/) |
| DeepSeek 拒绝对 NVIDIA/AMD 提供早期访问 | [Reuters, Feb 25, 2026](https://www.reuters.com/world/china/deepseek-withholds-latest-ai-model-us-chipmakers-including-nvidia-sources-say-2026-02-25/) |
| 走私调查 | [The Information](https://x.com/theinformation/status/1999150974221337056); [Tom's Hardware 分析](https://www.tomshardware.com/tech-industry/artificial-intelligence/nvidia-decries-far-fetched-reports-of-smuggling-in-face-of-deepseek-training-reports-unnamed-sources-claim-chinese-company-is-involved-in-blackwell-smuggling-ring); [CFR 分析](https://www.cfr.org/articles/deepseek-v4-signals-a-new-phase-in-the-u-s-china-ai-rivalry) |
| Operation Gatekeeper | [US DOJ press release, Dec 2025](https://www.justice.gov/opa/pr/us-authorities-shut-down-major-china-linked-ai-tech-smuggling-network); [CNBC 深度报道](https://www.cnbc.com/2025/12/31/160-million-worth-export-controlled-nvidia-gpus-allegedly-smuggled-to-china.html) |
| 新加坡案件 | [Reuters, Feb 2025](https://www.reuters.com/technology/singapore-charges-three-with-fraud-that-media-link-nvidia-chips-2025-02-28/); [Reuters, $390M, Mar 2025](https://www.reuters.com/technology/servers-used-singapore-fraud-case-may-contain-nvidia-chips-minister-says-2025-03-03/) |
| 昇腾 950 发布 | 华为全联接大会 2025；[上海证券报, Apr 24, 2026](https://www.stcn.com/article/detail/3803257.html)；[每经网](https://www.nbd.com.cn/articles/2026-04-24/4358607.html) |
| DeepSeek API 定价 | [DeepSeek API Docs, Apr 24, 2026](https://api-docs.deepseek.com/news/news260424) |
| HGX B200 / GB200 NVL72 架构 | [NVIDIA 官方](https://www.nvidia.com/en-us/data-center/gb200-nvl72/); [Spheron](https://www.spheron.network/blog/nvidia-gb200-nvl72-guide/); [SemiAnalysis](https://semianalysis.com/2025/08/20/h100-vs-gb200-nvl72-training-benchmarks/) |
| GB200 NVL72 基准 | [LMSYS Part II](https://lmsys.org/blog/2025-09-25-gb200-part-2/); [LMSYS InferenceMAX](https://lmsys.org/blog/2025-10-14-sa-inference-max/); [NVIDIA Developer Blog](https://developer.nvidia.com/blog/nvidia-gb200-nvl72-delivers-trillion-parameter-llm-training-and-real-time-inference/) |
| DeepEP | [GitHub: deepseek-ai/DeepEP](https://github.com/deepseek-ai/DeepEP) |
| 走私指控（Reuters） | [Reuters（2026.02.24）](https://www.reuters.com/world/china/chinas-deepseek-trained-ai-model-nvidias-best-chip-despite-us-ban-official-says-2026-02-24/); [Reuters: DeepSeek 拒绝对 NVIDIA/AMD 提供早期访问（2026.02.25）](https://www.reuters.com/world/china/deepseek-withholds-latest-ai-model-us-chipmakers-including-nvidia-sources-say-2026-02-25/) |
| CM384 架构 | [CloudMatrix 384 论文 (arxiv 2506.12708)](https://arxiv.org/abs/2506.12708) |
| 盘古 Ultra MoE 训练 | [Pangu Ultra MoE 论文 (arxiv 2505.04519)](https://arxiv.org/abs/2505.04519); [华为 Cloud ModelArts](https://www.huaweicloud.com/intl/en-us/solution/modelarts.html) |
| xAI Colossus MFU 11% | [Business Insider（2026.04）](https://www.businessinsider.com/elon-musk-xai-compute-cursor-ai-model-training-2026-4); [Times of India](https://timesofindia.indiatimes.com/technology/tech-news/coding-startup-cursor-may-be-seeking-elon-musk-owned-xais-help-to-better-compete-against-anthropic/articleshow/130368875.cms) |
| CANN 适配 / 国产芯片分析 | [InfoQ](https://www.infoq.cn/article/wUUPEzvNajcaVN0k7HPF); [观察者网](https://www.guancha.cn/economy/2026_04_24_814819.shtml); [The China Academy](https://thechinaacademy.org/why-deepseek-v4-hasnt-fully-cut-ties-with-nvidia/) |

---

*本文写于 2026 年 5 月。所有推论基于截至写作日的公开信息。欢迎基于新证据的补充和纠正。*
