# 【arXiv‘2026】LAMP: Layerwise Adaptive Mixing of Predictors for Training-Free Few-Shot Vision-Language Adaptation

## Metadata

- **Title:** LAMP: Layerwise Adaptive Mixing of Predictors for Training-Free Few-Shot Vision-Language Adaptation
- **Authors:** Zhipeng Ye, Feng Jiang, Qiufeng Wang, Hao Li
- **Venue / Year:** arXiv / 2026（v1，2026-10-15）
- **Note Name:** 【arXiv‘2026】lamp
- **Paper:** [arXiv:2610.22182](https://arxiv.org/abs/2610.22182)；[PDF](https://arxiv.org/pdf/2610.22182)
- **Code:** 截至本次阅读，论文与 arXiv 页面均未给出可核验的作者代码仓库
- **Dataset / Artifact:** ImageNet、Caltech101、Oxford-IIIT Pets、Stanford Cars、Flowers102、Food101、FGVC Aircraft、SUN397、DTD、EuroSAT、UCF101；分布迁移测试使用 ImageNet-V2 与 ImageNet-Sketch
- **Scope / Subfield:** 少样本视觉语言模型适配（few-shot vision-language adaptation）
- **Tags:** `CLIP`、`Few-shot Classification`、`Vision-Language Model`、`Training-free Adaptation`、`Kernel Residual Learning`、`Multi-layer Features`
- **Status:** DONE

## TL;DR

LAMP 解决冻结 CLIP 做 few-shot 适配时只使用最终全局特征、浪费中间层互补信息的问题。它让最终全局特征和三个中间层区域特征分别用核回归拟合同一个“标签减去 CLIP 先验分数”的残差，再把各分支输出放到统一的类别分数空间中加权融合，从而绕过不同层特征维度不一致的问题。十一数据集上，LAMP 相对配对的全局特征基线平均提升 ViT-B/16 1.14 个百分点、RN50 1.59 个百分点；LAMP-F 再通过文本锚定的类别原型优化，把平均增益扩大到 1.96 和 2.54 个百分点。论文最值得借鉴的是“异构表示不必先对齐，在共同任务输出空间中融合预测”这一设计模式。

## 毒舌评论

这篇论文的方法点子干净，实验也比常见的“换个模块涨零点几”扎实，但标题里的 **training-free** 容易让人忽略它对额外标注验证集的强依赖：EuroSAT 每类只有 1 张验证图时，20 组配置中有 19 组退化为选择全局基线；用完整验证集时每类却有 540 张验证图。换句话说，方法没有梯度训练，不等于没有监督调参成本。更尴尬的是 ImageNet 沿用既有实现，用同一个 50,000 张官方 validation split 既选参又报 benchmark 分数；这不必然使结果无效，但确实让“泛化提升”比表格中的粗体数字更需要打折阅读。

## Research Question

- **研究对象：** 冻结视觉语言模型（主要是 CLIP）的 C-way K-shot 图像分类适配。
- **小领域范围：** 无需更新视觉编码器参数的 few-shot VLM adaptation。
- **具体问题：** 如何利用编码器中间层保留的局部与多抽象层次信息，同时处理各层特征维度不同、不同任务偏好层次不同的问题？
- **为什么重要：** 现有 training-free 方法如 Tip-Adapter 和 ProKeR 主要基于最终全局描述符；中间层特征已经在一次前向传播中产生，如果能低成本利用，可能在不微调编码器的前提下改善目标任务表现。
- **论文边界：** 只研究闭集 few-shot 分类；需要标注 support set，并依赖标注 validation set 选择核与混合参数。LAMP 冻结编码器和文本原型；LAMP-F 会优化类别原型，因此严格说并非完全 training-free。

## Motivation and Basic Idea

- **Motivation：** 最终全局 CLIP 表示擅长图文对齐，却压缩了中间 token grid / convolutional feature map 中的区域细节和不同抽象层信息。直接拼接或投影多层特征又会遇到维度不一致、需要学习额外投影、以及任务间最优层不同的问题。
- **Basic idea：** 不在特征空间强行对齐各层，而让每一层独立解决同一个类别残差预测任务；因为所有分支最终都输出 C 维类别分数，所以可以在 class-score space 中直接融合。
- **这个 idea 如何回应 motivation：** 各层保留原生维度与相似度结构，避免学习共享投影；独立拟合后缓存分支预测，验证集只需搜索输出权重，不必为每个候选混合重新求解核系统。
- **作者给出的证据：** 在匹配搜索预算的 MULTIBAND 对照中，真正使用多层原生信息比对同一最终全局表示使用多个带宽高 ViT 1.286、RN50 1.635 个百分点；WHOLE 多层池化也比只用最深层 FINAL 高 1.193 和 0.905 个百分点。
- **我的判断：** 这是动机与方法衔接最自然的部分。主要增益来自“多深度信息”，并不强依赖复杂的 2×2 区域集合表示；论文自己的 REGION、CENTROID、WHOLE 结果几乎相同，也说明更简单的多层摘要可能更值得部署。

## Background

- **背景：** CLIP 以图像全局表示和文本类别原型的相似度完成零样本分类；few-shot adaptation 用少量标注样本修正目标任务预测。
- **问题：** prompt/prototype 方法通过训练调整类别表示，Tip-Adapter / ProKeR 等 training-free 方法则主要依赖最终全局图像表示；两类方法都没有系统利用多个中间层的原生表示。
- **Gap：** 缺少一种无需把异维多层特征投影到共同特征空间、又能按任务选择各层贡献的轻量方法。

## Threat Model / Assumptions (Optional)

本文不是安全攻击论文，这里记录影响结论成立的实验与建模假设：

- **任务假设：** 闭集 C-way K-shot 分类，support、validation、test 的类别集合一致。
- **数据假设：** 每类有 K 个标注 support 图像；每张 support 图像扩增为 10 个视图。模型选择还需要额外的标注 validation 数据，且这些标签不计入 K-shot budget。
- **表示假设：** 冻结 CLIP 编码器的中间层确实包含对目标任务互补的信息；不同视图的区域错位可通过无序的区域集合核处理。
- **选参假设：** validation accuracy 能可靠选择全局/区域混合强度、层权重和核带宽。论文的验证预算实验显示，小验证集下该假设可能失效。
- **评估边界：** 十个数据集有独立 validation/test 列表；ImageNet 按既有 Tip-Adapter / ProKeR 实现，使用同一个官方 50,000 图 validation split 做参数选择和 benchmark 评分。
- **统计边界：** bootstrap 区间以已经选定的模型为条件，不覆盖反复选模或方法开发过程的不确定性；主 benchmark 任务也参与了方法开发。

## Method

### 核心思路

用 CLIP 文本先验给出初始类别分数，再让全局分支和三个中间层区域分支分别拟合 support set 上的类别分数残差。各分支输出都为 C 维，因此可以在类别分数空间加权相加。

### 核心思路是怎么想到的

论文从两个观察出发：第一，中间层已由同一次 encoder forward 免费产生；第二，虽然这些层的特征维度和语义不同，但它们最终要回答的是同一个分类问题。由此将“融合异构特征”改写为“融合对同一残差目标的多个预测器”，避开表示对齐。

### 从 motivation 到 method 的逻辑链

1. 最终全局表示遗漏中间层互补信息。
2. 不同层不能安全地直接拼接或共享投影，于是每层保留自己的原生空间。
3. 所有层都用核相似度把 support label residual 传播到 query，形成统一 C 维修正量。
4. 用 validation labels 选择全局与区域修正的组合。
5. LAMP-F 再调整文本类别原型，并利用核预测对 residual 的线性关系缓存更新响应。

### 关键设计取舍

- **一次前向、多层取特征：** ViT-B/16 取 block 4/8/12；RN50 取 C3/C4/C5。
- **每层四个区域：** 使用固定 2×2 空间池化。ViT 对 token 先 LayerNorm 和 L2 normalize 后区域平均；RN50 用指数为 3 的 GeM pooling，再做 L2 normalize。
- **区域集合核：** 每对图像比较所有区域对并取平均，不要求区域位置一一对应；再按自相似度归一化。
- **support calibration：** 区域核在由 support bank 定义的核空间中居中和归一化；全局核不做该校准。
- **两阶段选参：** 先选全局带宽和正则化，再选区域带宽、全局/区域权重和层权重。
- **LAMP-F：** 保持核和混合权重固定，仅优化 C×d_g 个类别原型参数，并用文本锚定惩罚防止偏离原始 CLIP 文本向量。

### 为什么不是更直接 / 更简单的方案

- 直接拼接需要处理不同层的维度、尺度和语义差异，通常还要学习投影。
- 只对最终全局特征使用多个带宽不能复现多层特征的增益，论文用 MULTIBAND 对照验证了这一点。
- 只用最深中间层（FINAL）不如联合多个深度（WHOLE）。
- 不过，论文也发现 REGION 与 CENTROID / WHOLE 基本持平，因此 2×2 区域集合并非主要增益来源；部署时可以考虑更简单的 whole-map 或 centroid 表示。

### 系统流程或算法步骤

1. 对每个 support 图像生成 V=10 个增强视图；query 使用一个确定性视图。
2. 一次冻结编码器前向得到全局描述符，以及三个层级、每层 J=4 个区域描述符。
3. 用固定文本原型 T 计算 CLIP 先验和 support residual。
4. 构造全局核与三个经 support 校准的区域核。
5. 四个分支分别解线性系统，得到各自的类别残差预测。
6. 在标注 validation set 上选择核超参数、混合强度 ρ 和区域层权重 w。
7. 测试时把加权残差加到 CLIP 先验分数，取最大类别。
8. 可选的 LAMP-F 优化文本锚定原型，并通过缓存响应精确更新完整残差预测。

### 关键定义 / 公式 / 不变量

设 support view 的全局特征矩阵为 \(S_g\)，query 特征为 \(Q_g\)，one-hot 标签为 \(Y\)，类别文本原型为 \(T\)。所有分支拟合同一个 residual：

\[
F_0(Q)=Q_gT^\top,\qquad R_T=Y-S_gT^\top.
\]

全局 RBF-like angular kernel：

\[
k_\beta(a,b)=\exp[-\beta(1-a^\top b)].
\]

对分支 \(b\in\{g,1,2,3\}\)，独立求解：

\[
G_b=\mathcal K_b(S,S),\quad H_b=\mathcal K_b(Q,S),\quad
\alpha_b=(I_N+G_b/\lambda)^{-1}R_T.
\]

最终 LAMP 预测为：

\[
F_T(Q)=F_0(Q)+(1-\rho)H_g\alpha_g
+\rho\sum_{\ell=1}^{3}w_\ell H_\ell\alpha_\ell,
\]

其中 \(w_\ell\ge 0\)、\(\sum_\ell w_\ell=1\)、\(\rho\in[0,1]\)。\(\rho=0\) 退化为全局 ProKeR-style predictor。

LAMP-F 的原型变化记为：

\[
\Gamma(W)=\operatorname{norm}_2(W)-\operatorname{norm}_2(T),\qquad
P(W)=T+\Gamma(W).
\]

固定核和混合权重后，缓存响应 \(E(Q)\) 使新原型下的预测可以精确写成：

\[
F_{P(W)}(Q)=F_T(Q)+[Q_g-E(Q)]\Gamma(W)^\top.
\]

这个式子同时更新 query prior 与 support residual 对核修正的影响，无需为每个 prototype checkpoint 重新分解核矩阵。

### 实现细节

- 主实验使用 ViT-B/16、RN50；补充实验使用 RN101、ViT-B/32。
- 支持预算为每类 1、2、4、8、16 张图；每个设置使用 3 个 support sampling seeds。
- 每张 support 图像用 10 个增强视图，所以核矩阵大小为 \(N=VCK\)。
- LAMP-F 用 AdamW 优化类别原型，loss 为 support 分类交叉熵加按原始 CLIP 置信度加权的文本锚定项。
- 当前未发现作者公开代码，复现时需要从论文公式和实现细节自行搭建。

## Evaluation

- **实验思路：** 在 11 个分类数据集、5 个 support budgets、ViT/ResNet 多种 backbone 上比较 published baselines；再通过配对全局基线、MULTIBAND、多层/单层池化、校准消融、验证标签预算、额外 backbone、ImageNet 分布迁移和部署成本实验定位增益来源。
- **评估指标：** top-1 accuracy；数据集等权平均；部分配对分析使用 5,000 次 crossed bootstrap；验证预算实验使用 10 个新 support draws 和 paired Student-t intervals。
- **主要结果：**
  - ViT-B/16：LAMP 的 1/2/4/8/16-shot 十一数据集均值为 72.18/74.27/77.70/80.65/83.30%；LAMP-F 为 73.28/75.62/78.80/80.94/83.57%。
  - RN50：LAMP 为 64.79/67.50/71.19/75.03/78.73%；LAMP-F 为 65.91/68.83/72.65/75.50/79.10%。在论文列举的 training-free RN50 方法中，LAMP 在 4/8/16-shot 达到最高均值。
  - 相比配对全局基线，跨 11 数据集和 5 个 budgets 平均，LAMP 提升 ViT 1.14 pp、RN50 1.59 pp；LAMP-F 提升 1.96 pp 和 2.54 pp。
  - 4-shot 最大多层增益来自 EuroSAT：ViT +5.860 pp、RN50 +6.695 pp；LAMP-F 相对 LAMP 的最大增益来自 SUN397：+5.620 pp 和 +6.715 pp。
  - LAMP-F 在 55 个数据集×预算组合中，ViT 改善 53 个、RN50 改善 52 个；但随着 shot 增加，原型优化的边际增益下降。
  - ImageNet→ImageNet-V2 / Sketch 冻结迁移中，四个 backbone-target 聚合设置均有正增益，约 +0.453 到 +0.860 pp。
  - 代价显著：ImageNet 16-shot 单图延迟从 ViT 14.22 ms 增至 64.83 ms、RN50 14.81 ms 增至 65.86 ms；batch-128 GPU 峰值从 2.71→7.64 GiB 和 3.02→11.55 GiB。LAMP-F 更高，分别达到 8.56 与 13.39 GiB。

## Key Artifacts

- **Figure 1（方法总览）：** 展示多层编码、四个独立核残差求解器和类别分数空间的选择性融合，是理解 LAMP 最关键的图。它支撑“不同维度的表示无需对齐，只需输出到共同类别空间”的核心设计。
- **Equations (1)、(3)、(9)、(10)：** 分别定义共同 residual、全局核残差预测、各分支独立求解和最终混合，构成 LAMP 的数学主干。
- **Equations (13)、(14)：** 给出 LAMP-F 的缓存线性响应，证明 prototype 更新可同时精确作用于 prior 和 residual correction，而无需重复矩阵分解。
- **Tables 1–2：** 汇总 ViT-B/16 与 RN50 的 published-method 比较；支撑 RN50 的 training-free SOTA 声明，但不同方法协议并不完全一致。
- **Table 4：** 与共享 support banks、prompts 和全局超参数的配对全局基线比较，是比跨论文榜单更可信的核心效果证据。
- **Tables 5–6：** 表明多层增益集中在 EuroSAT/DTD 等任务，并用 MULTIBAND、WHOLE、FINAL 等控制说明“不同深度”是主要增益来源，而细粒度区域集合本身贡献有限。
- **Figure 2（验证标签预算）：** 显示验证标签数量会强烈左右模型选择；这是方法监督成本和适用性的关键反证性证据。
- **Tables 7–9：** 支撑跨 backbone 和冻结目标分布迁移的稳健性结论。
- **Figures 3–4：** 显示最佳全局/区域比例和层权重随任务、backbone 变化，支持自适应混合而非固定层组合。
- **Table 10：** 量化部署开销，揭示 accuracy gain 与延迟、显存、核计算之间的现实权衡。

## Findings

- **发现 1：** 多层原生表示确实提供最终全局特征之外的信息；在匹配搜索预算的控制下，增益不能仅由多个核带宽解释。
- **发现 2：** 主要价值来自联合多个深度，而不是保留精细的区域集合。REGION、CENTROID 与 WHOLE 表现接近，说明可以用更紧凑表示换取接近的精度。
- **发现 3：** 最佳层组合高度任务相关。纹理/遥感类任务更偏好区域或中间层信息，Pets 等任务更依赖全局分支。
- **发现 4：** LAMP-F 的 prototype adaptation 与 LAMP 的多层 residual integration 作用互补；前者在 1–4 shot 更有效，后者在更高 shot 下仍能持续受益。
- **发现 5：** labeled validation set 是隐藏的关键资源。验证预算太小时，搜索经常无法发现有效的区域混合。

## Strengths

- **论文最有说服力的地方：** 核心方法非常清楚，且通过共享 support、prompt、query 的配对基线和多个 representation controls 拆解增益来源，没有只依赖跨论文排行榜。
- **方法优势：** 在 score space 融合异维特征的设计通用且可解释；分支独立求解后可复用预测；LAMP-F 的缓存响应有明确的线性代数依据，而不是启发式加速。
- **实验优势：** 覆盖 11 数据集、5 个 shots、4 个 backbone、额外分布迁移、验证预算和部署成本；报告了均值、标准差与若干配对置信区间。
- **相比已有工作的有效推进：** 相比 ProKeR 仅用最终全局特征，LAMP 系统利用多个中间层；相比 SSP 的局部子空间投影，LAMP 在共同 residual target 上融合多个层级的类别修正，并保留各层原生维度。

## Limitations

- **监督预算表述：** support K-shot 数量没有包含 validation labels。对 DTD / EuroSAT，完整验证集分别有每类 24 / 540 张图，可能远大于 support budget；因此“few-shot”主要描述拟合 support predictor 的样本数，而不是整个模型选择流程的标注成本。
- **ImageNet 协议：** 同一官方 50,000-image validation split 同时用于参数选择和 benchmark scoring，存在选择偏乐观风险。作者说明这是沿用 Tip-Adapter / ProKeR 实现，但协议惯例不能消除评估边界。
- **跨论文比较公平性：** published baselines 的选参协议不同；例如 CLIP-LoRA 跨任务共享超参数，LAMP 使用 dataset-specific validation。表 1–3 的粗体不能被解读为严格同预算、同协议的全面胜出。
- **统计不确定性：** 置信区间以已选模型为条件，不覆盖 repeated selection 和方法开发不确定性；主 benchmark 还参与了方法开发。
- **增益不均匀：** 某些数据集增益很小或为负。例如 4-shot RN50 在 SUN397 上 LAMP 相对全局基线为 −0.032 pp；RN101 的 55 个设置中也有 5 个负变化。
- **计算与存储：** 每个分支需要 \(O(N^2)\) 存储和 \(O(N^3)\) 稠密分解；多层区域匹配将 ImageNet 单图延迟提高约 4.5 倍，并显著增加显存。
- **复现性：** 当前未找到作者代码；论文虽给出较多公式和实验细节，但 337-candidate mixture search、数据增强、validation splits 和完整超参数仍会增加复现成本。
- **方法叙事：** LAMP 可称 training-free（不做梯度更新），但 LAMP-F 明确用 AdamW 训练类别原型；两者应在结论中严格区分。

## My Takeaways

- **对视觉语言研究的启发：** 当不同层或不同模态表示难以对齐时，可以先让它们各自完成同一个下游预测，再在输出空间融合；“prediction alignment”常比“feature alignment”更便宜、更稳健。
- **可复用的方法：** 共同 residual target、独立核求解、validation-selected mixture，以及利用线性系统响应缓存参数更新影响，都可以迁移到多 backbone、多模态专家或多尺度特征融合。
- **实验设计启发：** 必须把“支持集标注预算”和“选参验证集标注预算”分开报告；否则 training-free / few-shot 很容易只省了梯度，却把成本转移到 validation labels 与网格搜索。
- **工程判断：** 若部署受限，应优先试验 WHOLE / CENTROID 多层摘要，而不是完整 REGION；论文结果表明它们保留了大部分增益，却只需更少存储。
- **可能的后续方向：** 用无需额外标注的混合权重估计、低秩/随机特征核近似、稀疏 support bank 或在线层选择，降低 validation、\(N^3\) 求解与显存成本。

## Related Papers

- **前置阅读：** CLIP（Radford et al., ICML 2021）——零样本图文对齐基础；Tip-Adapter（Zhang et al., ECCV 2022）——training-free cache-based adaptation。
- **最接近的相关工作：** ProKeR（Bendou et al., CVPR 2025）——LAMP 的全局 residual kernel 基础；SSP（Zhu et al., ACM MM 2024）——使用局部特征的 training-free CLIP adaptation。
- **可对比论文：** CLAP（Silva-Rodríguez et al., CVPR 2024）——LAMP-F 的文本锚定原型适配来源；CLIP-LoRA（Zanella & Ben Ayed, CVPRW 2024）——通过低秩更新视觉编码器的训练式适配。
- **关键差异：** ProKeR 只用最终全局描述符；SSP 通过局部视觉/语言子空间改善跨模态对齐；LAMP 则让多层区域核分别拟合共同 residual，再在类别分数空间混合。
- **后续阅读：** 可关注低秩核近似、无验证集的 mixture-of-experts gating，以及面向 distribution shift 的 layer selection 工作。

## Open Questions

- 如果把 validation labels 也计入总标注预算，LAMP 相对只增加 support shots 的简单基线是否仍然占优？
- 是否能用 support-only 的交叉验证、leave-one-out 风险或无监督置信度选择 \(\rho,w,r\)，避免额外 validation set？
- REGION 与 WHOLE 几乎持平，这一结论能否在细粒度定位、长尾类别或非中心目标数据集上保持？
- 稠密核求解能否用 Nyström、random features、conjugate gradient 或类中心压缩降低到可扩展规模？
- ImageNet 上若使用严格独立的 validation 与 test，当前增益及超参数偏好会发生多大变化？
- LAMP-F 的提升有多少来自 prototype adaptation 本身，有多少来自继承了 LAMP 的 337-candidate mixture search？更严格的总搜索预算匹配是否改变结论？
- 作者代码若发布，能否复现论文声明的所有 split、augmentation、search grid、bootstrap 和部署成本？
