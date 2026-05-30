# Claude Sonnet Implementation Prompt - Jenkins Pipeline System

你好！我想要開發一個專門為 Claude Code 設計的專案架構與 CI/CD 工作流生成系統。
我已經編寫並審查完畢一份完整的 `SPEC.md`（規格書），現在需要你作為資深的 Anthropic 系統架構師，幫我實作這套系統核心代碼。

### 1. 系統目標
本專案透過開發兩個自訂的 Claude Skills，來動態分析專案、建立 SDLC 工作流（Workflows）、自訂子代理（Subagents）與執行腳本。

### 2. 專案目錄結構
請確保所有產出的代碼與配置檔案嚴格遵循以下路徑規範：
{專案根目錄}/
├── .claude/
│   ├── project-plan.md                # 專案規劃書
│   ├── agents/                        # 專案級自訂子代理 md 檔案
│   │   ├── plan-agent.md
│   │   ├── build-agent.md
│   │   ├── test-agent.md
│   │   └── release-agent.md
│   ├── workflows/                     # 實際運行的背景 Workflow JS
│   │   └── {flow}.js
│   └── pipelines/                     # Pipeline 配置與實體執行腳本
│       └── {flow}/
│           ├── pipeline.yaml          # Git-committed pipeline 流程定義
│           ├── pipeline_config.local.yaml  # Git-ignored 機敏設定與本機路徑
│           └── scripts/
│               └── *.py               # Python 實體執行器

### 3. 需要你實作的兩大核心 Claude Skills

請為我編寫這兩個 Skill 檔案的詳細系統提示詞（Markdown Instructions），這兩個 Skill 會安裝在使用者本機的 `~/.claude/skills/` 目錄中：

#### A. `project-architect` Skill
- **職責**：分析 repo 中的程式語言、框架與部署方式，產生 `.claude/project-plan.md`，並動態生成 4 個子代理配置檔案（`.claude/agents/{agent-name}.md`）。
- **關鍵路由規則**：產出的 4 個子代理 frontmatter 中的 `description` 必須精確且互斥，避免路由衝突。請使用以下推薦描述：
  - `plan-agent`: "Creates architecture documents, requirements specifications, threat models, and software design workflows."
  - `build-agent`: "Defines project build, compilation, packaging, artifact generation, and dependency tree workflows."
  - `test-agent`: "Defines unit test, integration test, SAST, DAST, SCA, secret scanning, and code review workflows."
  - `release-agent`: "Defines release, deployment, smoke test, rollout, rollback, and delivery verification workflows."

#### B. `pipeline-architect` Skill
- **職責**：依據使用者命令或補充說明，動態產生新的 Workflow JS 檔案、YAML 設定與 Python 腳本，並支援 `.draft/` 暫存目錄推播至正式目錄的確認機制。

---

### 4. 請特別遵循以下進階安全與健壯性規範（極重要）：

#### 規範一：機敏資料去敏感驗證 (Config Validation)
Workflow JS 不能直接處理明文金鑰。請為生成的 Workflow 設計第一個 Stage 為 **Config Validation**。
- 呼叫 `agent()` 讀取本機的 `pipeline_config.local.yaml`，確認必填欄位是否存在。
- **安全性限制**：驗證 agent 絕不能將機敏內容（如 API Key、密碼、Token）以變數形式傳回給 Workflow JS。Workflow JS 只能接收去敏感的狀態（如 `{ scanner_path: "present", has_token: true }`）來做流程分支判斷，後續實體 Python 腳本再自行去讀取該本地 yaml。

#### 規範二：結構化錯誤處理 (Failure Handling)
背景運行的 Workflow JS 必須具備錯誤阻斷與回報機制。請使用以下結構進行 `try-catch` 控制：
- 每個 stage 的 `agent()` 呼叫需返回包含 `success`, `exitCode`, `stdoutSummary`, `stderrSummary` 的結果。
- 發生失敗時，Workflow 必須停止後續 dependent stages。
- 呼叫 reporter agent 產生不含機敏金鑰的報告 `.claude/pipelines/{flow}/reports/error-report.md`，內容需包含：錯誤階段、原因、本機日誌路徑、建議修復方式以及**重新執行的命令建議**。

---

### 5. 你的任務
請先幫我規劃這兩個 Skill（`project-architect` 與 `pipeline-architect`）的具體 Markdown 指令內容（System Prompt），並為我提供一個符合上述「驗證」與「錯誤處理」規範的 `sast-scan.js` 工作流 JS 實作範例。
