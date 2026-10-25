# 【arXiv‘2026】RSIGame: Autonomous Agentic Game Development with Recursive Self-improvement

## Metadata

- **Title:** RSIGame: Autonomous Agentic Game Development with Recursive Self-improvement
- **Authors:** Wenyi Wu, Minghao Fu, Jieyu You, Kun Zhou, Siqi Liu, Aayush Salvi, Yiheng Lin, Ce Zhang, Xiaohan Lan, Jiahui Zhu, Yujie Zhong, Qi She, Biwei Huang
- **Venue / Year:** arXiv / 2026（v1，2026-09-30）
- **Note Name:** 【arXiv‘2026】rsigame
- **Paper:** [arXiv:2609.39045](https://arxiv.org/abs/2609.39045)
- **Code:** [WenyiWU0111/RSIGame](https://github.com/WenyiWU0111/RSIGame)
- **Dataset / Artifact:** [RSIGame Hugging Face](https://huggingface.co/RSIGame)；[Table Artifacts](https://huggingface.co/datasets/RSIGame/RSIGame-TableArtifacts)；[Project Page](https://huggingface.co/spaces/RSIGame/rsigame-page)
- **Scope / Subfield:** 自主智能体游戏开发（autonomous agentic game development）与递归自我改进（recursive self-improvement）
- **Tags:** `Game AI`、`Agentic Coding`、`Recursive Self-improvement`、`Multi-agent System`、`Runtime Verification`、`Experience Internalization`、`Godot`、`Phaser`
- **Status:** DONE

## TL;DR

RSIGame 把一次性游戏生成改造成一个可验证的长期开发过程：局部循环负责探索游戏、诊断问题、修改项目并验证结果，全局循环负责保存最佳版本、监控停滞和避免后续编辑抹掉已有收益。系统还把成功的生成、规划和修复轨迹用于监督微调，使开发经验从当前任务的上下文进一步写入模型参数。作者在 GameCraft-Bench 的 140 个任务、Godot/Phaser 两个引擎和多个生成器上报告稳定提升，说明高质量的过程控制比“让 Agent 多改几轮”本身更重要。

## 毒舌评论

论文最吸睛的“Qwen3.8-27B 超过 GPT-5.5”并不是一个干净的同预算模型能力胜负：Qwen 先用包含 GPT-5.5 生成轨迹的数据做 SFT，再通过多轮 RSIGame 开发达到 61.38，而对照的 GPT-5.5 是 50.26 的一次生成。这个结果能证明“经验内化 + 长期迭代”有价值，却不能证明 Qwen 基础模型本身更强。更值得相信的是那些过程层面的证据——最佳 checkpoint 能阻止回退、运行时验证能发现编译检查看不见的失败、同预算下 RSIGame 通常优于 Play2Code；但 Phaser 上 Qwen 对 Play2Code 的 +1.55 分差异并不显著（p=0.394），所以“全面碾压”并不成立。

## Research Question

- **研究对象：** 从自然语言需求出发，自动生成并持续改进可执行游戏项目的智能体系统。
- **小领域范围：** Agentic game development、长程 agentic coding、runtime-grounded recursive self-improvement。
- **具体问题：** 如何让自动游戏开发在多轮修改中持续提高整体质量，而不是过拟合少数测试轨迹、修复一个问题又引入另一个问题，或者在后期回退？
- **为什么重要：** 完整游戏包含代码、资产、配置、交互逻辑和玩家体验，一次生成很难同时正确；然而朴素的“运行—修改”循环也可能快速停滞或越改越差。
- **论文边界：** 主要评估规模相对紧凑、可在最多 30 轮开发内处理的游戏；质量集中在 Mechanics、Depth、Visuals、Art 四个维度，不等价于原创性、叙事、长期留存或真人玩家乐趣。

## Motivation and Basic Idea

- **Motivation：** 现有模型已经能把需求变成可玩的游戏，但“可运行”离“质量可靠”很远。朴素迭代往往只围绕少量现有测试修改，容易遗漏未探索行为、边界条件和回归。
- **Basic idea：** 用局部的 `explore → diagnose → improve → verify` 循环积累运行证据和问题清单，再用全局 monitor 保存历史最佳版本、检测停滞，并在必要时开启新的开发阶段；最后把验证成功的开发轨迹训练回生成模型。
- **这个 idea 如何回应 motivation：** 广泛探索缓解对少数测试的过拟合；独立验证阻止未经证实的修改污染后续状态；最佳 checkpoint 让长期迭代具备单调保护；经验内化让成功过程跨任务复用。
- **作者给出的证据：** RSIGame 在 Godot 多个生成器上相对冻结初始项目提高 10.7–20.3 分；运行时 verification 将失败修改的检测 recall 从 build-only 的 0% 提高到 76.2%；训练完整的 generation+plan+improve 数据使 Qwen 一次生成从 37.07 提高到 48.22。
- **我的判断：** 方法最扎实的贡献是“过程控制”，而不是多 Agent 的角色数量。Controller、checklist、verifier、best-checkpoint 和 saturation stopping 共同把开放式修改变成可追踪、可回退的搜索过程。

## Background

- **背景：** 自动游戏生成正从静态代码生成转向完整、可执行、可互动的项目生成；GameCraft-Bench 等基准开始通过实际回放评估游戏机制、内容深度和视觉表现。
- **问题：** 既有基准通常只评分最终提交；既有迭代系统虽然能够试玩和修改，却没有充分解决长程改进中的方向选择、回归控制和停滞判断。
- **Gap：** 缺少一个同时覆盖自适应测试、证据驱动修改、独立验证、全局版本控制以及跨任务经验内化的自主开发框架。

## Threat Model / Assumptions (Optional)

本文不研究安全攻击，以下是影响结论成立的系统与评估假设：

- **参与者能力：** Controller、Explorer、Editor、Verifier 和 Monitor 均由具有视觉理解、代码修改和工具调用能力的模型承担，不同实验可使用不同底座模型。
- **环境假设：** 游戏能够自动构建、运行、重放交互轨迹，并从画面和运行状态中收集证据。
- **反馈隔离：** 开发 Agent 可以使用提交项目自带的 replay demonstrations，但不能访问隐藏 benchmark rubric、分数或 judge feedback；Monitor 使用独立 proxy rubric。
- **质量假设：** Mechanics、Depth、Visuals、Art 的加权分数能近似表示游戏质量；不可运行的项目由 `BUILD=0` 直接记零。
- **评估假设：** LLM judge 对同一组回放的相对排序足够稳定。论文用三次独立回放评分、跨 judge 重评和自由试玩补充验证这一点。
- **不覆盖的情况：** 大型长期项目、复杂跨系统依赖、真人长期游玩体验、商业生产中的团队协作、资产版权与持续运营。

## Method

### 核心思路

RSIGame 把游戏项目视为随开发轮次演化的状态 `P0 → P1 → … → PT`。局部循环每轮主动选择最有价值的开发方向，收集游戏运行证据，实施修改并重新验证；全局循环定期比较新版本与历史最佳版本，只保留真正变好的 checkpoint，并根据连续无改进次数判断是否停止或进入新阶段。

### 核心思路是怎么想到的

论文将人类游戏开发中的“试玩—发现问题—修改—回归测试—版本管理”抽象成 Agent 可执行的闭环。朴素 Agent 循环失败的关键不是不会写代码，而是缺少测试覆盖、共享状态、独立验收和全局版本控制；因此改进重点从更强的单次生成器转向更可靠的开发过程。

### 从 motivation 到 method 的逻辑链

1. 一次生成难以得到完整正确的游戏，因此必须迭代。
2. 只针对已知测试迭代会过拟合，因此需要主动探索和不断扩展的 checklist。
3. Agent 自己声称“修好了”不可靠，因此需要运行时 verifier 独立复查。
4. 单轮修改成功不代表整体游戏变好，因此需要全局 monitor 做成对比较并保存最佳版本。
5. 长期无提升继续烧算力没有意义，因此用 patience-based saturation detection 停止或切换阶段。
6. 成功开发过程具有跨任务价值，因此把生成、规划和已验证修复轨迹用于模型训练。

### 关键设计取舍

- **方向选择与执行分离：** Controller 决定测试/开发方向，Explorer 负责实际交互，避免探索退化成漫无目的试玩。
- **Editor 与 Verifier 分离：** 修改者不直接为自己的修改打分，减少自证式反馈。
- **Evolving checklist：** 保存问题、机会、优先级和验证状态，作为跨轮次共享工作记忆。
- **局部与全局评价分离：** local verifier 判断目标修改是否成功，global monitor 判断整个游戏是否变好。
- **保留最佳而非返回最后版本：** 后续编辑可能回归，最终产物应来自历史最佳 checkpoint。
- **可选高层指导：** 仅在自主开发饱和后由人或更强模型提供稀疏战略建议，而不是逐步编辑指令。
- **只训练验证成功经验：** Improvement 数据必须经过独立 post-edit interaction 验证；失败尝试作为 negatives 发布，但不作为正样本混入。

### 为什么不是更直接 / 更简单的方案

- 单纯增加开发轮数不能保证质量：Play2Code 和无 monitor 的轨迹可能停滞或回退。
- 只检查构建成功不能发现行为错误：build-only 对失败修改的 recall 为 0%。
- 固定 round-robin 开发方向不能响应当前瓶颈；盲测中 adaptive policy 以 59% 对 37% 被偏好。
- 只模仿最终游戏不足以学习如何开发；训练消融显示 generation traces 之外的 planning 和 verified improvement 数据提供进一步增益。

### 系统流程或算法步骤

1. Generator 根据自然语言规格 `x` 生成初始项目 `P0` 和可重放 demonstrations。
2. Controller 读取规格、当前项目、checklist 和可选阶段指导，选择下一开发方向 `dt`。
3. Explorer 针对 `dt` 试玩当前项目，得到交互轨迹 `τt` 和运行证据。
4. Editor 根据项目、方向和证据生成修改 `Δt`，得到候选项目 `Pt+1`。
5. Verifier 重放修改后的游戏，检查目标问题是否解决以及是否出现回归，产生结果 `vt`。
6. 更新 checklist：加入新问题、修改优先级并记录验证状态。
7. Global Quality Monitor 将候选版本与保留版本比较，更新 `P*s = SelectBest(P*s, Pt+1)`。
8. 若连续多次全局评估无提升，则停止并返回最佳版本，或加入稀疏高层指导后开启新阶段。
9. 收集成功的 generation、planning、verified improvement 轨迹，通过 LoRA SFT 内化到 Qwen3.8-27B。

### 关键定义 / 公式 / 不变量

- 初始项目：`P0 ~ πθ(· | x)`。
- 局部方向与探索：`dt ~ πcontroller(· | x, Pt, Ct, γs)`，`τt ~ πexplorer(· | Pt, dt)`。
- 修改：`Δt ~ πeditor(· | Pt, dt, τt)`，`Pt+1 = Apply(Pt, Δt)`。
- 验证：`vt ~ πverifier(· | Pt+1, dt, τt)`。
- Checklist 更新：`Ct+1 = U(Ct, dt, τt, Δt, vt)`。
- 全局不变量：任何阶段都保留截至当前观察到的最佳 checkpoint，不因最后一轮修改而无条件覆盖。
- Benchmark 总分：`Q = BUILD × (0.15M + 0.35D + 0.15V + 0.35A)`；不可运行时 `BUILD=0`。

### 实现细节

- 主实验：GameCraft-Bench 140 个任务、15 个游戏家族，分别在 Godot 与 Phaser 上实现。
- Play2Code 与 RSIGame 在比较中最多允许每轮 26 次工具调用、30 个开发轮次，并从相同 `P0` 开始。
- 每个固定游戏进行三次独立 replay-and-score，主 judge 为 Qwen3.8-27B。
- 训练原始数据包含 2,213 条生成轨迹、2,108 条规划和 4,003 次改进尝试，其中 2,013 轮通过独立验证。
- 最终训练变体使用嵌套数据集：1,294 条 generation、再加 308 条 planning、再加 509 条 improvement，共 2,111 条长上下文训练文档。
- Qwen3.8-27B 使用 4-bit NF4、rank-16 LoRA、57,344 token 上下文、1 epoch，在 4×H100 80GB 上训练。

## Evaluation

- **实验思路：** 在匹配初始项目和开发预算的条件下，对比 frozen base、Play2Code 与 RSIGame；再验证跨引擎、跨生成器、开发预算 scaling、global monitor、adaptive direction、agentic verification、跨 judge、自由试玩和经验内化消融。
- **评估指标：** Mechanics、Depth、Visuals、Art 及其加权 Overall；token、费用和开发轮数；verification 的 grounded precision、failure recall、specificity、balanced accuracy；盲测 win/loss/tie。
- **主要结果：**
  - Godot 上，RSIGame 相对 frozen base 在五组生成器设置中提高约 10.7–20.3 Overall；相对 Play2Code 提高约 7.2–13.8 分。
  - Codex+GPT-5.5：50.26（Base）→ 50.74（Play2Code）→ 64.53（RSIGame）。
  - Qwen3.8-27B：37.07 → 40.53 → 47.77；SFT 后 Base 达到 48.22，配合 RSIGame 达到 61.38。
  - Phaser 上，RSIGame 相对 frozen base 提高 8.8–13.4 分；但 Qwen3.8-27B 的 RSIGame 对 Play2Code 差异为 +1.55，95% CI [-1.50, 4.66]，p=0.394，未达到显著。
  - Agentic verification 将改进目标 grounded precision 从 58.6% 提高到 72.3%，每轮 unsupported targets 从 1.93 降至 0.50；对失败修改的 recall 为 76.2%，balanced accuracy 为 84.4%。
  - Cross-judge 40 任务重评中，Qwen 与 GPT-5.5 都保持 `RSIGame > Play2Code > Base` 的方法排序。
  - 自由试玩中，RSIGame 对 Base 为 17/3/0、对 Play2Code 为 18/2/0，说明提升不只局限于开发时复用的 demonstrations。
  - SFT 消融：BASE 37.07，+GEN 41.45，+GEN+PLAN 41.77，+GEN+PLAN+IMPROVE 48.22；大部分额外提升来自 verified improvement 数据加入后的阶段。

## Key Artifacts

- **Figure 2（系统总览）：** 清楚区分 local explore-diagnose-improve、global progress control 和 training-based knowledge internalization，是理解系统边界的核心图。
- **Table 1（主结果）：** 展示 140 个任务、Godot/Phaser、多生成器下的质量、token 和费用；支撑跨模型、跨引擎的整体提升结论。
- **Figures 3–4（开发时间 scaling 与经验迁移）：** 表明盲目增加轮数可能回退，而 best-checkpoint monitor 与 saturation stopping 提升质量—算力折中；同时展示经验内化带来的一次生成提升。
- **Figure 5（自适应方向）：** 说明系统会随游戏状态改变开发重心，并在盲测中优于固定 round-robin。
- **Table 2（Agentic verification）：** 量化运行时验证相对 free critic 和 build-only 检查的收益。
- **Tables 10–14（可靠性附录）：** 分析 replay 方差、跨 judge、自由试玩与 task-level bootstrap，给主结果补上统计与分布外交互证据。
- **Tables 15–18（训练数据与消融）：** 记录数据来源、污染检查、LoRA 配置和嵌套训练变体，支撑“改进经验可以内化”的结论。
- **核心算法不变量：** 始终返回历史最佳 checkpoint，而非最后 checkpoint；隐藏 benchmark rubric、分数和反馈不得进入开发循环。

## Findings

- **发现 1：** 迭代次数本身不是关键，能否持续探索新问题、验证修改并保留最好版本，才决定额外算力能否转化为质量。
- **发现 2：** 编译或构建成功远远不够；很多游戏修改在运行行为上失败，必须通过实际 replay 才能识别。
- **发现 3：** 自适应开发方向比固定轮换更有效，说明长程 Agent 应围绕当前系统瓶颈分配计算，而不是机械执行预设流程。
- **发现 4：** 规划和经过验证的修复轨迹具有跨任务学习价值；尤其 improvement 数据为一次生成带来明显额外增益。
- **发现 5：** 最强证据来自配对过程实验，而不是“开源模型超过闭源模型”的标题式对比。

## Strengths

- **论文最有说服力的地方：** 对失败模式定义具体，并用 local/global control、验证审计、自由试玩、跨 judge 与 bootstrap 对核心主张进行多角度验证。
- **方法优势：** 系统把开放式 Agent 行为转化为有状态、可验收、可回退的工程流程；每个角色都有清晰输入输出，而不是泛化的多 Agent 对话。
- **数据与复现优势：** 论文提供代码、完整 prompts、评分 artifacts、训练语料说明、运行树和详细超参数；还明确报告数据清洗、记录缺口与 benchmark task 去重检查。
- **相比已有工作的有效推进：** Play2Code 强调 gameplay-feedback editing，VibeGame 强调多角色协作；RSIGame 的新增价值在于自适应方向选择、全局最佳 checkpoint、饱和检测以及 verified experience internalization。

## Limitations

- **规模限制：** 实验集中在能用有限 Agent 预算迭代的小型游戏，尚未证明对大型项目、复杂资产管线和长期依赖有效。
- **评价维度限制：** Mechanics、Depth、Visuals、Art 不能完整覆盖原创性、叙事质量、长期玩家参与和主观乐趣。
- **自动评价依赖：** 主要评分依赖 replay demonstrations 和 LLM judge。作者用跨 judge 与自由试玩缓解担忧，但样本分别只有 40 和 20 个任务，仍不能替代大规模真人评测。
- **比较口径限制：** “SFT Qwen + RSIGame 超过 GPT-5.5 one-shot”同时改变了模型训练数据、推理过程和计算预算，不宜解读为单纯基础模型能力反超。
- **结果并非处处显著：** Phaser Qwen 设置下对 Play2Code 的差异未显著，说明优势大小与引擎、初始项目和可改进空间有关。
- **教师依赖与闭环成本：** 经验内化使用 GPT-5.5 生成数据和 GLM-5.3-Flash 改进数据，并需 4×H100 训练；小模型效率提升建立在前期昂贵的数据生产与筛选之上。
- **高层指导研究有限：** 人或强模型在自主改进饱和后介入的 multi-stage 机制只在少量案例中评估。
- **潜在同族偏差：** 主 judge 与被训练/评估的 Qwen3.8-27B 属于同一模型家族。跨 GPT-5.5 judge 结果支持排序稳健性，但仍应关注评价模型偏好。

## My Takeaways

- **对 Game AI / Agentic Coding 的启发：** 长程自主开发的核心不是一次调用多强，而是建立“主动测试—证据修改—独立验收—全局回退”的闭环。
- **可复用的方法：** evolving checklist、best-checkpoint tracking、patience-based stopping、验证成功轨迹筛选，可以直接迁移到软件工程、网页开发、机器人策略改进和数据分析 Agent。
- **评估启发：** 要区分“最后版本”“保留最佳版本”和“预算内 oracle best”，并对运行噪声、judge 变化和独立交互轨迹分别做稳健性分析。
- **训练启发：** 最终产物不是唯一有价值的监督；规划、工具调用、诊断和经过验证的修复决定了模型是否真正学会开发过程。
- **工程判断：** 如果预算有限，优先实现 verifier、checkpoint retention 和 saturation stopping，可能比继续增加 Agent 角色更有效。

## Related Papers

- **前置阅读：** GameCraft-Bench——提供本文 140 个任务及 interaction-grounded 游戏评价框架。
- **最接近的相关工作：** Play2Code——通过试玩反馈迭代修改游戏，是本文主要 iterative baseline。
- **可对比论文：** VibeGame——多角色 generate-and-verify 游戏开发系统；OpenGame——积累跨项目开发技能；Reflexion / Self-Refine——以反馈和记忆驱动语言模型自我改进。
- **后续阅读：** RSI Agent——在新环境中的自主探索与递归自我改进；更广泛的 process supervision、runtime verification 和 experience replay 工作。
- **关键差异：** RSIGame 不以“能否迭代”为主要贡献，而强调如何通过主动探索、全局控制和验证数据内化，让迭代在长程中可靠地产生净收益。

## Open Questions

- 在相同总 token、美元成本和 wall-clock 时间下，RSIGame 与更强的一次生成或较少轮次的强模型相比是否仍占优？
- 如果开发 demonstrations 完全隐藏，系统能否通过纯自由探索得到相同提升？
- Monitor 的 proxy rubric 是否会把游戏优化到某种固定审美或结构偏好？如何学习更接近真人玩家的质量模型？
- verified improvement 数据为何带来最大训练增益：是修复模式、运行反馈、工具使用，还是更高质量代码分布在起作用？
- 能否用因果归因或 counterfactual replay 判断某个修改真正改善了哪些质量维度，而不只依赖整体 pairwise judge？
- 在数百文件、长期版本演化和多开发者约束下，checklist 与 checkpoint search 如何扩展？
- 真人玩家对 RSIGame、Play2Code 和 one-shot 游戏的偏好是否与自动评估一致？
- 如何控制自动游戏开发中的版权、内容安全、奖励投机以及生成资产来源问题？
