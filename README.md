# opencode-skill-swarm-cluster

> opencode 技能：主 Agent 自主拆解任务，并行 spawn 2-4 个 subagent 分头工作，最后汇总。

## 是什么

给 opencode 加一条"多 subagent 并行集群"工作流规则：

- 主 Agent 判断任务是否值得开集群（≥2 独立子部分 / 需要多角度更好答案 / 用户显式说 "开启集群 / cluster it"）
- 一条 message 里同时发出 2-4 个 `task` 工具调用 = 天然并行
- 每个 worker 独立 session、独立 context、可指定不同 model
- 全部返回后主 Agent in-line 合成，或委托 `swarm-synth` 专门合成

零 npm 依赖，只用 opencode 原生 subagent + Task tool 能力。

## 快速安装

```bash
git clone --depth=1 https://github.com/Yulimfish/opencode-skill-swarm-cluster.git \
  ~/.config/opencode/skills/swarm-cluster
```

或用 [opencode-codex-kit](https://github.com/Yulimfish/opencode-codex-kit) 一键装齐：

```bash
curl -fsSL https://raw.githubusercontent.com/Yulimfish/opencode-codex-kit/main/install.sh | bash
```

## 配套 agent 定义

skill 只是规则文本。要真正让主 Agent 能 spawn 带独立 model 的 worker（`swarm-worker-kimi` 等），还需要装 [opencode-swarm-agents](https://github.com/Yulimfish/opencode-swarm-agents) 提供 5 个 worker 变体 + 1 个 synth。

## 触发词

- 显式：`开启集群` · `并行 agent` · `分头研究` · `多角度` · `spawn a swarm` · `cluster it` · `swarm on this`
- 自动：主 Agent 判断任务有 ≥2 个独立可并行子部分 / 需要多角度更好答案

## 内容

- **触发规则**：什么时候开、什么时候别开
- **并发策略**：起手 4 并行，遇 429 减半到 2，再串行
- **3 种拆解模式**：divide-by-slice / multi-angle / divide-by-region
- **worker prompt 模板**：persona / slice / out-of-scope / deliverable
- **合成规则**：consensus > majority > highest-confidence

## License

MIT © Yulimfish
