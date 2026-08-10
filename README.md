# Foam-Agent    <a href="https://arxiv.org/abs/2505.04997"><img src="https://img.shields.io/badge/arXiv-2505.04997-b31b1b.svg" alt="Paper"></a>
<p align="center">
  <img src="overview.png" alt="Foam-Agent 系统架构" width="800">
</p>

<p align="center">
    <em>一个端到端可组合的多智能体框架，用于自动执行 OpenFOAM CFD 仿真</em>
</p>

**Foam-Agent** 可以从单个自然语言提示自动执行整个基于 **OpenFOAM** 的 CFD 仿真工作流程。它负责管理网格划分、案例设置、运行执行、错误校正和后处理——极大降低了计算流体力学（CFD）的专业门槛。在包含 110 个仿真任务的 [FoamBench](https://arxiv.org/abs/2509.20374) 基准评估中，我们的框架使用 Claude Opus 4.6 达到了 **100% 的成功率**。

访问 [deepwiki.com/csml-rpi/Foam-Agent](https://deepwiki.com/csml-rpi/Foam-Agent) 获取详细介绍并交互式提问。

## 核心特性

- **端到端自动化**：从网格划分（包括外部 Gmsh `.msh` 文件）到 HPC 作业提交，再到 ParaView/PyVista 可视化——一个提示即可完成所有操作。
- **多智能体工作流**：Architect（架构师）、Input Writer（输入编写器）、Runner（运行器）和 Reviewer（审查器）智能体通过 LangGraph 流水线协作，具备自动错误校正功能（最多 25 次迭代）。
- **RAG 增强生成**：基于 OpenFOAM 教程构建的分层 FAISS 索引，为准确的配置文件生成提供上下文相关的检索。
- **可组合服务架构**：核心功能以 MCP 工具形式暴露，可与 Claude Code、Cursor 及其他智能体系统集成。

## 快速开始

### 1. 拉取并运行 Docker 镜像

```bash
docker run -it \
  -e OPENAI_API_KEY=your-key-here \
  -p 7860:7860 \
  --name foamagent \
  leoyue123/foamagent
```

容器预装了 OpenFOAM v10、Conda 和所有依赖项。

> 如需特定版本：`docker pull leoyue123/foamagent:v2.0.0`

### 2. 编写提示词

在容器内编辑 `user_requirement.txt`：

```text
do a Reynolds-Averaged Simulation (RAS) pitzdaily simulation. Use PIMPLE algorithm.
The domain is a 2D millimeter-scale channel geometry. Boundary conditions specify a
fixed velocity of 10m/s at the inlet (left), zero gradient pressure at the outlet
(right), and no-slip conditions for walls. Use timestep of 0.0001 and output every
0.01. Finaltime is 0.3. use nu value of 1e-5.
```

### 3. 运行

```bash
python foambench_main.py --output ./output --prompt_path ./user_requirement.txt
```

就这样。Foam-Agent 会自动规划案例、生成所有 OpenFOAM 文件、运行仿真并自动修复错误。

## 配置

所有设置在 `src/config.py` 中，并带有合理的默认值。每个设置都可以通过环境变量覆盖——无需编辑文件，特别适用于 Docker 和 CI 环境。

### LLM 提供商与模型

| 环境变量 | 用途 | 允许的值 |
|---|---|---|
| `FOAMAGENT_MODEL_PROVIDER` | LLM 后端 | `openai`、`openai-codex`、`anthropic`、`bedrock`、`ollama` |
| `FOAMAGENT_MODEL_VERSION` | 模型标识符 | 例如 `gpt-5-mini`、`gpt-5.3-codex`、`claude-opus-4-6` |

示例：
```bash
docker run -it \
  -e FOAMAGENT_MODEL_PROVIDER=anthropic \
  -e ANTHROPIC_API_KEY=your-key-here \
  -e FOAMAGENT_MODEL_VERSION=claude-opus-4-6 \
  -p 7860:7860 \
  leoyue123/foamagent
```

### Embedding 提供商与模型

| 环境变量 | 用途 | 允许的值 |
|---|---|---|
| `FOAMAGENT_EMBEDDING_PROVIDER` | Embedding 后端 | `openai`、`huggingface`、`ollama` |
| `FOAMAGENT_EMBEDDING_MODEL` | Embedding 模型 | 例如 `Qwen/Qwen3-Embedding-0.6B`、`text-embedding-3-small` |

默认使用 `huggingface` 和 `Qwen/Qwen3-Embedding-0.6B`（本地运行，无需 API 密钥）。

### API 密钥

| 变量 | 何时需要 |
|---|---|
| `OPENAI_API_KEY` | 使用 `openai` 提供商时 |
| `ANTHROPIC_API_KEY` | 使用 `anthropic` 提供商时 |
| AWS credentials | 使用 `bedrock` 提供商时 |

### Input Writer 生成模式

在 `src/config.py` 中通过 `input_writer_generation_mode` 设置：

| 模式 | 行为 | 最适合 |
|---|---|---|
| `sequential_dependency` | 按顺序生成文件，附带跨文件上下文 | 成本较高的运行（HPC、长时间仿真） |
| `parallel_no_context` | 并行生成文件，无跨文件上下文 | 快速本地运行，重试成本低 |

### 推荐模型

| 框架 | 模型 | 基础任务 | 高级任务 |
|---|---|---|---:|---:|
| FoamAgent 2.0.0 (10 轮循环) | Opus 4.6 | 85.45% | 100% |
| FoamAgent 2.0.0 (25 轮循环) | Opus 4.6 | 100% | 100% |
| FoamAgent 2.0.0 (25 轮循环) | Sonnet 4.6 | 87.88% | 75.00% |
| FoamAgent 2.0.0 (25 轮循环) | Haiku 4.6 | 54.55% | 37.50% |
| FoamAgent 2.0.0 (25 轮循环) | gpt-5.4 | 45.45% | 75.00% |
| FoamAgent 2.0.0 (25 轮循环) | gpt-5.3-codex | 54.55% | 62.50% |

推荐使用 **Anthropic Claude Opus 4.6** 以获得最佳效果。

## 高级用法

### 自定义网格文件

Foam-Agent 支持外部 Gmsh `.msh` 文件（ASCII 2.2 格式）。在提示中描述边界条件并传入网格文件：

```bash
python foambench_main.py \
  --output ./output \
  --prompt_path ./user_req_tandem_wing.txt \
  --custom_mesh_path ./tandem_wing.msh
```

将主机上的网格文件挂载到 Docker 中：

```bash
docker run -it \
  -e OPENAI_API_KEY=your-key-here \
  -v /path/to/my_mesh.msh:/home/openfoam/Foam-Agent/my_mesh.msh \
  -p 7860:7860 \
  leoyue123/foamagent
```

### Skill / MCP 集成（Claude Code、Cursor、Windsurf 等）

Foam-Agent 将其完整的 CFD 工作流以 **MCP 服务器** 的形式暴露——这是 Claude Code、Cursor、Windsurf 及其他 AI 驱动工具支持的通用协议。它同时附带一个 **Claude Code skill**（`/foam`），支持一键仿真运行。

#### 快速配置（本地安装）

```bash
# 1. 安装（添加 foamagent-mcp 命令）
pip install -e .

# 2. 注册到你的 AI 工具
claude mcp add foamagent -- foamagent-mcp                # Claude Code
```

对于 **Cursor**：打开 Settings > Features > MCP > Edit MCP Settings，添加：

```json
{
  "mcpServers": {
    "foamagent": {
      "command": "foamagent-mcp"
    }
  }
}
```

对于 **Windsurf / 其他 MCP 兼容工具**，使用上述相同的 JSON 配置。

#### 快速配置（Docker）

如果在 Docker 中运行，启动 HTTP 服务器并将 MCP 客户端指向它：

```bash
docker run -it \
  -e OPENAI_API_KEY=your-key-here \
  -p 7860:7860 \
  leoyue123/foamagent \
  foamagent-mcp --transport http --host 0.0.0.0 --port 7860
```

然后配置你的 MCP 客户端：

```json
{
  "mcpServers": {
    "foamagent": {
      "url": "http://localhost:7860/mcp"
    }
  }
}
```

> 如果在远程服务器上运行 Docker，请确保 7860 端口可访问（例如通过 SSH 端口转发或 `-p 7860:7860`）。

#### 可用的 MCP 工具

Foam-Agent 默认按照 **Foundation OpenFOAM v10** 规范生成输出。如果设置了
`FOAMAGENT_OPENFOAM_FORK=esi`，生成的输入文件将在返回前尽量转换为 ESI OpenFOAM
（`openfoam.com`）的命名和字典规范。运行/审查/修复工作流仍然主要在 Foundation OpenFOAM v10 中验证。

| 工具 | 描述 |
|------|------|
| `plan` | 分析需求并使用 Foundation v10 参考规划仿真结构 |
| `input_writer` | 生成 OpenFOAM 配置文件；当 `FOAMAGENT_OPENFOAM_FORK=esi` 时可选择转换生成的文件 |
| `run` | 本地执行 Allrun 脚本并收集错误信息；主要在 Foundation OpenFOAM v10 上验证 |
| `review` | 通过 LLM 使用 Foundation v10 参考分析仿真错误并建议修复方案 |
| `apply_fixes` | 根据审查分析重写 OpenFOAM 文件；ESI 案例仍然是尽力而为 |
| `visualization` | 生成仿真结果的 PyVista 可视化 |

#### Claude Code Skill

对于克隆了本仓库的 Claude Code 用户，`.claude/skills/foam.md` 中包含一个 `/foam` skill。它将 MCP 工具编排为完整的工作流：

```
/foam Simulate lid-driven cavity flow at Re=1000
```

这将触发完整流程：规划 -> 生成文件 -> 运行 -> 审查/修复循环 -> 可视化。

### Codex OAuth 登录（无需 API 密钥）

如果你有 ChatGPT/Codex 订阅，可以通过 OAuth 认证而非 API 密钥：

1. 在宿主机上安装 [Codex CLI](https://github.com/openai/codex)。
2. 运行 `codex login` 并选择 **"Sign in with ChatGPT"**。
3. 验证 Token 缓存存在：`ls ~/.codex/auth.json`
4. 将其挂载到容器中：

```bash
docker run -it \
  -e FOAMAGENT_MODEL_PROVIDER=openai-codex \
  -e FOAMAGENT_MODEL_VERSION=gpt-5.3-codex \
  -v ~/.codex/auth.json:/root/.codex/auth.json:ro \
  -p 7860:7860 \
  leoyue123/foamagent
```

Foam-Agent 在以下位置搜索 OAuth Token（优先使用先匹配的）：
- `$CODEX_HOME/auth.json`
- `~/.codex/auth.json`
- `~/.clawdbot/agents/main/agent/auth-profiles.json`

> 安全提示：`auth.json` 包含访问令牌，请像保护密码一样保护它。

### 手动安装（不使用 Docker）

```bash
git clone https://github.com/csml-rpi/Foam-Agent.git
cd Foam-Agent
conda env create -n FoamAgent -f environment.yml
conda activate FoamAgent
```

你还需要安装并加载 **Foundation OpenFOAM v10**（[openfoam.org](https://openfoam.org)）作为默认的、经过充分验证的运行环境。ESI OpenFOAM（`openfoam.com`）的文件生成可通过设置 `FOAMAGENT_OPENFOAM_FORK=esi` 进行尽力而为的转换，但 ESI 的运行和修复循环应逐案例验证。按照[官方 Foundation v10 安装指南](https://openfoam.org/version/10/)操作，并验证：

```bash
echo $WM_PROJECT_DIR   # 应输出例如 /opt/openfoam10
```

然后运行：

```bash
python foambench_main.py --output ./output --prompt_path ./user_requirement.txt
```

### 从源码构建 Docker 镜像

```bash
git clone https://github.com/csml-rpi/Foam-Agent.git
cd Foam-Agent
docker build -f docker/Dockerfile -t foamagent:latest .
docker run -it \
  -e OPENAI_API_KEY=your-key-here \
  -p 7860:7860 \
  foamagent:latest
```

## 常见问题排查

| 问题 | 解决方案 |
|---|---|
| 找不到 OpenFOAM 环境 | 确保已加载对应的 OpenFOAM bashrc。默认验证路径为 Foundation OpenFOAM v10（[openfoam.org](https://openfoam.org)）；ESI OpenFOAM 需要设置 `FOAMAGENT_OPENFOAM_FORK=esi` 并逐案例验证 |
| 数据库文件缺失 | 确保克隆了完整仓库，包括 `database/` 目录。Docker 镜像已预构建这些文件 |
| 缺少依赖项 | `conda env update -n FoamAgent -f environment.yml --prune` |
| API 密钥错误 | 确保设置了正确的密钥（`OPENAI_API_KEY`、`ANTHROPIC_API_KEY` 等） |
| MCP 连接错误 | 验证容器正在运行且 7860 端口可访问 |

> **OpenFOAM 版本：** Foam-Agent 默认面向 **Foundation OpenFOAM v10**（[openfoam.org](https://openfoam.org)）。设置 `FOAMAGENT_OPENFOAM_FORK=esi` 后，生成的文件将尽力转换为 ESI OpenFOAM（[openfoam.com](https://openfoam.com)，例如 v2312、v2406、v2512）规范。Docker 镜像预装了 Foundation OpenFOAM v10。

## 社区

### 加入微信社区

中文用户可以通过添加志愿者的微信号 **ZDSJTUCFD** 加入 Foam-Agent 微信社区。志愿者会邀请您进群。

## 引用

如果您在研究中使用了 Foam-Agent，请引用我们的论文：

```bibtex
@article{yue2025foam,
  title={Foam-Agent: Towards Automated Intelligent CFD Workflows},
  author={Yue, Ling and Somasekharan, Nithin and Zhang, Tingwen and Cao, Yadi and Chen, Zhangze and Di, Shimin and Pan, Shaowu},
  journal={arXiv preprint arXiv:2505.04997},
  year={2025}
}

@article{somasekharan2026cfdllmbench,
    title={CFDLLMBench: A Benchmark Suite for Evaluating Large Language Models in Computational Fluid Dynamics},
    author={Somasekharan, Nithin and Yue, Ling and Cao, Yadi and Li, Weichao and Emami, Patrick and Bhargav, Pochinapeddi Sai and Acharya, Anurag and Xie, Xingyu and Pan, Shaowu},
    journal={Journal of Data-centric Machine Learning Research},
    year={2026},
    url={https://openreview.net/forum?id=kTcH1MnkjY},
    note={}
}
```

## Star 历史

[![Star History Chart](https://api.star-history.com/svg?repos=csml-rpi/Foam-Agent&type=timeline&legend=top-left)](https://www.star-history.com/#csml-rpi/Foam-Agent&type=timeline&legend=top-left)
