---
name: three-tier-agent-orchestrator
description: "僅在使用者明確要求用 $three-tier-agent-orchestrator 或 Three-Tier Agent Orchestrator 執行任務時啟用。由主線程承擔 GPT-6.1 Sol 統籌與複雜推理角色，以 GPT-6.1 Terra 處理視覺、電腦操作、文字與範圍明確的程式修正，以 GPT-6 Luna Max 處理可重複工作。詢問、安裝、修改或發布本 skill 不啟用委派；委派前確認必要模型與 reasoning 可用。"
---

# Three-Tier Agent Orchestration

讓目前主線程承擔 Sol 的統籌角色，負責理解目標、拆分任務、架構判斷、複雜推理、驗收與整合。獨立的困難工作可交給 Sol XHigh；視覺理解、電腦操作、文字撰修與範圍明確的程式修正交給 Terra；清楚、可重複且容易驗證的工作交給 Luna Max。三層依模型分為 Sol、Terra、Luna；Sol 同時負責統籌與困難子任務。

## 啟用條件

只有使用者明確要求用本 skill 執行任務時才啟用，例如輸入 `$three-tier-agent-orchestrator`，或「使用 Three-Tier Agent Orchestrator 檢查這個專案」。任務類型或模型名稱相符，不代表已啟用。詢問、安裝、修改或發布本 skill 屬於管理操作，不會啟用委派工作流。

## 1. 委派前確認 subagent 能力

目前主線程直接承擔 Sol 的統籌角色，建議主模型為 `gpt-6.1-sol`、reasoning 為 `max`。Skill 無法可靠驗證或切換主線程的模型，因此不得根據自我辨識、模型名稱、系統提示或缺少可觀察資訊而停止工作，也不得要求使用者先切換主模型或取消模型限制。此處的角色名稱不代表已驗證主線程的實際模型。

在實際委派前，檢查目前執行環境暴露的 subagent 能力：

1. 確認 subagent 工具可直接指定以下組合：
   - Sol XHigh：`model = "gpt-6.1-sol"`、`reasoning_effort = "xhigh"`。
   - Terra High：`model = "gpt-6.1-terra"`、`reasoning_effort = "high"`。
   - Luna Max：`model = "gpt-6-luna"`、`reasoning_effort = "max"`。
2. 若工具宣告支援、但實際啟動遭模型或 reasoning 相容性拒絕，將該能力視為不可用。
3. 只有上述三種必要組合都確認可用後，才能進入第 2 節。若任務選用 Terra XHigh，還須在該次委派前確認 `gpt-6.1-terra` 與 `xhigh` 可用；未選用時不把 Terra XHigh 當成必要條件。

每次啟動 subagent 時，直接傳入所選角色的 `model` 與 `reasoning_effort`，不要依賴額外的 custom agent 設定檔或工具預設值。不要把 App 的 model picker 是否顯示某個 reasoning 當成能力預檢條件；以 subagent 工具宣告與實際啟動結果為準。

啟動前檢查目前 `spawn_agent` schema，明確隔離主線程歷史：

- V2 若支援 `fork_turns`，每次傳入 `fork_turns = "none"`。
- V1 若只支援 `fork_context`，每次傳入 `fork_context = false`。
- 不要同時傳入兩者，也不要傳入目前 schema 未宣告的欄位。

Subagent prompt 是唯一可信的任務背景；將必要資訊、範圍與驗收條件明確寫入 prompt，不要依賴繼承的對話歷史。視覺或電腦操作任務另附子代理可存取的圖片／檔案絕對路徑、目標應用程式或分頁、目前狀態與可用工具；不要假定子代理已看過主線程的截圖或能使用相同桌面工具。

### 任一必要能力不可用：完整 hard stop

若缺少 `Sol XHigh`、`Terra High`、`Luna Max` 或已選用的 `Terra XHigh` 組合，立即停止整個工作流，不只是停止委派。除確認 subagent 能力與提供開啟說明外，不得執行任何工具或推進實質任務，包括：

