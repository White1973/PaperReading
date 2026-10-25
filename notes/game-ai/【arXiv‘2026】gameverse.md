# 【arXiv‘2026】GameVerse: Can Vision-Language Models Learn from Video-based Reflection?

## Metadata

- **Title:** GameVerse: Can Vision-Language Models Learn from Video-based Reflection?
- **Authors:** Kuan Zhang, Dongchen Liu, Qiyue Zhao, Jinkun Hou, Xinran Zhang, Qinlei Xie, Miao Liu, Yiming Li
- **Venue / Year:** arXiv / 2026（v2，2026-03-10）
- **Note Name:** 【arXiv‘2026】gameverse
- **Paper:** [arXiv:2603.06656](https://arxiv.org/abs/2603.06656)；本地 PDF：`/Users/white1973/Zotero/storage/ZKIVHWNA/Zhang 等 - 2026 - GameVerse Can Vision-Language Models Learn from Video-based Reflection.pdf`
- **Code:** 未在论文中给出可核验的代码仓库；[Project Page](https://gameverse-bench.github.io/)
- **Dataset / Artifact:** GameVerse benchmark：15 款游戏、两种动作空间、视频反思协议与里程碑评分流程
- **Scope / Subfield:** 视觉语言模型游戏智能体；视频反思；端到端游戏 agent 评测
- **Tags:** `VLM Agent`、`Game Benchmark`、`Video Reflection`、`GUI Control`、`Generalization`、`Milestone Evaluation`
- **Status:** DONE

## TL;DR

GameVerse 是一个测试 VLM 是否能从“自己的失败录像 + 专家教程录像”中改进下一次游戏行为的 benchmark。它覆盖 15 款游戏，以画面结构、实时性和因果线性度建立认知层级分类；提供语义动作与原生 GUI 动作两种设置，并通过原生分数或视觉里程碑评估进度。

论文发现视频反思通常能带来提升，且“失败经验 + 教程示范”优于只用其中一项，作者将之类比为不更新参数的 RL 式错误剪枝与 SFT 式行为模仿的组合。不过，模型在复杂、实时、开放世界任务中仍严重落后于人类；不少模型能说出正确反思，却无法把它稳定转成精确、及时的下一步操作。

## 毒舌评论

“模型能从视频反思中学习”这个标题比证据走得更快：所谓 learning 主要是把一次性视频对比得出的文字经验塞回下一轮 prompt，而不是参数更新、稳定的技能归纳或长期持续学习。更尴尬的是，复杂游戏进度本身由单一强 VLM 从视频里判定；即使作者手工核验里程碑，评估器偏差和被评模型共享视觉盲点的风险仍未完全排除。它确实证明了上下文里的失败对比有用，但离人类式反思学习还有很长一段“知道 - 做到”的距离。

## Research Question

- **研究对象：** 在视觉输入和游戏交互中运行的通用 VLM agent。
- **小领域范围：** 视觉原生游戏评测、失败反思、教程学习、GUI 控制与跨游戏泛化。
- **具体问题：** VLM 能否像人类玩家一样，先尝试任务、回看失败、参考专家教程，并据此在下一轮改进策略？
- **为什么重要：** 传统 fire-and-forget benchmark 只测第一次尝试，无法测试 agent 是否会从实际视觉经历中更新行为；而真实交互智能体应具备失败诊断、经验积累与恢复能力。
- **论文边界：** 评测的是单次 reflect-and-retry 的 in-context 策略修正，不是在线参数学习、长期记忆训练或开放式持续学习。

## Motivation and Basic Idea

- **Motivation：** 现有游戏 benchmark 常有三个缺口：（1）用 RPG/FPS 等商业类型分类，无法定位模型究竟在哪种认知压力下失败；（2）借助 API 或文本化真值状态，减弱人类式视觉游戏负担；（3）采用 fire-and-forget，只测即时表现，不测失败后的反思与重试。
- **Basic idea：** 构造视觉原生的游戏循环：agent 先玩并失败，系统保存失败视频并检索专家教程；VLM 比较两条轨迹，压缩成可执行的经验，再将该经验加入下一次系统提示。
- **这个 idea 如何回应 motivation：** 失败录像提供“我在哪个具体状态走错”的负样本，专家视频提供“应该怎么走”的正样本；二者的对比使模型有机会从抽象策略和具体动作两层改进。
- **作者给出的证据：** 在 15 款游戏、7 个 VLM 的结果中，视频反思通常有正增益；对 3 款游戏的消融显示，联合失败与教程比单用失败、单用教程更好。
- **我的判断：** 该设计将反思从泛泛的自我批评变成可观察的“失败轨迹 - 专家轨迹”差异诊断，问题设定很扎实；但 prompt 注入的改进是否可泛化到新关卡、能否长期保留，仍未被证明。

## Background

- **背景：** VLM 在静态理解与部分 GUI 基准中表现突出，但动态游戏要求视觉定位、时间同步、规划、记忆和动作执行形成闭环。
- **问题：** 视觉游戏中，模型可能正确理解目标却无法点击正确位置、无法在实时环境及时执行，或不能把失败归因为可修复策略。
- **Gap：** 先前视频学习工作常离线训练，文字反思工作常只检查内部 CoT；缺少直接将“自我失败视频”和“外部专家教程视频”并置、再评估重试改进的视觉 benchmark。

## Threat Model / Assumptions (Optional)

本文不是安全论文。影响结论的关键实验假设为：

- **视觉输入假设：** 游戏 agent 主要依赖截图/视频，而非游戏内部状态；这更接近人类玩家，但视觉噪声会与规划错误混杂。
- **教程假设：** 能从公共平台检索到与目标任务足够匹配的高质量专家 walkthrough；教程的版权、可得性、版本匹配和任务覆盖度会影响可扩展性。
- **反思假设：** 模型能从一次失败轨迹和一次专家轨迹中抽取能迁移到下一次尝试的经验；这对强模型和低复杂度任务更成立。
- **评测假设：** 复杂游戏的完成进度可由 Gemini-3-pro 从视频抽取、匹配里程碑，并经人工验证。该过程不是环境真值，也可能带来评估器偏差。
- **不覆盖：** 多轮主动检索、对教程提问、长期记忆更新、参数化在线学习，以及在无可用教程的首次接触任务中的表现。

## Method

### 核心思路

GameVerse 将评测从一次性执行改为 **reflect-and-retry**：

```text
当前截图 → VLM 执行动作 → 失败轨迹视频
                               │
专家教程视频 ───────────────────┤
                               ↓
                     对比、诊断、生成经验摘要
                               ↓
                  将经验注入下一轮 prompt，重新尝试
```

反思可同时提取两类信息：抽象原则（如 2048 中把最大数固定在角落并维持“蛇形”排列）与具体行为模仿（如 Angry Birds 中攻击中央支柱而非上方木板）。

### Cognitive Hierarchical Game Taxonomy

论文不用商业类型，而用三条更直接影响 agent 难度的认知轴：

1. **Image Structure：** Grid / 2D / 3D，控制视觉空间、深度和界面复杂度；
2. **Temporal Dynamics：** Non-real-time / Real-time，控制模型推理延迟是否会使观察过时；
3. **Causal Linearity：** Linear / Non-linear，控制目标路径是否相对固定，或需在开放分支中探索和恢复。

这三条轴产生五类环境，每类又按环境复杂度与动作空间分为 easy / medium / hard：

| 类别 | 含义 | 代表游戏 | 主要检验能力 |
|---|---|---|---|
| **Markov Grid** | 回合制、离散状态、通常完全可观察；当前网格基本包含下一步决策所需信息。 | Tic-Tac-Toe、Baba Is You、2048 | 规则理解、合法动作、离散空间与概率规划。 |
| **Non-Real-Time Linear** | 无连续时间压力，任务总体沿明确目标/关卡推进。 | Maze、Angry Birds、Slay the Spire | 路径规划、物理因果、资源管理、多步执行。 |
| **Non-Real-Time Non-Linear** | 无实时压力但有开放分支、探索和长期取舍。 | Ace Attorney、Civilization VI、Scene Investigators | 记忆、信息整合、证据推理、长期规划。 |
| **Real-Time Linear** | 环境持续变化且任务方向较明确。 | Snake、Plants vs. Zombies、Forza Horizon 5 | 低延迟反应、视觉-动作对齐、动态空间预测。 |
| **Real-Time Non-Linear** | 实时、开放、分支目标叠加，是最接近完整人类游戏循环的压力测试。 | Mini Metro、Genshin Impact、Red Dead Redemption 2 | 长程目标、3D感知、实时控制、重规划与错误恢复。 |

一个易混淆点：Markov Grid 与 Non-Real-Time Linear 都可以不受实时性约束，但前者重点是“当前离散盘面下，哪一步合法且优”；后者重点是“如何用一串连续行动推进既定目标”。例如井字棋中历史已压缩在当前棋盘；迷宫则要求视觉定位、保持路径、规划整段路线并在走错后修正。

### Dual Action Space

- **Semantic action ($A_S$)：** 高层命令，例如 `Position(1,3)`；主要测视觉理解与策略推理，屏蔽精确鼠标/按键难题。
- **GUI action ($A_G$)：** 低层键鼠操作，例如 `KeyPress(A)`；测端到端视觉定位、动作参数和时序控制。

二者是刻意设置的对照：若 semantic 强、GUI 弱，说明瓶颈在“知道 - 做到”的映射，而非高层策略本身。

### 评测与评分

对于 8 款有细粒度内置数值的游戏，直接使用原生分数。其余 7 款复杂游戏使用里程碑评分：

1. 评估 VLM 观看专家 walkthrough，提取参考里程碑集合 $M_{ref}=\{M_1,\ldots,M_N\}$；每项含时间戳、标题和场景描述。
2. 评估 VLM 再观看 agent playthrough，匹配完成的里程碑 $M_{finished}$。
3. 过程分数为：

\[
S=\frac{|M_{finished}|}{|M_{ref}|}\in[0,1].
\]

论文称所有里程碑均由人工验证；这使闭源商业游戏无需内部 API 也能比较进度，但没有消除 VLM 评估器的系统误判风险。

### 为什么失败反思与教程要结合

- 失败（Self-F）指出哪条错误分支应被排除，作者类比为 RL 的负向约束；
- 教程（Self-T）给出成功行为和优质轨迹，作者类比为 SFT 的正向示范；
- 两者结合（Self）同时回答“哪里错”与“怎么改”，应比单一信息源更稳定。

这一比喻描述的是上下文内的功能对应，**不是**证明其在优化意义上等价于 RL + SFT。

## Evaluation

- **实验思路：** 在 15 款游戏中测试 7 个 VLM（Qwen3-VL-8B/32B、GPT-4o/4o-mini、Seed-1.8、Gemini-2.5-Pro/Flash），比较无视频反思与使用失败+教程视频反思的重试表现；辅以随机、人类新手、人类专家三类基线。
- **样本与设置：** 共 37 名人类参与者、458 次游戏 session。每游戏 3–20 次 trial；10 个游戏使用仅含当前截图和上一步视觉-动作对的 zero-shot agent，5 个游戏使用有长短期历史的 memory-aware agent。分数被归一化到 100 分并人工核验。
- **主要结果：**
  - Gemini-2.5-Pro 在 GUI 设置的平均排名最佳（无反思/有反思为 9.2/7.5，越小越好），但在困难任务并不稳定。
  - 简单井字棋中 Gemini-2.5-Pro 反思后达到 100，接近人类专家 98.9；困难的 Scene Investigators 与 Red Dead Redemption 2 上，模型表现可降至 0，显著低于人类新手约 58.4–70.4。
  - 反思平均增益与模型能力正相关、与任务复杂度负相关：Gemini-2.5-Pro 增加约 6.47%，GPT-4o-mini 约 1.60%；非实时任务平均增加约 4.4%，实时任务约 1.7%。
  - 在四个对照游戏上，语义动作平均 50.5，GUI 动作仅 33.5；反思增益亦为语义模式约 8.7%、GUI 约 3.75%。
  - Table 5 的三游戏消融中，联合失败和教程优于任一单独来源：GPT-4o-mini 的平均分从无反思 45.6 升至 65.5；Qwen3-VL-32B 从 61.0 升至 76.7。
  - 强 Qwen3-VL-32B 生成的反思无法进一步提升弱 GPT-4o-mini（59.1 < 其自身反思的 65.5），表明瓶颈更可能在反思的**内化与执行**，而不只是信息提取。

## Key Artifacts

- **Table 1：** 将 GameVerse 与已有 benchmark 对比，突出其认知 taxonomy、视觉原生、reflect-and-retry 与双动作空间。
- **Figure 2 + Table 2：** 15 个游戏的五类认知矩阵、动作设置、评分流程和能力覆盖，是 benchmark 构造的主要证据。
- **Table 3 + Figure 3：** 七个模型跨五类游戏的结果；支撑“简单任务可行，复杂/实时/非线性任务崩塌”与“强者越能从反思受益”。
- **Table 4：** semantic 与 GUI 的并行比较；直接量化 knowing-doing gap。
- **Figure 5：** 推理型模型受实时延迟影响更大；停止时间时 PvZ 可约 85 分，正常实时下降到约 35 分。
- **Table 5：** 分离失败和教程的反思来源，支撑二者互补而非冗余。
- **Figure 6：** 2048 与 Angry Birds 的定性案例，展示抽象策略和物理因果策略如何改变行为。
- **Figure 7：** 错误分为感知、推理、执行、延迟四类；随着环境从网格走向高保真动态世界，Qwen3-VL-32B 的感知错误比例从 16.7% 升至 39.8%，且推理/执行错误在各类中都超过 15%。

## Findings

- **发现 1：** VLM 在简单、低复杂度视觉游戏中已有非随机的基本能力，但难以随环境复杂度和开放性保持表现。
- **发现 2：** 反思不是普遍魔法：只有能力足以理解并执行反思的模型才能稳定获益，复杂实时任务会压缩其收益。
- **发现 3：** 失败经验和专家示范分别提供负向与正向学习信号；结合更有效，但仍受模型内化与执行能力制约。
- **发现 4：** 语义动作显著优于 GUI 控制，说明当前 VLM 的大问题不只是“不会规划”，更是不能将正确规划变成精准、及时的身体动作。
- **发现 5：** 感知、推理、执行、延迟会在真实游戏中叠加；终局得分不足以说明模型到底哪里失败，过程和错误分析是必要的。

## Strengths

- **问题设定：** 把教程学习和失败反思纳入 benchmark，而不只评一次性成功率，补上了动态 agent 评测的重要维度。
- **评测设计：** 用认知轴而非商业类型组织游戏，能更准确地联系任务结构与模型失败类型。
- **接口对照：** 同时设置 semantic/GUI action，使“高层推理不足”与“低层执行失败”可以部分分离。
- **实验解释力：** 失败/教程消融、延迟实验和错误分类，共同支持论文关于 cognitive bottleneck 与 physical disconnect 的叙述。

## Limitations

- **反思并非训练：** 每次只把摘要放进下一轮上下文，未测量跨任务、跨关卡或长时间尺度的经验保持，也未显示模型在无教程条件下会主动求证。
- **教程依赖：** 公共专家视频可能与游戏版本、关卡、目标或输入设置不匹配；公开教程广泛存在的热门游戏，也未必代表真正的 first-contact 泛化。
- **评测器依赖：** 7 个复杂游戏的里程碑分数来自 Gemini-3-pro 视觉匹配；人工验证减少误差但仍不能等价于环境真值，且单评估模型可能系统性偏向某些轨迹风格。
- **实验异质性：** 10 个游戏使用 zero-shot agent，5 个使用 memory-aware agent；结果横向比较时，模型能力和系统 memory 配置是混杂因素。
- **样本量与方差：** 每游戏仅 3–20 trial，部分任务分数方差大；对高成本长时开放世界任务，平均提升的稳健性仍需更多重复运行。

## My Takeaways

- benchmark 不应只问“第一次能否完成”，还应问 agent 是否能定位失败、吸收外部示范、在第二次做出可测改进。
- 评测中应并列 semantic 和原生 GUI 设置；否则在高层接口取得的策略分数容易被误解为端到端视觉游戏能力。
- “视频反思”最合适的近期用途是生成任务特定的执行约束和经验记忆；若要称为学习，需要评估它跨 episode 的持久性、主动检索能力和对未见情境的迁移。
- 与上一篇 *Towards Generalist Game Players* 的关联：GameVerse 将其中 Benchmark 与 Harness 两支柱具体化；其反思提示相当于外部适应模块，也恰好暴露了其所述的 reasoning-vs-reactivity 和 modular-workflow-vs-model-as-whole 两项 trade-off。

## Related Papers

- **前置阅读：** Reflexion、Self-Refine、Voyager、GROOT、Reflection of Episodes。
- **可比较 benchmark：** VideoGameBench、V-MAGE、BALROG、FlashAdventure、LMGAME-BENCH、Orak、AI GameStore。
- **可比较 agent：** Cradle、CombatVLA、JARVIS-VLA、UI-TARS、Game-TARS、NitroGen、Lumine。
- **最接近的相关工作：** VideoGameBench 和 FlashAdventure 均保留较强视觉/GUI 游戏交互，但主要采用 fire-and-forget；GameVerse 的差异是加入 failure-video/tutorial-video 驱动的 reflect-and-retry 协议与双动作空间对照。

## Open Questions

- 反思摘要能否跨关卡、跨游戏和跨多天 session 被稳定地检索、验证和复用？
- 若没有专家教程，agent 能否主动检索、提出反事实问题或通过试验建立可靠的自我教学信号？
- 如何在不使用单一强 VLM 裁判的条件下，对闭源复杂游戏建立可信、可扩展且抗偏差的过程评分？
- 如何把视觉反思“编译”为低延迟、连续、精细的动作策略，而非仅停留在正确的文字计划？
- 如何分离评测得到的主模型能力、记忆模块收益、反思模板收益、教程信息泄漏和接口特权？
