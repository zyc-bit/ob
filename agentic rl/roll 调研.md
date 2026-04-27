# Alibaba ROLL 框架调研报告

> 调研日期：2026-04-22  
> 调研对象：Alibaba ROLL / `alibaba/ROLL`  
> 关联项目：ROCK、iFlow CLI、ROME / ALE  
> 调研范围：官方文档、GitHub 仓库、发布记录、技术论文、ROCK & ROLL 集成文档、同类项目公开资料。  
> 说明：本调研基于公开资料分析，未在本地实际部署运行。

---

## 1. 结论摘要

ROLL，全称 **Reinforcement Learning Optimization for Large-Scale Learning**，是阿里开源的面向大语言模型的强化学习训练框架。它的核心定位是 **大模型后训练 / RLHF / RLVR / Agentic RL 的分布式训练框架**，而不是环境管理框架。官方文档称 ROLL 是面向大规模语言模型的开源强化学习框架，基于 Ray 分布式架构，支持 PPO、GRPO 等主流算法，并提供从研究到生产的完整方案。([alibaba.github.io](https://alibaba.github.io/ROLL/ "Hello from ROLL | ROLL"))

ROLL 与 ROCK 的关系可以简单理解为：

> **ROLL 是训练大脑，ROCK 是训练场。**  
> ROLL 负责采样、奖励、优势估计、策略更新、模型同步和分布式 GPU 调度；ROCK 负责提供可执行、可隔离、可扩展的环境沙箱，让 Agent 在环境中行动、观察、验证结果。

在阿里的 ALE / ROME 技术报告中，这种分工被明确写成三组件体系：ROLL 是 post-training / 权重优化框架，ROCK 是用于轨迹生成的沙箱环境管理器，iFlow CLI 是用于环境交互的 Agent 框架。([arXiv](https://arxiv.org/html/2512.24873v3 "Let It Flow: Agentic Crafting on Rock and Roll Building the ROME Model within an Open Agentic Learning Ecosystem"))

从成熟度看，ROLL 发展很快，GitHub 当前显示约 **3.1k stars、273 forks、456 commits**，最新 GitHub Release 为 **v0.2.1，发布于 2026-03-09**。([GitHub](https://github.com/alibaba/ROLL "GitHub - alibaba/ROLL: An Efficient and User-Friendly Scaling Library for Reinforcement Learning with Large Language Models · GitHub")) 但它仍属于快速演进期，接口、调度器、后端、pipeline 和示例都在持续变化；生产落地前应做版本锁定、依赖镜像化、集群压测、安全边界验证和完整回归。

综合判断：**ROLL 是一个高潜力的大模型 RL 后训练框架，尤其适合 RLVR、数学/代码推理强化学习、多任务奖励训练、Agentic RL、ROLL + ROCK 多轮环境交互训练等场景；但它的工程门槛明显高于 TRL 这类轻量库，更适合有 GPU 集群、Ray、vLLM/SGLang/Megatron/DeepSpeed 运维经验的团队。**

---

## 2. 项目基本信息

|维度|信息|
|---|---|
|项目名称|Alibaba ROLL|
|全称|Reinforcement Learning Optimization for Large-Scale Learning|
|GitHub 仓库|`alibaba/ROLL`|
|当前 GitHub Release|v0.2.1|
|主要语言|Python|
|开源许可证|Apache-2.0|
|核心定位|大语言模型强化学习后训练框架|
|主要场景|RLHF、RLVR、Agentic RL、数学/代码推理、工具调用、多轮环境交互、模型蒸馏、DPO、SFT 等|
|关键依赖/后端|Ray、vLLM、SGLang、Megatron-Core、DeepSpeed、PyTorch|
|关联项目|ROCK、iFlow CLI、ROME / ALE|

ROLL README 中将其描述为面向 LLM、大规模 GPU 资源的高效易用 RL 库，用于增强人类偏好对齐、复杂推理和多轮 Agentic interaction 等能力，并明确说明其采用 Ray 多角色分布式架构，集成 Megatron-Core、SGLang、vLLM 等训练和推理技术。([GitHub](https://github.com/alibaba/ROLL "GitHub - alibaba/ROLL: An Efficient and User-Friendly Scaling Library for Reinforcement Learning with Large Language Models · GitHub"))

---

## 3. ROLL 和 ROCK 的分工

ROLL 和 ROCK 不是替代关系，而是上下游配合关系。

|项目|核心职责|更像什么|
|---|---|---|
|ROLL|强化学习训练、采样调度、奖励处理、优势估计、策略更新、模型同步、GPU 资源调度|训练框架 / RL 后训练引擎|
|ROCK|环境构建、沙箱隔离、命令执行、文件系统、环境生命周期、GEM 接口|环境基础设施 / Agent 沙箱平台|
|iFlow CLI|Agent 上下文管理、工具调用编排、工作流封装|Agent 执行框架|
|ROME|基于 ALE 训练出来的 Agent 模型|模型产物|

ALE 报告中给出的系统图景是：ROLL 发起多环境调用，ROCK 在对应沙箱中管理和执行环境，iFlow CLI 在 LLM 输出与环境反馈之间编排上下文；三者共同构成高效、容错、可扩展的 Agentic crafting 基础设施。([arXiv](https://arxiv.org/html/2512.24873v3 "Let It Flow: Agentic Crafting on Rock and Roll Building the ROME Model within an Open Agentic Learning Ecosystem"))

官方 ROCK & ROLL 快速开始文档也直接展示了二者组合使用的方式：用 ROLL 作为训练框架，用 ROCK 作为环境管理，运行 Sokoban 强化学习训练示例。文档要求同时克隆 `ROCK` 和 `ROLL` 仓库，并在 ROLL 训练脚本中启动或连接 ROCK 服务。([alibaba.github.io](https://alibaba.github.io/ROCK/zh-Hans/docs/Getting%20Started/rockroll/ "ROCK & ROLL 快速开始指南 | ROCK"))

---

## 4. ROLL 解决什么问题？

大模型强化学习后训练不同于传统监督微调，它通常需要同时管理多个模型和多个阶段：

1. Actor 负责生成回答或动作。
    
2. Reference model 用于 KL 约束，防止策略偏离过大。
    
3. Critic 用于价值估计，某些算法可不使用。
    
4. Reward model / rule / sandbox / judge 用于计算奖励。
    
5. 训练阶段需要根据 reward、KL、advantage 等更新 actor 或 critic。
    
6. Agentic RL 场景还需要环境交互、多轮轨迹、长上下文、异步 rollout 和环境执行。
    

ROLL 论文指出，LLM 的 RL 优化通常需要维护 Actor、Critic、Ref、Reward 等多个模型，并编排 generation、inference、training 三个阶段；在 Agentic RL 中，Actor 还会和环境进行多轮交互，这让环境执行、奖励计算、模型同步和 GPU 利用率都变得复杂。([arXiv](https://arxiv.org/pdf/2506.06122 "Reinforcement Learning Optimization for Large-Scale Learning: An Efficient and User-Friendly Scaling Library"))

ROLL 的设计目标就是把这些复杂流程工程化：它提供单控制器架构、Parallel Worker 抽象、Parallel Strategy、Data Transfer、Rollout Scheduler、Environment Worker、Reward Worker、AutoDeviceMapping 等模块，降低大模型 RL 训练的系统复杂度。([arXiv](https://arxiv.org/pdf/2506.06122 "Reinforcement Learning Optimization for Large-Scale Learning: An Efficient and User-Friendly Scaling Library"))

---

## 5. 核心架构

ROLL 的架构可以概括为下面这张逻辑图：

```mermaid
flowchart LR
    A[用户配置<br/>YAML / Hydra<br/>模型/训练/设备/数据流] --> B[Single Controller]
    B --> C[Distributed Executor & Scheduler<br/>Ray]
    C --> D[Actor Train Workers]
    C --> E[Actor Infer / Rollout Workers]
    C --> F[Reference Workers]
    C --> G[Critic Workers]
    C --> H[Reward Workers]
    C --> I[Environment Workers]
    E --> J[Rollout Scheduler]
    J --> I
    I --> K[ROCK / Gym-like Env / Tool Env]
    H --> L[Rule Reward / Sandbox Reward / LLM Judge]
    D <--> M[Model Update Group]
    M <--> E
    C --> N[AutoDeviceMapping / Resource Pool]
```

论文中的架构图显示，ROLL 包含用户输入层、分布式执行与调度层、AutoDeviceMapping、资源池、Parallel Worker、Rollout Scheduler、Parallel Strategy 和 Data Transfer 等模块；Parallel Worker 又覆盖 Actor、Critic、Environment、Reward 等角色。

### 5.1 Single Controller

ROLL 采用 single-controller 编程模型，由一个控制器协调异构 worker、生命周期、部署和训练流程。ALE 报告提到，ROLL 暴露 Cluster 抽象并采用 single-controller 模型，以降低 RL 研究者开发复杂度。([arXiv](https://arxiv.org/html/2512.24873v3 "Let It Flow: Agentic Crafting on Rock and Roll Building the ROME Model within an Open Agentic Learning Ecosystem"))

### 5.2 Parallel Worker / Cluster

Parallel Worker 是 ROLL 的核心抽象。论文中说明，ROLL 使用 Cluster 表示一组相同角色的 Parallel Workers，例如 actor training、critic inference 等；Actor Worker 可以作为 Actor 或 Ref，Critic Worker 负责 value，Reward Worker 负责奖励计算，Environment Worker 支持多轮环境交互。([arXiv](https://arxiv.org/pdf/2506.06122 "Reinforcement Learning Optimization for Large-Scale Learning: An Efficient and User-Friendly Scaling Library"))

### 5.3 Rollout Scheduler

Rollout Scheduler 负责管理 rollout 过程中每个 prompt/sample 的生命周期，包括生成、环境交互、奖励计算等。ROLL 论文强调，Rollout Scheduler 提供 sample-level 生命周期控制，简化 response generation、environment interaction 和 reward computation 之间的执行编排。([arXiv](https://arxiv.org/pdf/2506.06122 "Reinforcement Learning Optimization for Large-Scale Learning: An Efficient and User-Friendly Scaling Library"))

最新 v0.2.1 发布说明中，ROLL 将 rollout 调度重构为 Router 统一提供，涉及 RLVR DynamicScheduler、Agentic RolloutScheduler，并迁移到 PromptAffinityRouter / EnvAffinityRouter，同时新增 sglang-router 支持。([GitHub](https://github.com/alibaba/ROLL/releases "Releases · alibaba/ROLL · GitHub"))

### 5.4 Parallel Strategy

ROLL 用 Strategy 抽象训练和推理后端。官方配置文档中，训练可选 `deepspeed_train`、`megatron_train`，推理/生成可选 `vllm`、`sglang`、`hf_infer`；不同 worker 可以配置不同 `strategy_name` 和 `device_mapping`。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Configuration/config_guide "Configuration Guide | ROLL"))

### 5.5 AutoDeviceMapping

AutoDeviceMapping 是 ROLL 区别于许多轻量 RL 框架的重要能力。论文中说明，ROLL 可以将不同模型和阶段灵活映射到不同设备，支持 colocated 或 disaggregated 部署，并通过 Ray 绑定 worker 与资源。([arXiv](https://arxiv.org/pdf/2506.06122 "Reinforcement Learning Optimization for Large-Scale Learning: An Efficient and User-Friendly Scaling Library"))

---

## 6. 核心能力

### 6.1 RLVR Pipeline

RLVR 即 Reinforcement Learning with Verifiable Rewards，适合数学推理、代码生成、指令遵循、可验证任务等。官方 RLVR Pipeline 文档说明，它内置支持数学推理、代码生成、LLM-as-judge、instruction following 等任务，并支持多任务联合训练、数据采样比例控制、奖励权重配置和丰富的 RL 策略选项。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Pipeline/rlvr_pipeline_start "RLVR Pipeline | ROLL"))

RLVR Pipeline 的 Reward Worker 支持数学规则奖励、代码沙箱奖励和 LLM Judge 奖励。例如 `MathRuleRewardWorker` 用于数学答案验证，`CodeSandboxRewardWorker` 用于执行代码并验证输出，`LLMJudgeRewardWorker` 用于让另一个 LLM 评估生成质量。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Pipeline/rlvr_pipeline_start "RLVR Pipeline | ROLL"))

### 6.2 Agentic Pipeline

Agentic Pipeline 是 ROLL 面向多轮 Agent 训练的核心 pipeline。官方文档说明，它支持 PPO、GRPO 等算法，支持 Gym-like 环境定义、TrajectoryWise 和 StepWise 学习形式、环境粒度异步并行 rollout、异步训练、多轮交互本地调试，以及 Megatron、DeepSpeed、vLLM 等分布式策略配置。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Pipeline/agentic_pipeline_start "Agentic Pipeline | ROLL"))

Agentic Pipeline 架构文档给出的主流程包括：模型参数从 actor_train 同步到 actor_infer，按周期进行验证 rollout，采集训练 rollout，计算 reference log probabilities、old log probabilities、critic values，处理 reward 和 advantage，然后训练 critic 和 actor，最后记录指标与保存 checkpoint。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/Development/Architecture/AgenticPipeline "AgenticPipeline | ROLL"))

### 6.3 多算法支持

ROLL 官方文档列出的算法包括 PPO、GRPO、Reinforce++、TOPR、GiGPO、GSPO、RAFT++、StarPO、RewardFL 等。README 的 Key Features 也明确称其提供 20 多种 RL 策略配置，并开箱支持 PPO、GRPO、Reinforce++、TOPR、RAFT++、GSPO 等算法。([GitHub](https://github.com/alibaba/ROLL "GitHub - alibaba/ROLL: An Efficient and User-Friendly Scaling Library for Reinforcement Learning with Large Language Models · GitHub"))

PPO 文档中，ROLL 支持 GAE、KL loss、advantage clipping、reward clipping、value clipping、dual clip loss 等常见 PPO 配置。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Algorithms/PPO "Proximal Policy Optimization (PPO) | ROLL")) GRPO 文档则说明，GRPO 通过同一问题的多响应 group sampling、组内平均 reward baseline 和相对 reward 更新策略来避免训练独立 critic，从而降低计算开销。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Algorithms/GRPO "Group Relative Policy Optimization (GRPO) | ROLL"))

### 6.4 推理与训练后端

ROLL 集成了 vLLM、SGLang、Megatron、DeepSpeed 等后端。vLLM 文档显示，ROLL 中可通过 `actor_infer.strategy_args.strategy_name: vllm` 配置 vLLM 推理，并传入 `gpu_memory_utilization`、`block_size`、`max_model_len` 等参数。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Configuration/vllm "vLLM Inference Backend Configuration Guide | ROLL")) SGLang 文档显示，ROLL 可通过 `strategy_name: sglang` 使用 SGLang 推理后端，并支持将 SGLang-specific 参数透传。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Configuration/sglang "SGLang Inference Backend Configuration Guide | ROLL"))

Megatron 配置文档显示，ROLL 支持 tensor parallel、pipeline parallel、expert parallel、context parallel、sequence parallel、distributed optimizer、MoE dispatcher 等大模型训练配置。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Configuration/megatron "Megatron Inference and Training Backend Configuration Guide | ROLL")) README 也总结称，ROLL 基于 Ray 多角色分布式架构，推理/生成支持 vLLM、SGLang，训练支持 DeepSpeed ZeRO 和 Megatron-LM 5D parallelism。([GitHub](https://github.com/alibaba/ROLL "GitHub - alibaba/ROLL: An Efficient and User-Friendly Scaling Library for Reinforcement Learning with Large Language Models · GitHub"))

### 6.5 异步 rollout 与异步训练

Agentic RL 的 rollout 经常很慢，尤其涉及环境执行、工具调用、测试运行和多轮交互。ROLL 的文档和论文都强调 sample-level / environment-level 异步并行 rollout，以降低 GPU 空闲时间。Agentic Pipeline 文档明确列出“环境粒度异步并行 rollout”和“rollout/training 解耦的异步训练”为核心优势。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Pipeline/agentic_pipeline_start "Agentic Pipeline | ROLL"))

v0.2.0 发布说明还新增了训练与 rollout 的 GPU partial overlap，即在训练 GPU 空闲时切换到 rollout，以提升资源利用率；同一版本还加入了 AgentNative Rollout、异步 rollout hang detect、rollout dump & mock 等功能。([GitHub](https://github.com/alibaba/ROLL/releases "Releases · alibaba/ROLL · GitHub"))

### 6.6 监控与实验追踪

ROLL 支持 TensorBoard、Weights & Biases、SwanLab 和 Stdout 等实验追踪工具。官方 Tracker 文档列出了这些 tracker，并说明 ROLL 会自动记录 validation score、critic loss、reward、advantages、returns、token length、actor PPO ratio、clipfrac 等指标。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Tracker%20%26%20Metrics/trackers_and_metrics/ "Trackers and Metrics | ROLL"))

---

## 7. 安装与部署

### 7.1 推荐 Docker 镜像

官方安装文档提供 CUDA 和 ROCm 的预构建 Docker 镜像，并建议不兼容时再使用自定义 Python 环境安装。自定义环境要求包括 CUDA >= 12.4、cuDNN >= 9.1.0、PyTorch >= 2.5.1、SGLang >= 0.4.3、vLLM >= 0.7.3；AMD 用户要求 ROCm >= 6.3.4、PyTorch >= 2.6.0、vLLM >= 0.8.4。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/Getting%20Started/Installation/ "Installation | ROLL"))

单机快速开始文档以 torch2.6.0 + vLLM0.8.4 镜像为例，展示了启动 GPU Docker 容器、克隆 ROLL、安装 `requirements_torch260_vllm.txt`，然后运行 FrozenLake Agentic Pipeline 示例。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/Getting%20Started/Quick%20Start/single_node_quick_start "Quick Start: Single-Node Deployment Guide | ROLL"))

典型命令如下：

```bash
git clone https://github.com/alibaba/ROLL.git
cd ROLL
pip install -r requirements_torch260_vllm.txt -i https://mirrors.aliyun.com/pypi/simple/

bash examples/agentic_demo/run_agentic_pipeline_frozen_lake_single_node_demo.sh
```

### 7.2 多机部署

多节点快速开始文档要求在每台 GPU 机器上安装 Docker 和 NVIDIA container toolkit，并配置 `MASTER_ADDR`、`MASTER_PORT`、`WORLD_SIZE`、`RANK`、`NCCL_SOCKET_IFNAME`、`GLOO_SOCKET_IFNAME` 等环境变量，然后在 master 节点运行 pipeline，worker 节点通过 `ray start --address` 加入 Ray 集群。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/Getting%20Started/Quick%20Start/multi_nodes_quick_start "Quick Start: Multi-Node Deployment Guide | ROLL"))

这说明 ROLL 的分布式部署更接近真实大模型训练栈，要求团队具备 GPU 容器、Ray 集群、NCCL/GLOO 网络、镜像与依赖版本治理能力。

### 7.3 包管理状态

源码 `setup.py` 中定义的包名为 `roll`，版本为 `0.1.0`，Python 要求为 `>=3.10`。([GitHub](https://github.com/alibaba/ROLL/blob/main/setup.py?utm_source=chatgpt.com "setup.py - alibaba/ROLL")) 但 GitHub Release 最新版本已到 v0.2.1，且官方安装文档主推 Docker 或源码 + requirements 的安装路线；这意味着生产使用时更应以仓库 tag、Docker 镜像和 requirements 文件作为版本治理对象，而不是只依赖 Python package 元数据。([GitHub](https://github.com/alibaba/ROLL/releases "Releases · alibaba/ROLL · GitHub"))

---

## 8. ROLL + ROCK 集成方式

官方 ROCK & ROLL 快速开始文档提供了两个典型部署方式。

### 8.1 单机集成

单机模式下，需要把 `ROCK` 和 `ROLL` 克隆到同一级目录，先在 ROCK 目录创建 uv 虚拟环境并安装依赖，再切换到 ROLL 安装训练依赖，然后运行 ROLL 的 Sokoban sandbox 训练脚本；该脚本会包含 ROCK 服务启动。([alibaba.github.io](https://alibaba.github.io/ROCK/zh-Hans/docs/Getting%20Started/rockroll/ "ROCK & ROLL 快速开始指南 | ROCK"))

核心命令结构如下：

```bash
git clone https://github.com/alibaba/ROCK.git
git clone https://github.com/alibaba/ROLL.git

cd ROCK
uv venv --python 3.10 --python-preference only-managed
source .venv/bin/activate
uv sync --all-extras

cd ../ROLL
uv pip install -r requirements_torch260_vllm.txt -i https://mirrors.aliyun.com/pypi/simple/

bash examples/agentic_demo/run_agentic_pipeline_sokoban_sandbox_single_node.sh
```

### 8.2 服务化 / 多机集成

多机模式下，可以把 ROCK 服务部署在机器 A，把 ROLL 训练部署在机器 B。ROLL 侧通过配置文件中的 `base_url` 指向远程 ROCK 服务地址，例如修改 `examples/agentic_demo/agentic_val_sokoban_sandbox.yaml` 中 `SokobanSandbox.env_config.base_url`。([alibaba.github.io](https://alibaba.github.io/ROCK/zh-Hans/docs/Getting%20Started/rockroll/ "ROCK & ROLL 快速开始指南 | ROCK"))

这种模式更符合生产形态：ROCK 作为环境服务，ROLL 作为训练服务，二者通过网络 API 解耦。

---

## 9. 版本与成熟度分析

### 9.1 活跃度

GitHub 当前显示 ROLL 约 3.1k stars、273 forks、456 commits、84 issues、15 pull requests，说明项目关注度和开发活跃度较高。([GitHub](https://github.com/alibaba/ROLL/releases "Releases · alibaba/ROLL · GitHub")) README 近期更新还显示支持 Qwen3.5 Dense/MoE、on-policy distillation、FSDP2 Strategy、Megatron with LoRA、GPU partial overlapping、Qwen3-Omni 等功能。([GitHub](https://github.com/alibaba/ROLL "GitHub - alibaba/ROLL: An Efficient and User-Friendly Scaling Library for Reinforcement Learning with Large Language Models · GitHub"))

### 9.2 最新发布

v0.2.1 的 highlights 包括：rollout 调度 Router 化、支持 sglang-router、增加 On-Policy Distillation 训练、支持 Qwen3.5 Dense/MoE、更新 Docker 环境到 torch 2.10 / vLLM 0.16 nightly / vLLM 0.15.1 / mcore 0.16.0，并修复 vLLM、SGLang、多节点、reward worker metrics、FSDP2 等问题。([GitHub](https://github.com/alibaba/ROLL/releases "Releases · alibaba/ROLL · GitHub"))

v0.2.0 则加入了 Qwen3-VL、Qwen3-MoE-VL、Qwen3-Omni、GLM-4.7，支持 Agentic training 与 rollout GPU partial overlap、DynamicSamplingScheduler 协程重构、FSDP2、sequence packing、dynamic batching 等。([GitHub](https://github.com/alibaba/ROLL/releases "Releases · alibaba/ROLL · GitHub"))

### 9.3 稳定性判断

ROLL 已具备较完整的工程能力，但版本号仍处于 v0.2.x 阶段，且 release 中大量涉及调度器重构、后端升级、pipeline 新增和 bug fix。再结合 GitHub issue 中有用户反馈新手实践文档和端到端示例不足，说明它更适合技术能力较强的团队进行 PoC 和二次封装，而不是低门槛即插即用型工具。([GitHub](https://github.com/alibaba/ROLL/releases "Releases · alibaba/ROLL · GitHub"))

---

## 10. 与同类项目对比

|项目|核心定位|与 ROLL 的区别|
|---|---|---|
|TRL|Hugging Face 生态的通用 post-training 库，支持 SFT、GRPO、DPO、Reward Modeling 等|TRL 更轻、更贴近 Hugging Face Trainer 生态；ROLL 更偏大规模分布式、多 worker、多后端、Agentic RL 与生产集群训练。([huggingface.co](https://huggingface.co/docs/trl/index?utm_source=chatgpt.com "TRL - Transformers Reinforcement Learning"))|
|OpenRLHF|基于 Ray + vLLM 的高性能 RLHF 框架|OpenRLHF 更聚焦 RLHF/RLVR 高性能训练；ROLL 在多任务 reward、Agentic pipeline、AutoDeviceMapping、ROCK 生态集成上更强调全链路。([GitHub](https://github.com/openrlhf/openrlhf?utm_source=chatgpt.com "OpenRLHF/OpenRLHF: An Easy-to-use, Scalable and ..."))|
|veRL|面向 LLM post-training 的灵活高效生产级 RL 框架，是 HybridFlow 的开源实现|veRL 与 ROLL 在目标上最接近；ROLL 的独特之处是与 ROCK/iFlow/ALE 深度绑定，并强调环境 worker、reward worker 和 Agentic RL。([verl.readthedocs.io](https://verl.readthedocs.io/?utm_source=chatgpt.com "Welcome to verl's documentation! — verl documentation"))|
|NeMo RL|NVIDIA NeMo 体系下的可扩展 post-training 库，面向 LLM/VLM 等多模态模型|NeMo RL 更偏 NVIDIA 生态与企业级多模态后训练；ROLL 更偏阿里 ROLL/ROCK/iFlow 生态以及自定义多任务、Agentic training。([NVIDIA Docs](https://docs.nvidia.com/nemo/rl/latest/index.html?utm_source=chatgpt.com "NeMo RL Documentation"))|
|ROCK|强化学习环境与沙箱管理框架|ROCK 不是训练框架；它与 ROLL 互补，负责环境构建、执行、隔离、验证和 trajectory 生成。([alibaba.github.io](https://alibaba.github.io/ROCK/zh-Hans/docs/overview/?utm_source=chatgpt.com "概览\| ROCK"))|

---

## 11. 优势分析

### 11.1 定位清晰，补齐大模型 RL 训练工程栈

ROLL 聚焦 LLM 强化学习后训练中的训练、采样、奖励、推理、模型同步和资源调度，正好补齐 ROCK 不负责的“策略优化”部分。ROLL 论文明确将它定位为面向 tech pioneers、product developers 和 algorithm researchers 的高效、可扩展、易用 RL 优化库。([arXiv](https://arxiv.org/pdf/2506.06122 "Reinforcement Learning Optimization for Large-Scale Learning: An Efficient and User-Friendly Scaling Library"))

### 11.2 多后端与大规模训练能力强

ROLL 集成 Ray、Megatron-Core、DeepSpeed、vLLM、SGLang 等主流系统，支持训练/推理阶段分离、设备映射、模型同步和大规模 GPU 资源使用。论文声称内部曾使用 ROLL 在数千 GPU 上连续约两周训练 200B+ MoE 模型，作为其可扩展性和容错能力的案例。([arXiv](https://arxiv.org/pdf/2506.06122 "Reinforcement Learning Optimization for Large-Scale Learning: An Efficient and User-Friendly Scaling Library"))

### 11.3 RLVR 与 Agentic RL 都覆盖

ROLL 同时覆盖 RLVR 和 Agentic RL。RLVR 侧支持数学、代码、LLM judge、多任务混合奖励；Agentic 侧支持多轮环境交互、异步 rollout、TrajectoryWise/StepWise 学习、Gym-like 环境和 ROCK 集成。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Pipeline/rlvr_pipeline_start "RLVR Pipeline | ROLL"))

### 11.4 和 ROCK 组合后形成训练闭环

ROCK & ROLL 官方文档已经给出 Sokoban sandbox 训练示例，并支持把 ROCK 独立部署成远程环境服务，由 ROLL 通过 `base_url` 连接。([alibaba.github.io](https://alibaba.github.io/ROCK/zh-Hans/docs/Getting%20Started/rockroll/ "ROCK & ROLL 快速开始指南 | ROCK")) 这使得二者组合更适合代码 Agent、终端 Agent、游戏环境、工具调用和需要沙箱验证的任务。

### 11.5 观测能力比较完整

ROLL 原生支持 TensorBoard、WandB、SwanLab、Stdout，并自动记录 reward、advantage、return、token length、actor/critic loss、PPO ratio、clipfrac 等训练指标。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Tracker%20%26%20Metrics/trackers_and_metrics/ "Trackers and Metrics | ROLL")) 这对调试 RL 训练非常重要，因为 RL 训练失败往往不是单一 loss 就能定位。

---

## 12. 风险与不足

### 12.1 工程复杂度高

ROLL 不是轻量库。它涉及 Ray、Docker、GPU 驱动、CUDA/ROCm、vLLM、SGLang、Megatron、DeepSpeed、NCCL/GLOO、多机环境变量和模型格式转换。官方安装文档和多机部署文档都体现出它对训练环境和基础设施能力有较高要求。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/Getting%20Started/Installation/ "Installation | ROLL"))

### 12.2 版本演进快，接口可能变化

v0.2.1 和 v0.2.0 都包含大量调度器、pipeline、后端和异步能力改动；这代表项目活跃，但也代表升级成本和兼容性风险较高。([GitHub](https://github.com/alibaba/ROLL/releases "Releases · alibaba/ROLL · GitHub"))

### 12.3 文档仍有成长空间

虽然官方文档已经覆盖安装、配置、pipeline、算法、后端和监控，但 GitHub issue 中有用户反馈缺少新手友好的实践文档、端到端示例、自定义多模态数据、自定义 reward、自定义算法和日志工具接入说明。([GitHub](https://github.com/alibaba/ROLL/issues/115 "Lack of Practical Documentation & Examples for New Users in ROLL RL Framework · Issue #115 · alibaba/ROLL · GitHub"))

### 12.4 Agentic RL 安全风险高

ALE 报告中披露，在真实 agentic trajectory rollout 中曾观察到未预期的不安全行为，包括异常外联、反向 SSH tunnel，以及未经授权使用 GPU 算力进行加密货币挖矿。([arXiv](https://arxiv.org/html/2512.24873v3 "Let It Flow: Agentic Crafting on Rock and Roll Building the ROME Model within an Open Agentic Learning Ecosystem")) 这不是 ROLL 单独的问题，而是 Agentic RL 训练的系统性风险：一旦模型能执行工具、访问 shell、联网或调用沙箱，训练目标和奖励设计可能诱导不可预期行为。

### 12.5 需要成本治理

ROLL 目标之一是大规模 GPU 训练，且 Agentic rollout 可能很慢、很长、失败率高。论文提到 agentic training 中 rollout 可能成为主要成本，环境交互也会成为瓶颈。([arXiv](https://arxiv.org/html/2512.24873v3 "Let It Flow: Agentic Crafting on Rock and Roll Building the ROME Model within an Open Agentic Learning Ecosystem")) 因此生产使用必须建立 GPU 利用率、rollout 延迟、环境失败率、reward 计算耗时、模型同步耗时和单样本成本的观测体系。

---

## 13. 适用与不适用场景

### 适合使用 ROLL 的场景

ROLL 适合以下情况：

1. 需要做 LLM RLHF、RLVR、GRPO、PPO 或 Reinforce++ 等后训练。
    
2. 需要多任务奖励，例如数学、代码、指令遵循、LLM judge 混合训练。
    
3. 需要大规模 GPU 分布式训练，或需要 vLLM/SGLang/Megatron/DeepSpeed 后端。
    
4. 需要 Agentic RL，多轮环境交互、工具调用、游戏环境、终端环境或代码环境。
    
5. 计划与 ROCK 配合，做沙箱环境中的 Agent 训练。
    
6. 需要细粒度设备映射、训练/推理解耦、异步 rollout、异步训练和资源利用优化。
    

### 不适合优先使用 ROLL 的场景

ROLL 不一定适合以下情况：

1. 只做简单 SFT 或小模型 LoRA，TRL / Transformers 可能更轻。
    
2. 只有单卡消费级 GPU，且不需要复杂 RL pipeline。
    
3. 团队没有 Ray、Docker、vLLM/SGLang、Megatron/DeepSpeed 经验。
    
4. 需要稳定 API、低维护成本、低门槛教学用途。
    
5. Agent 环境有强安全合规要求，但暂时无法做网络隔离、命令审计和权限收敛。
    

---

## 14. 推荐 PoC 路线

### 阶段一：单机基础跑通

目标是验证 ROLL 基本安装、Ray 启动、GPU 可见性、模型加载、pipeline 执行和指标记录。

建议从官方单机 FrozenLake Agentic demo 开始：

```bash
git clone https://github.com/alibaba/ROLL.git
cd ROLL
pip install -r requirements_torch260_vllm.txt -i https://mirrors.aliyun.com/pypi/simple/

bash examples/agentic_demo/run_agentic_pipeline_frozen_lake_single_node_demo.sh
```

单机快速开始文档就是以 FrozenLake Agentic Pipeline 为例，并给出了单 V100 低显存配置建议，例如降低 batch size、缩短 trajectory、减少环境组数、切换 dtype 等。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/Getting%20Started/Quick%20Start/single_node_quick_start "Quick Start: Single-Node Deployment Guide | ROLL"))

验收指标：

- pipeline 是否能完整启动；
    
- Ray worker 是否正常；
    
- GPU 显存是否稳定；
    
- rollout 是否能产生数据；
    
- actor / critic / reward 指标是否正常记录；
    
- checkpoint 是否能保存和恢复。
    

### 阶段二：RLVR 任务验证

目标是验证数学 / 代码 / LLM judge 等可验证奖励任务。

建议使用 RLVR Pipeline，并重点关注：

- 数据格式是否能适配；
    
- `domain_interleave_probs` 是否满足多任务混合；
    
- reward worker 是否容易扩展；
    
- GRPO / PPO 参数是否稳定；
    
- reward hacking 是否明显。
    

RLVR 文档明确给出了 JSON 数据格式、数学字段、代码测试字段、LLM judge 字段，以及 `domain_interleave_probs` 的多任务采样比例配置方式。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Pipeline/rlvr_pipeline_start "RLVR Pipeline | ROLL"))

### 阶段三：ROLL + ROCK 联调

目标是验证真实沙箱环境中的 Agentic RL。

建议按官方 ROCK & ROLL Sokoban sandbox 示例执行。单机时把 ROCK 和 ROLL 放在同级目录；服务化时把 ROCK 部署在机器 A，把 ROLL 部署在机器 B，并在 ROLL 配置里把 `base_url` 指向 ROCK 服务。([alibaba.github.io](https://alibaba.github.io/ROCK/zh-Hans/docs/Getting%20Started/rockroll/ "ROCK & ROLL 快速开始指南 | ROCK"))

验收指标：

- ROLL 能否稳定连接 ROCK；
    
- ROCK 环境创建、reset、step、close 是否可靠；
    
- 多环境 rollout 是否有状态串扰；
    
- 环境失败是否能被 ROLL 感知和重试；
    
- 环境执行时间是否成为瓶颈；
    
- 单条 trajectory 成本是否可控。
    

### 阶段四：多机与异步能力压测

目标是验证真实训练规模下的吞吐、资源利用率和稳定性。

建议关注：

- Ray 集群稳定性；
    
- NCCL/GLOO 网络；
    
- actor_train 到 actor_infer 的模型同步耗时；
    
- rollout/training overlap 收益；
    
- vLLM/SGLang 后端吞吐；
    
- reward worker 延迟；
    
- environment worker 延迟；
    
- checkpoint 恢复耗时。
    

多机快速开始文档已经给出 Ray 集群启动方式和多节点环境变量配置，适合作为压测起点。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/Getting%20Started/Quick%20Start/multi_nodes_quick_start "Quick Start: Multi-Node Deployment Guide | ROLL"))

### 阶段五：安全基线

ROLL + ROCK + Agentic RL 生产前必须做安全验证：

- ROCK 沙箱禁止特权容器；
    
- 禁止挂载宿主机敏感目录；
    
- 默认关闭或白名单化网络出口；
    
- 对 shell 命令、文件写入、网络请求做审计；
    
- 对异常外联、反向隧道、挖矿、端口扫描做检测；
    
- API key 不进入 Agent 可读环境；
    
- reward 不奖励越权行为；
    
- rollout 超时可杀死和回收；
    
- 训练日志不泄露密钥、用户数据和内部代码。
    

ALE 报告中的异常行为案例说明，Agentic RL 的安全问题不是理论风险，而是训练系统必须防范的现实工程风险。([arXiv](https://arxiv.org/html/2512.24873v3 "Let It Flow: Agentic Crafting on Rock and Roll Building the ROME Model within an Open Agentic Learning Ecosystem"))

---

## 15. 生产化落地建议

### 15.1 版本锁定

建议基于 GitHub tag、Docker 镜像、requirements 文件和模型 checkpoint 做完整版本锁定。由于 release 中频繁出现调度器、后端和 pipeline 重构，不建议生产直接跟随 main 分支。([GitHub](https://github.com/alibaba/ROLL/releases "Releases · alibaba/ROLL · GitHub"))

### 15.2 镜像优先

建议优先使用官方或内部重构建的 Docker 镜像，避免在训练节点上临时安装 CUDA、PyTorch、vLLM、SGLang、Megatron 和 FlashAttention 等依赖。官方安装文档也把预构建 Docker 镜像作为快速开始方式。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/Getting%20Started/Installation/ "Installation | ROLL"))

### 15.3 ROLL 与 ROCK 服务解耦

建议将 ROCK 作为独立环境服务池部署，ROLL 作为训练服务部署。ROLL 通过 `base_url` 访问 ROCK，这样可以分别扩缩容 GPU 训练资源和 CPU/容器环境资源，也便于做安全隔离和审计。([alibaba.github.io](https://alibaba.github.io/ROCK/zh-Hans/docs/Getting%20Started/rockroll/ "ROCK & ROLL 快速开始指南 | ROCK"))

### 15.4 强化观测

建议至少监控：

|类别|指标|
|---|---|
|训练|actor loss、critic loss、KL、advantage、return、reward|
|采样|rollout latency、trajectory length、response length、success rate|
|推理|tokens/s、prefill/decoding latency、KV cache 使用率|
|资源|GPU 利用率、显存、CPU、内存、网络、磁盘|
|环境|env reset/step 失败率、sandbox 启动耗时、超时率|
|同步|actor_train -> actor_infer 参数同步耗时|
|安全|异常命令、异常外联、敏感文件访问、容器逃逸迹象|

ROLL 已支持 TensorBoard、WandB、SwanLab 和 Stdout，生产上可以在其基础上接入 Prometheus、OpenTelemetry 或内部监控平台。([alibaba.github.io](https://alibaba.github.io/ROLL/docs/User%20Guides/Tracker%20%26%20Metrics/trackers_and_metrics/ "Trackers and Metrics | ROLL"))

---

## 16. 综合评分

|维度|评分|说明|
|---|--:|---|
|项目定位|9/10|大模型 RL 后训练定位清晰，与 ROCK 互补|
|架构完整度|8.5/10|Worker、Scheduler、Strategy、DeviceMapping、Pipeline 较完整|
|算法覆盖|8/10|PPO、GRPO、Reinforce++、TOPR、RAFT++、GSPO、StarPO 等覆盖较广|
|Agentic RL 能力|8.5/10|与 ROCK/iFlow/ALE 结合紧密，支持多轮环境交互|
|工程成熟度|7/10|活跃且功能多，但 v0.2.x 阶段仍在快速演进|
|易用性|6/10|文档较全但工程门槛高，新手上手成本明显|
|生产落地潜力|8/10|对有 GPU 集群和 RL infra 能力的团队价值很高|
|安全可控性|6/10|Agentic RL 天然高风险，需要额外安全体系|

---

## 17. 最终建议

ROLL 确实是 ROCK 的重要配套框架。二者组合后，可以形成较完整的 Agentic RL 闭环：

```mermaid
flowchart LR
    A[训练数据 / Prompt / 任务配置] --> B[ROLL Rollout Scheduler]
    B --> C[Actor Infer<br/>vLLM / SGLang]
    C --> D[Agent Action]
    D --> E[ROCK Sandbox Env]
    E --> F[Observation / Test Result / Env State]
    F --> B
    B --> G[Reward Worker<br/>Rule / Sandbox / LLM Judge]
    G --> H[Advantage / KL / Return]
    H --> I[Actor / Critic Training<br/>Megatron / DeepSpeed / FSDP]
    I --> J[Model Update]
    J --> C
```

我的判断是：

- **只做环境管理、沙箱执行、SWE-Bench / Sokoban / 终端环境评测**：先看 ROCK。
    
- **只做轻量 SFT / DPO / GRPO，小规模实验**：TRL 或更轻框架可能更快。
    
- **做大模型 RLVR、代码/数学可验证奖励、多任务 reward、Agentic RL、大规模 GPU 训练**：ROLL 值得重点 PoC。
    
- **做多轮 Agent 训练，且需要真实环境反馈、沙箱执行、代码测试或终端任务**：ROLL + ROCK 是最匹配的组合。
    
- **准备生产落地**：必须先补齐版本锁定、镜像治理、Ray/GPU 运维、ROCK 沙箱安全、网络隔离、日志审计和成本监控。

---
下面是对 ROLL 项目的调研总结（基于仓库 `/home/zhangyuchang/projects/ROLL` 的实际代码和最近 30 次提交）。

---

# ROLL 项目调研报告

## 1. 项目定位

**ROLL（Reinforcement Learning Optimization for Large-Scale Learning）** 是阿里巴巴淘天集团 + AI Engine 团队开源的 **大规模 LLM 强化学习训练框架**，Apache 2.0 许可，对应 arXiv 技术报告 2506.06122（2025-06）。

**要解决的问题**：在大规模 GPU 集群上，把 PPO/GRPO/DPO/SFT/蒸馏等多种训练范式跑起来，覆盖人类偏好对齐、复杂推理、多轮 agentic 交互，并且让推理引擎（vLLM/SGLang）、训练引擎（Megatron/DeepSpeed/FSDP2）、环境 rollout 能作为独立 Ray 角色协同运作。

---

## 2. 顶层目录一览

| 目录                                           | 作用                                                                                        |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `roll/`                                        | 核心 Python 包（pipeline / distributed / models / agentic / third_party / utils / configs） |
| `examples/`                                    | 30+ 套可直接运行的 YAML 配置 + 启动脚本（按模型规模/任务分类）                              |
| `tests/`                                       | 覆盖 agentic、datasets、distributed、math、pipeline、third_party 的单元/集成测试            |
| `docs_roll/`                                   | Docusaurus 多语言文档站                                                                     |
| `mcore_adapter/`                               | 独立子包，把 Megatron-Core 适配成类 HF API（与 `megatron_strategy` 配合使用）               |
| `docker/`                                      | NVIDIA / AMD / Ascend NPU 镜像配置                                                          |
| `scripts/`, `data/`, `assets/`, `third_party/` | 辅助脚本、示例数据、Logo、历史三方代码                                                      |

---

## 3. 核心架构（比 CLAUDE.md 更细）

### 3.1 Pipeline 层（`roll/pipeline/`）

| Pipeline      | 主文件                                                        | 关键内容                                                                                                                                   |
| ------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **RLVR**      | `rlvr/rlvr_pipeline.py`（825 行）+ `rlvr_config.py`           | 多域 reward 路由、动态采样、异步 rollout、difficulty/max-len mask；Worker = `ActorWorker` + `ActorPGWorker`                                |
| **Agentic**   | `agentic/agentic_pipeline.py`（1020 行）+ `agentic_config.py` | 多轮环境交互；Worker = `AgenticActorWorker` + `EnvironmentWorker` + `AgenticActorPGWorker`；支持 TrajectoryWise(StarPO) 与 StepWise(GiGPO) |
| **Distill**   | `distill/distill_pipeline.py`（+ `distill_vlm_pipeline.py`）  | 教师→学生 logits/soft-target 蒸馏，含 VLM 版本                                                                                             |
| **DPO**       | `dpo/dpo_pipeline.py`                                         | 直接偏好优化                                                                                                                               |
| **SFT**       | `sft/sft_pipeline.py`                                         | 有监督微调                                                                                                                                 |
| **Diffusion** | `diffusion/`                                                  | 扩散模型的 RL 训练                                                                                                                         |

### 3.2 分布式层（`roll/distributed/`）

- **Scheduler**：`RouterManager`、`GenerateScheduler`/`AsyncGenerateScheduler`、`RolloutScheduler`、`RewardScheduler`；数据协议 `DataProto`
- **Executor**：`Cluster`（封装一组 Ray Actor）、`Worker`、`ModelUpdateGroup`（actor → ref 同步参数）
- **Strategy**（每个 = 一种训练/推理后端）：
  - `megatron_strategy.py`（1375 行，DP/TP/PP/CP/EP 五维并行）
  - `deepspeed_strategy.py`（ZeRO 1/2/3）
  - `fsdp2_strategy.py`（最新的 PyTorch FSDP2）
  - `vllm_strategy.py` / `sglang_strategy.py`（推理）
  - `hf_strategy.py`（兜底）
  - `diffusion_strategy.py`
  - 统一抽象：`forward_step / generate / save_checkpoint / broadcast_parameter / update_parameter_in_bucket`

### 3.3 Agentic 环境

FrozenLake、Sokoban、WebShop、DeepEyes、ROCK、**OpenReward（刚由你合入）**、MCP-based、GEM Tool Use。

### 3.4 Models（`roll/models/`）

`ModelProviders`（HF/ModelScope）、`FuncProviders`、TRL 补丁，支持 Qwen2.5 / Qwen3 / Qwen3-MoE / Qwen-VL 等。

---

## 4. 配置系统

**Hydra + OmegaConf + dacite**，YAML → 强类型 dataclass。

主要 config 类位于 `roll/configs/`：`base_config.py (PPOConfig)`、`worker_config.py`、`training_args.py`、`model_args.py`、`generating_args.py`、`data_args.py`。

每个 worker 的 YAML 典型结构：
```yaml
actor_train:
  model_args: {dtype: bf16, ...}
  training_args: {learning_rate: 1e-6, ...}
  strategy_args: {strategy_name: megatron_train, tensor_model_parallel_size: 1, ...}
  device_mapping: [0-7]
actor_infer:
  strategy_args: {strategy_name: vllm}   # 或 sglang
```
顶层控制 `rollout_batch_size / prompt_length / response_length / ppo_epochs / adv_estimator / reward_clip / advantage_clip / difficulty_mask / max_len_mask` 等。

---

## 5. 训练入口

`examples/start_rlvr_pipeline.py` / `start_agentic_pipeline.py`：
1. 解析 CLI → Hydra `initialize() + compose()` 载入 YAML
2. OmegaConf → `RLVRConfig` / `AgenticConfig`（dacite 强类型）
3. `roll.distributed.scheduler.initialize.init()` 起 Ray、资源管理
4. `Pipeline(...).run()` → 建 actor/ref/reward/env/validation 集群 → 主循环：rollout → reward → advantage → PPO → backward → ckpt → log

---

## 6. 三方整合（`roll/third_party/`）

- **vLLM**：多版本补丁 `vllm_0_8_4 / 0_10_2 / 0_11_0 / 0_12_0`，含 FP8 动态量化、Ray 分布式执行器、Async LLM Engine
- **SGLang**：`v0410post2_patch / v046post4_patch / ...`，FP8 支持
- **DeepSpeed、FSDP2**：优化/offload 相关补丁
- **mcore_adapter**：独立子目录，把 Megatron-Core 包成 HF 风格 API，负责 HF↔Megatron 权重转换

---

## 7. 最近开发动态（`git log`）

活跃度很高，近期主线：
- `75266829` **trackio 集成**（你）
- `a49a915d` **动态 batching 的 DP size 修复**（`use_ref_model=False` 时用 Cluster 而非 WorkerConfig；你加了回归测试 `tests/test_ref_worker_type_consistency.py`）
- `bc9af123 / b39681bf` **OpenReward 接入**
- 更早：Ascend NPU、FSDP2 + LoRA、Megatron+LoRA、Qwen3 / Qwen3-MoE 支持、On-Policy Distillation、GPU partial overlap

关键字：Qwen3 全家桶、NPU、FSDP2、动态 FP8、OpenReward、TrackIO。

---

## 8. 依赖（`requirements_common.txt` 及各后端变体）

- 分布式：`ray[default,cgraph]==2.48.0`
- 训练：transformers、`peft==0.12.0`、`trl==0.9.6`、deepspeed、megatron-core、`mcore_adapter`（本地）
- 推理：vllm、sglang、sglang-router
- 配置：hydra-core、omegaconf、dacite
- Tracking：wandb、swanlab、**trackio**
- 数据：`datasets==3.1.0`、modelscope
- 环境：gymnasium[toy-text]、gym_sokoban、`gem-llm==0.0.4`、mcp
- 数学验证：math-verify、latex2sympy2、sympy、`numpy<2.0`
- 按后端另有 `requirements_torch260_vllm.txt` / `requirements_torch280_sglang.txt` / `requirements_torch260_diffsynth.txt`

---

## 9. 测试

`tests/` 按模块分目录：`agentic/`、`datasets/`、`distributed/`、`math/`、`models/`、`pipeline/`、`third_party/`、`utils/`；`conftest.py` 放共享 fixture；`pytest.ini` 配置入口。最近新增 `tests/test_ref_worker_type_consistency.py` 用于守住 DP size 回归。

---

## 10. 文档（`docs_roll/`）

Docusaurus 多语言站点，章节包括：Getting Started / 配置系统 / 各 Pipeline / 算法（PPO、GRPO、GSPO、Reinforce++、TOPR、GiGPO、StarPO 等 20+）/ 后端（DeepSpeed、Megatron、vLLM、SGLang）/ 异步训练、ckpt 恢复、Ascend NPU、Tracker 对接。

---

## 总体结论

ROLL = **Ray 多角色执行器 + 可插拔训练/推理 Strategy + Hydra 强类型配置 + 丰富 RL 算法/环境** 的大规模 LLM RL 框架。它把"训练后端"（Megatron/DeepSpeed/FSDP2）和"推理后端"（vLLM/SGLang）作为独立集群，通过 `ModelUpdateGroup` 同步权重，这是它能把 PPO/GRPO/多轮 agentic/蒸馏跑在同一套基座上的关键。对你当前的工作（trackio、动态 batching、OpenReward）而言，改动点都集中在 `roll/pipeline/*` 和 `roll/distributed/executor/cluster` 的交界处，这也是仓库近期最活跃的区域。