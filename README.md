# Three-Tier Agent Orchestrator

> 本 README 以中文為主，英文版請見文末。

一個 Codex skill，讓目前主線程承擔 GPT-6 Astra 的總指揮角色：理解目標、拆分任務、做架構決策、檢查結果並整合輸出；並依工作性質把獨立子任務交給 GPT-5.6 Sol Medium 與 GPT-5.6 Luna Max。

## 角色分工

| 角色 | 責任 |
| --- | --- |
| GPT-6 Astra | 理解目標、拆分任務、架構決策、檢查結果與整合輸出 |
| GPT-5.6 Sol Medium | 困難但邊界清楚的分析、實作、深度 code review 與複雜 debugging |
| GPT-5.6 Luna Max | 清楚、可重複且容易驗證的搜尋、測試、重現、機械式修改與摘要 |

## 主要行為

- 目前主線程直接承擔 Astra 的統籌角色；skill 不自行判定、切換或要求切換主模型。
- 委派前確認 GPT-5.6 Sol Medium 與 GPT-5.6 Luna Max subagent 是否可用。
- 啟動 subagent 時隔離主線程歷史：V2 使用 `fork_turns: "none"`，V1 使用 `fork_context: false`。
- 任一必要模型或指定 reasoning 不可用時，完整停止工作，不以其他模型或較低 reasoning 替代。
- 能力齊全後，使用明確的目標、範圍、完成條件與驗證方式委派子任務。
- 由 GPT-6 Astra 檢查重要 diff、測試與證據，並負責最終整合。

## 必要條件

- Codex 支援 skills 與 subagents，且 subagent 工具可直接指定模型與 reasoning effort。
- 建議主線程選用 `gpt-6-astra`，但這不是 skill 的 hard-stop 條件。
- Sol Medium subagent 使用 `gpt-5.6-sol` 與 `medium` reasoning effort。
- Luna Max subagent 使用 `gpt-5.6-luna` 與 `max` reasoning effort。

不需要建立額外的 custom agent 設定檔。Skill 每次啟動 subagent 時都會直接傳入指定的模型、reasoning effort 與目前 schema 支援的上下文隔離欄位；實際 subagent 啟動結果才是能力判準。

## 安裝

在 macOS 或 Linux 執行：

```bash
git clone https://github.com/irons163/three-tier-agent-orchestrator.git "${CODEX_HOME:-$HOME/.codex}/skills/three-tier-agent-orchestrator"
```

若目前 task 沒有重新載入 skill，請建立新 task；仍未出現時再重新啟動 Codex App。

## 使用

在 Codex prompt 明確啟用：

```text
$three-tier-agent-orchestrator
```

也可以直接描述工作，例如：

```text
使用 Three-Tier Agent Orchestrator 檢查這個專案：GPT-6 Astra 負責理解目標與整合，GPT-5.6 Sol Medium 做架構與安全審查，GPT-5.6 Luna Max 執行測試與整理錯誤。
```

## Repository 結構

```text
.
├── README.md
├── SKILL.md
└── agents/
    └── openai.yaml
```

不需要額外的 custom agent 設定檔。

---

## English Version

### Three-Tier Agent Orchestrator

A Codex skill that assigns the current main thread the GPT-6 Astra orchestration role: understanding the objective, breaking down tasks, making architectural decisions, reviewing results, and integrating the final output while delegating independent subtasks to GPT-5.6 Sol Medium and GPT-5.6 Luna Max based on the nature of the work.

### Role assignments

| Role | Responsibilities |
| --- | --- |
| GPT-6 Astra | Understand the objective, break down tasks, make architectural decisions, review results, and integrate the final output |
| GPT-5.6 Sol Medium | Handle difficult but clearly bounded analysis and implementation, in-depth code review, and complex debugging |
| GPT-5.6 Luna Max | Perform clear, repeatable, and easily verifiable searches, tests, reproductions, mechanical edits, and summarization |

### Core behavior

- The current main thread assumes the Astra orchestration role; the skill does not detect, switch, or require switching the main model.
- Verify that the GPT-5.6 Sol Medium and GPT-5.6 Luna Max subagents are available before delegation.
- Isolate main-thread history when spawning a subagent: use `fork_turns: "none"` on V2 and `fork_context: false` on V1.
- If any required model or specified reasoning capability is unavailable, stop completely instead of substituting another model or using a lower reasoning level.
- Once all capabilities are available, delegate subtasks with explicit objectives, scope, completion criteria, and validation methods.
- GPT-6 Astra reviews important diffs, tests, and evidence, and is responsible for the final integration.

### Requirements

- Codex supports skills and subagents, and the subagent tool can specify the model and reasoning effort directly.
- Using `gpt-6-astra` for the main thread is recommended but is not a hard-stop condition.
- The Sol Medium subagent uses `gpt-5.6-sol` with `medium` reasoning effort.
- The Luna Max subagent uses `gpt-5.6-luna` with `max` reasoning effort.

No additional custom agent configuration files are required. Whenever the skill launches a subagent, it directly passes the specified model, reasoning effort, and the context-isolation field supported by the current schema. The actual subagent startup result determines whether the capability is available.

### Installation

Run the following on macOS or Linux:

```bash
git clone https://github.com/irons163/three-tier-agent-orchestrator.git "${CODEX_HOME:-$HOME/.codex}/skills/three-tier-agent-orchestrator"
```

If the current task does not reload the skill, create a new task. If it still does not appear, restart the Codex App.

### Usage

Explicitly enable the skill in a Codex prompt:

```text
$three-tier-agent-orchestrator
```

You can also describe the work directly, for example:

```text
Use the Three-Tier Agent Orchestrator to review this project: GPT-6 Astra handles objective understanding and integration, GPT-5.6 Sol Medium performs the architecture and security review, and GPT-5.6 Luna Max runs tests and organizes the errors.
```

### Repository structure

```text
.
├── README.md
├── SKILL.md
└── agents/
    └── openai.yaml
```

No additional custom agent configuration files are required.
