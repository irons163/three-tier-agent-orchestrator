---
name: three-tier-agent-orchestrator
description: "讓主線程承擔 GPT-6 Astra 的總指揮角色，將困難但邊界清楚的工作交給 GPT-5.6 Sol Medium，將清楚、可重複的工作交給 GPT-5.6 Luna Max。當使用者要求 Astra 統籌、Sol Medium 或 Luna Max subagent、分層 multi-agent coding、平行 code review、模組分析、獨立功能實作、測試、debugging 或整合結果時使用；不要自行判定或要求切換主模型，委派前只檢查必要 subagent 的模型與 reasoning 是否可用。"
---

# Three-Tier Agent Orchestration

讓目前主線程承擔 Astra 的統籌角色，負責理解目標、拆分任務、架構判斷、驗收與整合。把困難但邊界明確的工作交給 Sol Medium，把清楚、可重複且容易驗證的工作交給 Luna Max。

## 1. 委派前確認 subagent 能力

目前主線程直接承擔 Astra 的統籌角色。Skill 無法可靠驗證或切換主線程的模型，因此不得根據自我辨識、模型名稱、系統提示或缺少可觀察資訊而停止工作，也不得要求使用者先切換主模型或取消模型限制。

在實際委派前，檢查目前執行環境暴露的 subagent 能力：

1. 確認 subagent 工具可直接指定以下組合：
   - Sol Medium：`model = "gpt-5.6-sol"`、`reasoning_effort = "medium"`。
   - Luna Max：`model = "gpt-5.6-luna"`、`reasoning_effort = "max"`。
2. 若工具宣告支援、但實際啟動遭模型或 reasoning 相容性拒絕，將該能力視為不可用。
3. 只有 Sol Medium 與 Luna Max 都確認可用後，才能進入第 2 節。

每次啟動 Sol Medium 或 Luna Max 時，直接傳入上述 `model` 與 `reasoning_effort`，不要依賴額外的 custom agent 設定檔。不要把 App 的 model picker 是否顯示 Max 當成能力預檢條件；以 subagent 工具宣告與實際啟動結果為準。

啟動前檢查目前 `spawn_agent` schema，明確隔離主線程歷史：

- V2 若支援 `fork_turns`，每次傳入 `fork_turns = "none"`。
- V1 若只支援 `fork_context`，每次傳入 `fork_context = false`。
- 不要同時傳入兩者，也不要傳入目前 schema 未宣告的欄位。

Subagent prompt 是唯一可信的任務背景；將必要資訊、範圍與驗收條件明確寫入 prompt，不要依賴繼承的對話歷史。

### 任一必要能力不可用：完整 hard stop

若缺少 `Sol Medium`、`Luna Max` 或其中任一必要模型／reasoning 組合，立即停止整個工作流，不只是停止委派。除確認 subagent 能力與提供開啟說明外，不得執行任何工具或推進實質任務，包括：

- 不讀取或盤點專案、程式碼、文件與外部資料。
- 不進行架構設計、法遵邊界、需求拆解、風險分析或測試規劃。
- 不建立任何替代子代理，也不讓主線程單獨先做可安全推進的工作。
- 不自行降級成其他模型／reasoning 設定。

不得使用「這不妨礙 Astra 主線程先做……」之類的例外。此 skill 啟用期間，缺少必要能力就是完整停止條件。只有使用者明確取消此 skill 或明確改變所需模型組合時，才能採用其他工作流。

唯一允許的修復例外：若使用者明確要求 AI 協助開啟必要 subagent 能力，AI 可以指引 App 設定；若目前環境提供桌面控制能力，也可以直接協助檢查模型與 reasoning 選項。此例外不得延伸到讀取原專案、分析原任務或推進任何主線工作。

### 必要模型或 reasoning 不可用

準確指出缺少的是哪個模型或 reasoning 組合，並請使用者在 Codex App 的 model／reasoning 控制中確認：

- Sol Medium 需要 `gpt-5.6-sol` 與 `medium`。
- Luna Max 需要 `gpt-5.6-luna` 與 `max`。

若選項不存在，請使用者檢查帳號方案、工作區管理員模型政策與目前 provider。Skill 本身無法解鎖未提供的模型。能力重新可用前持續 hard stop。

## 2. 建立任務地圖

由 Astra 先定義整體目標、限制、驗收條件與依賴關係，再依工作性質路由：

| 角色 | 分派內容 |
| --- | --- |
| GPT-6 Astra 主線程 | 理解目標、拆分任務、架構決策、檢查結果與整合輸出 |
| GPT-5.6 Sol Medium | 困難但可封裝的模組分析、非平凡獨立實作、深度程式碼審查、安全或併發推理、複雜根因排查 |
| GPT-5.6 Luna Max | 程式碼搜尋與事實整理、測試執行與錯誤重現、日誌分類、機械式修改、規格非常明確的小功能與結構化摘要 |

不要為了使用子代理而委派微小工作。核心需求仍不清楚時，先由 Astra 解決，不要只靠提高子代理 reasoning effort。

## 3. 用明確契約委派

每個子代理 prompt 都要包含：

- **目標**：只描述一個可獨立完成的成果。
- **背景與輸入**：提供必要檔案、符號、錯誤或規格。
- **範圍**：列出允許讀寫的模組或檔案。
- **禁止事項**：指出不可變更的介面、行為與相鄰工作。
- **完成條件**：定義可檢查的 acceptance criteria。
- **驗證方式**：指定測試、型別檢查、lint、重現步驟或證據。
- **回報格式**：要求摘要、變更檔案、驗證結果、風險與未解問題。

預設採星狀協作：Astra 直接分派給 Sol Medium 與 Luna Max，兩者直接回報 Astra。不要讓 Sol Medium 再管理 Luna Max，除非 Sol Medium 被授權完整承包一個獨立子系統。

## 4. 控制並行寫入

只把互不依賴的工作平行化。多個代理共用工作目錄時，為每個寫入任務指定互斥的檔案或模組所有權；範圍重疊時改成循序執行或使用隔離 worktree。

適合的交叉檢查模式：

- Sol Medium 實作複雜功能，Luna Max 建立或執行針對性測試，Astra 最終整合。
- Luna Max 完成明確修改，Sol Medium 做深度審查，Astra 判定是否接受。
- Luna Max 收集重現與日誌證據，Sol Medium 分析根因，Astra 決定跨模組修正方案。

## 5. 驗收並整合

等待所有必要結果後，由 Astra：

1. 檢查子代理是否遵守範圍與完成條件。
2. 直接查看重要 diff、檔案引用、測試與重現證據，不只接受摘要。
3. 以程式碼與驗證結果解決互相矛盾的結論。
4. 執行適合風險程度的整合驗證。
5. 在最終輸出說明完成內容、驗證結果、剩餘風險，以及實際使用的 Astra、Sol Medium 與 Luna Max 分工。

不要把同一代理的「已完成」當成最終批准；最終責任留在 Astra 主線程。