- 不讀取或盤點專案、程式碼、文件與外部資料。
- 不進行架構設計、法遵邊界、需求拆解、風險分析或測試規劃。
- 不建立任何替代子代理，也不讓主線程單獨先做可安全推進的工作。
- 不自行降級成其他模型／reasoning 設定。

不得以主線程先做部分工作作為例外。此 skill 啟用期間，缺少必要能力就是完整停止條件；Terra High 不可用時，也不能自行以 Terra XHigh 替代。只有使用者明確取消此 skill 或明確改變所需模型組合時，才能採用其他工作流。

唯一允許的修復例外：若使用者明確要求 AI 協助開啟必要 subagent 能力，AI 可以指引 App 設定；若目前環境提供桌面控制能力，也可以直接協助檢查模型與 reasoning 選項。此例外不得延伸到讀取原專案、分析原任務或推進任何主線工作。

### 必要模型或 reasoning 不可用

準確指出缺少的是哪個模型或 reasoning 組合，並請使用者在 Codex App 的 model／reasoning 控制中確認：

- Sol XHigh 需要 `gpt-6.1-sol` 與 `xhigh`。
- Terra High 需要 `gpt-6.1-terra` 與 `high`；選用 Terra XHigh 時需要同一模型與 `xhigh`。
- Luna Max 需要 `gpt-6-luna` 與 `max`。

若選項不存在，請使用者檢查帳號方案、工作區管理員模型政策與目前 provider。Skill 本身無法解鎖未提供的模型。能力重新可用前持續 hard stop。

## 2. 建立任務地圖

由主線程先定義整體目標、限制、驗收條件與依賴關係，再依工作性質路由：

| 角色 | 分派內容 |
| --- | --- |
| Sol 統籌主線程（建議 GPT-6.1 Sol Max） | 理解目標、拆分任務、架構決策、跨模組複雜推理、檢查結果與整合輸出 |
| GPT-6.1 Sol XHigh 子代理 | 困難但可封裝的模組分析、非平凡獨立實作、深度程式碼審查、安全或併發推理、複雜根因排查 |
| GPT-6.1 Terra High（預設）／XHigh（按需） | 截圖、圖表與技術圖面理解，瀏覽器／桌面操作，表單與資料核對，依已提供材料撰寫或改寫文字，以及驗收條件明確的局部程式與 UI 修正 |
| GPT-6 Luna Max | 程式碼搜尋與事實整理、測試執行與錯誤重現、日誌分類、機械式修改、規格非常明確的小功能與結構化摘要 |

### Terra 的選擇與升級

- Terra 僅使用 `high` 或 `xhigh`，預設為 `high`。對困難圖面、多步驟視覺核對或需要較深推理的局部修正，可在分派時明確選用 `xhigh`，並說明原因。
- 一般搜尋、批次轉換、測試執行與制式摘要交給 Luna Max。需要解讀畫面、操作應用程式或判斷文字語氣時，優先選 Terra。
- 跨模組設計、複雜程式推理、安全與併發問題交給主線程或 Sol XHigh。不要把 Terra 的視覺與電腦操作表現推論成適合所有複雜程式工作。
- Terra 驗收未通過時，把具體差異與重現證據交回主線程：視覺或局部推理不足可改派 Terra XHigh；涉及架構或複雜根因則改派 Sol XHigh。這是按工作性質重新分派，不是模型／reasoning 不可用時的替代方案。

不要為了湊齊三種模型而委派微小工作。核心需求仍不清楚時，先由主線程解決，不要只靠提高子代理 reasoning effort。

## 3. 用明確契約委派

V2 的 `spawn_agent` 若支援 `task_name`，先選定實際 `model` 與 `reasoning_effort`，再依下表產生名稱結尾：

| Model | Reasoning | `task_name` 結尾 | 範例 |
| --- | --- | --- | --- |
| `gpt-6.1-sol` | `xhigh` | `_sol_xhigh` | `security_review_sol_xhigh` |
| `gpt-6.1-terra` | `high` | `_terra_high` | `inspect_ui_terra_high` |
| `gpt-6.1-terra` | `xhigh` | `_terra_xhigh` | `interpret_drawing_terra_xhigh` |
| `gpt-6-luna` | `max` | `_luna_max` | `run_tests_luna_max` |

