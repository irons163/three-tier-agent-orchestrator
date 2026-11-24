# Three-Tier Agent Orchestrator

> 本 README 以中文為主，英文版請見文末。

English version skill is [https://github.com/irons163/three-tier-agent-symphony](https://github.com/irons163/three-tier-agent-symphony).

一個 Codex skill，採用 **Sol 統籌與複雜推理、Terra 視覺與操作、Luna 可重複工作**的三層分工。目前主線程負責理解目標、拆分任務、架構決策、驗收與整合，並按工作性質分派獨立子任務。

## 角色分工

| 角色 | 責任 |
| --- | --- |
| Sol 統籌主線程（建議 GPT-6.1 Sol Max） | 理解目標、拆分任務、架構決策、跨模組複雜推理、檢查結果與整合輸出 |
| GPT-6.1 Sol XHigh 子代理 | 困難但邊界清楚的分析、實作、深度 code review 與複雜 debugging |
| GPT-6.1 Terra High／XHigh | 視覺理解、瀏覽器與桌面操作、表單與資料核對、文字撰修，以及範圍與驗收條件明確的程式／UI 修正 |
| GPT-6 Luna Max | 清楚、可重複且容易驗證的搜尋、測試、重現、機械式修改與摘要 |

三層依模型分為 Sol、Terra、Luna；Sol 主線程與 Sol XHigh 子代理屬於同一層。

## 為何這樣安排 Terra

根據 [OpenAI 官方介紹](https://openai.com/zh-Hant/index/introducing-gpt-6-1-terra/)與 [OpenAI Docs 模型選擇指引](https://developers.openai.com/api/docs/guides/model-selection)，Terra 適合視覺、電腦操作、文字與局部程式工作，最複雜的程式工作仍適合 Sol。因此本 skill 採以下路由；這是工作流設計，不是每項任務的效能保證。

- **Terra High 為預設**：處理畫面與圖表解讀、操作應用程式、根據已提供材料撰寫或改寫文字，以及局部修正，保留速度與成本優勢。
- **Terra XHigh 按需使用**：困難圖面、多步驟視覺核對或需要較深推理的局部修正，分派時明確選用並說明原因；Terra 僅使用 High 或 XHigh。
- **複雜程式工作交給 Sol XHigh**：跨模組設計、安全、併發或複雜根因分析。Terra 驗收失敗時，主線程根據具體證據決定是否改派 Terra XHigh 或 Sol XHigh。
- **制式工作交給 Luna Max**：搜尋、批次處理、測試執行與機械式修改；需要視覺、操作或文字語氣判斷時選 Terra。

例如，Terra 根據畫面修正局部 UI，完成後由 Luna 執行回歸測試，主線程驗收；若問題涉及跨模組狀態或競態，交由 Sol 分析。

## 主要行為

- 僅在使用者輸入 `$three-tier-agent-orchestrator` 或明確點名要求使用此 skill 執行任務時啟用。
- 目前主線程直接承擔 Sol 的統籌角色；skill 不自行判定、切換或要求切換主模型。
- 委派前確認 GPT-6.1 Sol XHigh、GPT-6.1 Terra High 與 GPT-6 Luna Max subagent 是否可用；選用 Terra XHigh 時另行確認該組合。
- 啟動 subagent 時隔離主線程歷史：V2 使用 `fork_turns: "none"`，V1 使用 `fork_context: false`。
- V2 的 subagent 名稱依實際參數使用固定結尾：`_sol_xhigh`、`_terra_high`、`_terra_xhigh` 或 `_luna_max`。
- 任一必要模型或指定 reasoning 不可用時，完整停止工作，不以其他模型或較低 reasoning 替代。
- 能力齊全後，使用明確的目標、範圍、完成條件與驗證方式委派子任務。
- 視覺任務提供子代理可存取的圖片／檔案絕對路徑；電腦操作確認工具與授權，並指定應用程式、分頁或紀錄的所有權，共用桌面狀態時循序操作。
- 由主線程檢查重要 diff、測試與證據；電腦操作須核對儲存後的實際結果，文字須核對事實、讀者與語氣，再完成整合。

## 必要條件

- Codex 支援 skills 與 subagents，且 subagent 工具可直接指定模型與 reasoning effort。
- 建議主線程選用 `gpt-6.1-sol` 與 `max`，但這不是 skill 的 hard-stop 條件，也不代表 skill 已驗證主模型。
- Sol XHigh subagent 使用 `gpt-6.1-sol` 與 `xhigh` reasoning effort。
- Terra subagent 預設使用 `gpt-6.1-terra` 與 `high`；按需選用 `xhigh`，須另外確認可用。
- Luna Max subagent 使用 `gpt-6-luna` 與 `max` reasoning effort。

Sol XHigh、Terra High、Luna Max 是必要的 subagent 組合，任何一項不可用都完整停止工作流；未選用 Terra XHigh 時，不要求該組合可用。模型／reasoning 不可用時不能自行換成其他組合。驗收失敗後按工作性質重新分派，與能力缺失時的替代不同。

[Terra 模型文件](https://developers.openai.com/api/docs/models/gpt-6.1-terra)列出支援 `high` 與 `xhigh`；本 skill 明確以 `high` 為預設，按需選用 `xhigh`。Codex 子代理是否提供這些組合，仍以目前工具宣告與實際啟動結果為準。

不需要建立額外的 custom agent 設定檔。Skill 每次啟動 subagent 時都會直接傳入指定的模型、reasoning effort 與目前 schema 支援的上下文隔離欄位；實際 subagent 啟動結果才是能力判準。

V2 若支援 `task_name`，名稱會使用可辨識的工作前綴與固定角色結尾，例如 `security_review_sol_xhigh`、`inspect_ui_terra_high`、`interpret_drawing_terra_xhigh` 或 `run_tests_luna_max`。Skill 會在啟動前確認名稱結尾與實際 model／reasoning 一致；V1 不傳入此欄位。

## 安裝

在 macOS 或 Linux 執行：

```bash
git clone https://github.com/irons163/three-tier-agent-orchestrator.git "${CODEX_HOME:-$HOME/.codex}/skills/three-tier-agent-orchestrator"
```

若目前 task 沒有重新載入 skill，請建立新 task；仍未出現時再重新啟動 Codex App。

## 使用

此 skill 需要使用者主動啟用。在 Codex prompt 輸入：

```text
$three-tier-agent-orchestrator
```

也可以明確點名要求使用此 skill，例如：

```text
使用 Three-Tier Agent Orchestrator 檢查這個專案：主線程負責統籌與整合，GPT-6.1 Sol XHigh 做架構與安全審查，GPT-6.1 Terra High 檢查 UI 畫面與操作，GPT-6 Luna Max 執行測試與整理錯誤。
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

A Codex skill with three tiers: **Sol for orchestration and complex reasoning, Terra for visual and computer-use work, and Luna for repeatable work**. The current main thread owns the objective, task breakdown, architecture, acceptance, and integration, delegating independent subtasks according to their needs.

### Role assignments

| Role | Responsibilities |
| --- | --- |
| Sol orchestration main thread (GPT-6.1 Sol Max recommended) | Understand the objective, break down tasks, make architectural decisions, handle complex cross-module reasoning, review results, and integrate the final output |
| GPT-6.1 Sol XHigh subagent | Handle difficult but clearly bounded analysis and implementation, in-depth code review, and complex debugging |
| GPT-6.1 Terra High / XHigh | Interpret visuals, operate browsers and desktop apps, check forms and data, draft and revise text, and make scoped code or UI fixes with clear acceptance criteria |
| GPT-6 Luna Max | Perform clear, repeatable, and easily verifiable searches, tests, reproductions, mechanical edits, and summarization |

The three tiers are the Sol, Terra, and Luna models. The main thread and Sol XHigh subagents share the Sol tier.

### Why Terra has this role

The [official Terra announcement](https://openai.com/index/introducing-gpt-6-1-terra/) and [OpenAI Docs model-selection guidance](https://developers.openai.com/api/docs/guides/model-selection) support using Terra for visual, computer-use, writing, and scoped coding tasks, while reserving the most complex coding for Sol. The routing below is this skill's workflow design, not a performance guarantee.

- **Terra High by default:** use it for interpreting screens and charts, operating apps, writing from supplied material, and scoped fixes, preserving its speed and cost advantages.
- **Terra XHigh when justified:** select it explicitly for difficult drawings, multistep visual checks, or scoped fixes needing deeper reasoning. State why; use only High or XHigh for Terra.
- **Sol XHigh for complex coding:** route cross-module design, security, concurrency, and difficult root-cause analysis to Sol. When Terra fails acceptance, the main thread uses concrete evidence to choose Terra XHigh or Sol XHigh for the follow-up.
- **Luna Max for routine work:** use it for search, batch processing, test execution, and mechanical edits. Choose Terra when the task requires visual interpretation, app interaction, or judgment about writing tone.

For example, Terra fixes a local UI issue from screenshots, Luna runs regression tests after the edit, and the main thread checks the result. Cross-module state or race conditions go to Sol for analysis.

### Core behavior

- Activate only when the user invokes `$three-tier-agent-orchestrator` or explicitly asks to use this skill by name for a task.
- The current main thread assumes the Sol orchestration role; the skill does not detect, switch, or require switching the main model.
- Verify that GPT-6.1 Sol XHigh, GPT-6.1 Terra High, and GPT-6 Luna Max subagents are available before delegation. Also verify Terra XHigh if selected.
- Isolate main-thread history when spawning a subagent: use `fork_turns: "none"` on V2 and `fork_context: false` on V1.
- On V2, derive each subagent name from its actual launch arguments, using `_sol_xhigh`, `_terra_high`, `_terra_xhigh`, or `_luna_max`.
- If any required model or specified reasoning capability is unavailable, stop completely instead of substituting another model or using a lower reasoning level.
- Once all capabilities are available, delegate subtasks with explicit objectives, scope, completion criteria, and validation methods.
- Supply accessible absolute image/file paths for visual tasks. Confirm tools and authorization for computer use, assign ownership of apps, tabs, or records, and serialize operations that share desktop state.
- The main thread reviews important diffs, tests, and evidence. Verify saved results for computer use and check facts, audience, and tone for writing before integration.

### Requirements

- Codex supports skills and subagents, and the subagent tool can specify the model and reasoning effort directly.
- Using `gpt-6.1-sol` with `max` for the main thread is recommended but is not a hard-stop condition or a claim that the skill has verified the main model.
- The Sol XHigh subagent uses `gpt-6.1-sol` with `xhigh` reasoning effort.
- Terra subagents use `gpt-6.1-terra` with `high` by default. Select `xhigh` when needed and verify that combination separately.
- The Luna Max subagent uses `gpt-6-luna` with `max` reasoning effort.

Sol XHigh, Terra High, and Luna Max are required subagent combinations: if any is unavailable, stop the entire workflow. Terra XHigh is required only when selected. Never substitute a different combination for an unavailable model or reasoning level. Reassigning work after failed acceptance is a separate routing decision.

The [Terra model documentation](https://developers.openai.com/api/docs/models/gpt-6.1-terra) lists support for `high` and `xhigh`; this skill explicitly defaults to `high` and selects `xhigh` when needed. Availability through Codex subagent tools still depends on the current tool declaration and actual startup results.

No additional custom agent configuration files are required. Whenever the skill launches a subagent, it directly passes the specified model, reasoning effort, and the context-isolation field supported by the current schema. The actual subagent startup result determines whether the capability is available.

When V2 supports `task_name`, the name uses a recognizable task prefix and a fixed role suffix, such as `security_review_sol_xhigh`, `inspect_ui_terra_high`, `interpret_drawing_terra_xhigh`, or `run_tests_luna_max`. Before launch, the skill verifies that the suffix matches the actual model and reasoning arguments. V1 does not receive this field.

### Installation

Run the following on macOS or Linux:

```bash
git clone https://github.com/irons163/three-tier-agent-orchestrator.git "${CODEX_HOME:-$HOME/.codex}/skills/three-tier-agent-orchestrator"
```

If the current task does not reload the skill, create a new task. If it still does not appear, restart the Codex App.

### Usage

This skill requires explicit invocation. Enter the following in a Codex prompt:

```text
$three-tier-agent-orchestrator
```

You can also explicitly ask to use this skill by name, for example:

```text
Use the Three-Tier Agent Orchestrator to review this project: the main thread coordinates and integrates, GPT-6.1 Sol XHigh performs the architecture and security review, GPT-6.1 Terra High checks UI screens and interactions, and GPT-6 Luna Max runs tests and organizes the errors.
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
