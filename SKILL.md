---
description: Use to spawn a parallel Agent cluster (2-4 subagents) when the task is complex enough to benefit from divide-and-conquer or from multi-angle reasoning. Main agent decomposes → spawns swarm workers IN PARALLEL via the Task tool → collects their final messages → synthesizes one consolidated answer. Trigger phrases include "开启集群", "并行子 agent", "分头研究", "多角度想", "spawn a swarm", "swarm on this", "cluster it". Also auto-load when the task has ≥3 non-trivial parallelizable sub-questions, or when the user asks for a high-quality answer requiring multiple perspectives.
---

# Agent Cluster (Swarm)

Use this skill to run a **short-lived parallel swarm of subagents** on ONE user task and merge their outputs.

## 0. When to activate — main agent decides autonomously

Activate a swarm when **any** of these is true:

- The task decomposes into ≥2 **independent** sub-parts (research + implement + review, or region A + region B + region C).
- The user asks for a **best-effort / high-quality** answer where multiple perspectives converge to a better result (design critique, architecture proposal, ambiguous debugging).
- The user explicitly says: "开启集群", "并行 agent", "分头研究", "多角度", "spawn a swarm", "swarm on this", "cluster it", "开个 swarm".

Do NOT activate for:

- Single-file surgical edits.
- Pure information lookups (one search is enough).
- Tasks where sub-parts have strong sequential dependencies (chain, not fan-out).

If unsure, do not activate. Under-using is cheaper than over-using.

## 1. Concurrency policy (hard)

- **Start with 4 workers in parallel.** Issue all 4 `task` tool calls in **one message**.
- If any provider returns rate-limit / 429 / concurrent-request errors, **halve** to 2 in the retry and note it. If 2 still fails, run sequentially.
- **Max concurrent = 4.** Never spawn more than 4 in one wave. If the task truly needs more slices, run wave 1 → collect → wave 2.
- **All spawned workers MUST return before you synthesize.** Do not partial-answer.
- **No nested swarms.** Workers must not call `task` themselves (their agent definitions forbid it).

## 2. Model routing

The main agent's model is the default. Only pick a specific `subagent_type` for a worker when the user asks or when the slice clearly benefits:

| subagent_type                | model                             | Best for                              |
|-------------------------------|-----------------------------------|---------------------------------------|
| `swarm-worker` (default)      | inherits main agent's model       | anything, uniform swarm               |

> Note: `swarm-worker` / `swarm-worker` / `swarm-worker` were
> removed — their agent files no longer exist. To route a worker to a specific
> model, create `~/.config/opencode/agents/swarm-worker-<name>.md` pinned to it.

Also usable when appropriate:
- `explore` — read-only codebase reconnaissance
- `general` — general-purpose multi-step
- `goal-verify` — independent verification of a completed claim

If the user says "让 X 模型做 Y 部分", map to the matching `swarm-worker-*` agent. Otherwise stay on `swarm-worker`.

## 3. Decomposition patterns (pick one)

### 3a · Divide-by-slice (task splits into disjoint pieces)

```
Task: "重构 auth 模块 + 加单测 + 审计安全"
  ├─ worker-A (swarm-worker):     refactor
  ├─ worker-B (swarm-worker):          write tests
  └─ worker-C (swarm-worker): security audit (read-only advisory)
```

### 3b · Multi-angle (same question, different perspectives, then reconcile)

```
Task: "评估用 SQLite 还是 Postgres 存这个数据"
  ├─ worker-A (swarm-worker): arguing FOR SQLite
  ├─ worker-B (swarm-worker): arguing FOR Postgres
  ├─ worker-C (swarm-worker):      neutral trade-off matrix
  └─ worker-D (swarm-worker):  edge-case / migration-path angle
```

### 3c · Divide-by-region (large surface, spatially split)

```
Task: "扫描仓库找所有硬编码 secret"
  ├─ worker-A: src/**/*.ts
  ├─ worker-B: config/** + .env*
  ├─ worker-C: scripts/**  + docs/**
  └─ worker-D: tests/**    + fixtures/**
```

Pick the pattern before spawning. Do not switch mid-run.

## 4. Prompt template for each worker

Every worker prompt MUST contain, in this order:

```
## Your persona
<short role: e.g. "You are the security-audit specialist in a 3-worker swarm.">

## Your slice
<exactly what to do, bounded — file paths, line ranges, questions to answer>

## Out of scope (do NOT touch)
<other workers' slices>

## Context the main agent already knows
<3-5 bullets summarizing state so the worker doesn't redo exploration>

## Deliverable
<exact shape you want in their final message — usually the standard worker shape>

## Return format
Follow your agent's default return shape (verdict / What I did / Result / Assumptions / Adjacent findings / Confidence). Start the FIRST LINE with your verdict.
```

## 5. Synthesis — how the main agent merges

After **all** workers return:

1. Read every worker's final message in full.
2. Pick synthesis venue:
   - **In-line synthesis** (default): main agent writes the merged answer directly. Faster, keeps context.
   - **Delegated synthesis** (only when ≥3 workers disagreed or output is long): spawn `swarm-synth` (read-only) with all workers' verbatim final messages + original task. Use its output.
3. Apply reconciliation rule when workers disagree: **consensus > majority > highest-confidence single source**. State which rule you used.
4. Merge into ONE answer to the user with this shape:

```
<one-line result>

## 集群结论
<the actual answer — code / analysis / recommendation>

## 分工回顾
- worker-A (<agent>): <one-line what they contributed>
- worker-B (<agent>): <one-line what they contributed>
- ...

## 分歧与处理
- <if any, otherwise omit>

## 置信度
<low | medium | high> — <why>
```

## 6. Failure handling

- **Worker returned `BLOCKED:`** → note the missing slice, decide if remaining slices are enough. If not, decompose differently and re-spawn ONLY the missing slice.
- **Worker returned garbage / off-scope** → discard, mention it in synthesis, do not respawn unless critical.
- **Any tool errors during spawn** (rate-limit, provider 429) → halve concurrency, retry once. Never retry 3+ times.
- **Timeout / stuck worker** → skip that worker, synthesize with what came back, flag the gap.

## 7. Cost & escalation guard

Each swarm run = up to 4× the tokens of a single agent turn. Rules:

- Do not swarm inside a swarm. One level deep, always.
- Do not swarm on trivial fan-outs where a single loop is faster.
- If the user says "别开集群 / 单线程做" — respect it, do not activate.
- If mid-task you realize decomposition was wrong, cancel remaining workers (mentally — don't wait) and restart with a better split, but only ONCE per user turn.

## 8. Example — one full run

User: "调研三种记忆插件哪个最适合 opencode，给推荐"

Main agent decision: multi-angle, 3 workers parallel.

```
task(swarm-worker,          "研究 opencode-mem 的能力与缺点 …")
task(swarm-worker, "研究 mem0 的能力与缺点 …")
task(swarm-worker,      "研究 letta 的能力与缺点 …")
```

All 3 return. Main agent picks in-line synthesis, writes:

```
推荐 opencode-mem。

## 集群结论
opencode-mem 因 X/Y/Z 胜出，mem0 弱在 A，letta 弱在 B。
...
## 分工回顾
- worker-A (swarm-worker): opencode-mem 特性 …
- worker-B (swarm-worker): mem0 …
- worker-C (swarm-worker): letta …
## 置信度
high — 三份独立调研在 X/Y 上达成一致
```