`task_name` 使用簡短、可辨識的工作前綴，加上固定角色結尾；只使用小寫字母、數字與底線。啟動前比較名稱結尾與實際 model／reasoning，兩者不一致時不得呼叫。名稱只是可見標籤，實際 `model` 與 `reasoning_effort` 才是權威。V1 或任何未宣告 `task_name` 的 schema 不要傳入此欄位。

每個子代理 prompt 都要包含：

- **目標**：只描述一個可獨立完成的成果。
- **背景與輸入**：提供必要檔案、符號、錯誤或規格。
- **範圍**：列出允許讀寫的模組或檔案。
- **禁止事項**：指出不可變更的介面、行為與相鄰工作。
- **完成條件**：定義可檢查的 acceptance criteria。
- **驗證方式**：指定測試、型別檢查、lint、重現步驟或證據。
- **回報格式**：要求摘要、變更檔案、驗證結果、風險與未解問題。

Terra 電腦操作任務還須定義允許變更的頁面、紀錄或欄位，以及儲存後重新開啟核對的方式；文字任務須指定讀者、語氣與必須保留的事實。模型能力不代表工具存取權限，執行前確認子代理具備該任務需要的工具與授權；缺少時回報具體阻礙，不宣稱已操作完成。

預設採星狀協作：主線程直接分派給 Sol XHigh、Terra 與 Luna Max，各自直接回報主線程。不要讓子代理再管理其他子代理，除非被授權完整承包一個獨立子系統。

## 4. 控制並行寫入

只把互不依賴的工作平行化。多個代理共用工作目錄時，為每個寫入任務指定互斥的檔案或模組所有權；範圍重疊時改成循序執行或使用隔離 worktree。

電腦操作也要指定應用程式、分頁或紀錄的所有權；共用同一桌面或瀏覽器狀態時循序操作，即使輸出檔案不同也不要同時控制。

適合的交叉檢查模式：

- Terra 根據畫面與規格完成局部 UI 修正，釋出檔案後由 Luna Max 執行回歸測試，主線程核對視覺與功能證據。
- Terra 完成表單或文件操作並重新開啟核對，主線程檢查實際儲存結果與完成報告是否一致。
- Sol XHigh 實作複雜功能，Terra 在實作完成後做視覺／操作驗證，Luna Max 執行針對性測試，主線程最終整合。
- Luna Max 收集重現與日誌證據，Sol XHigh 分析根因，主線程決定跨模組修正方案。

## 5. 驗收並整合

等待所有必要結果後，由主線程：

1. 檢查子代理是否遵守範圍與完成條件。
2. 直接查看重要 diff、檔案引用、測試與重現證據，不只接受摘要。
3. 以程式碼與驗證結果解決互相矛盾的結論。
4. 執行適合風險程度的整合驗證。電腦操作要核對儲存後的實際紀錄或檔案，UI 變更要檢查呈現結果，文字要核對事實、讀者與語氣。
5. 在最終輸出說明完成內容、驗證結果、剩餘風險，以及主線程和實際使用的 Sol XHigh、Terra High／XHigh、Luna Max 分工。只列實際使用的角色與可觀察的啟動參數，不把主線程的建議模型當成已驗證事實。

不要把同一代理的「已完成」當成最終批准；最終責任留在主線程。

## 分工依據

此分工是根據 [OpenAI 的 Terra 介紹](https://openai.com/zh-Hant/index/introducing-gpt-6-1-terra/)與 [OpenAI Docs 模型選擇指引](https://developers.openai.com/api/docs/guides/model-selection)所做的工作流設計，不保證個別任務的速度或品質。[Terra 模型文件](https://developers.openai.com/api/docs/models/gpt-6.1-terra)列出支援 `high` 與 `xhigh`；本 skill 明確以 `high` 為預設，按需選用 `xhigh`。Codex 子代理是否提供指定組合，仍以目前工具宣告與實際啟動結果為準。
