# Jenkins Pipeline System — Specification Document

## 1. 系統架構

### 核心定位

本專案提供一組 Claude Code 擴充能力，用來分析專案、建立 SDLC pipeline workflows，並讓使用者後續補充或修訂 workflow。

系統由兩個 Claude skills、四個 Claude subagents，以及多個 Claude workflows 組成：

| 類型 | 名稱 | 定位 |
|------|------|------|
| Skill | `project-architect` | 主要入口。分析專案、產生 project plan，並建立 4 個領域 subagents |
| Skill | `pipeline-architect` | 人工補充/修訂工具。新增使用者想要的 workflow，或修改 agents 產生的 workflow |
| Subagent | `plan-agent` | 建立 Design / Planning 類 workflows |
| Subagent | `build-agent` | 建立 Build / Code Generation 類 workflows |
| Subagent | `test-agent` | 建立 Testing / Security / Review 類 workflows |
| Subagent | `release-agent` | 建立 Delivery / Release 類 workflows |
| Workflow | `{flow}.js` | 實際可執行的 Claude workflow，由 subagents 或 `pipeline-architect` 產生 |

### 整體資料流

```text
使用者
  │
  ▼
project-architect skill
  │
  ├─ 分析專案與需求
  ├─ 產生 .claude/project-plan.md
  └─ 產生 / 更新 4 個 Claude subagents
       │
       ├─ plan-agent    → Design workflows
       ├─ build-agent   → Build workflows
       ├─ test-agent    → Test / Security workflows
       └─ release-agent → Release workflows
              │
              ▼
        .claude/workflows/{flow}.js
        .claude/pipelines/{flow}/scripts/*.py
        .claude/pipelines/{flow}/pipeline.yaml
        .claude/pipelines/{flow}/pipeline_config.local.yaml

人工補充或修訂
  │
  ▼
pipeline-architect skill
  │
  ├─ 新增 workflow
  ├─ 修改既有 workflow
  └─ 管理 draft → confirm 流程
```

### Claude 元件分工

| 元件 | 負責內容 | 不負責內容 |
|------|----------|------------|
| Skill | 高階流程、規則、產生/更新檔案的指令策略 | 大量平行 orchestration |
| Subagent | 專門領域分析、產生 workflows/scripts/config skeleton | 全域產品決策 |
| Workflow | 可重跑的 orchestration script，串接 stages 與 agents | 中途問答、直接 shell/filesystem 操作 |
| Python script | deterministic executor，例如 CLI/API 呼叫 | 推理、規劃、改寫 workflow |

### Claude Workflow 限制

設計必須遵守 Claude dynamic workflows 的邊界：

- workflow script 本身負責 orchestration。
- workflow JS 只負責安排流程，不直接讀寫檔案或執行 shell。需要讀寫檔案、執行 Python/CLI/API、修改 `.gitignore` 時，workflow 應透過 `agent()` 指派給 agent 完成。
- workflow 一旦開始執行，就像背景任務，不應設計成跑到一半停下來問使用者問題。若某個決策需要使用者確認，應先產生 draft 與 summary 後結束；使用者再用下一次 `pipeline-architect` 指令提出修改或確認。
- workflow 適合大規模、多 agent、可重跑的工作；一般規則與檔案產生邏輯應放在 skill/subagent instructions。

---

## 2. `project-architect` Skill

`project-architect` 是主要入口 skill。它不是 Claude workflow，而是 Claude skill，負責引導 Claude 分析專案、建立 project plan，並產生四個領域 subagents。

### 主要職責

1. 分析目前專案型態、語言、部署方式、安全需求與既有 CI/CD 線索。
2. 產生 `.claude/project-plan.md`。
3. 產生或更新以下 Claude subagents：
   - `.claude/agents/plan-agent.md`
   - `.claude/agents/build-agent.md`
   - `.claude/agents/test-agent.md`
   - `.claude/agents/release-agent.md`
4. 指示四個 subagents 根據 plan 產生對應領域的 workflows。

### Project Plan

`.claude/project-plan.md` 是 commit-friendly 的專案規劃文件，包含：

- 專案名稱與類型
- 語言與主要框架
- build/test/deploy 方式
- 安全等級
- 安全需求；若使用者或既有專案未明確定義，預設以 NIST Secure Software Development Framework (SSDF) 作為基準
- 推薦 workflows
- 每個 workflow 的目的、領域、優先度與必要設定
- 哪些 workflow 需要使用者補充 credentials 或 local paths

### 規劃方式

`project-architect` 可以用對話協助使用者補足資訊，但不能把多輪問答設計在 workflow 裡。

建議流程：

1. 使用者要求建立 project architecture。
2. Claude 使用 `project-architect` skill 掃描 repo 並提出缺口。
3. Claude 產生 `.claude/project-plan.md` draft。
4. 若安全需求未定義，Claude 以 NIST SSDF 作為預設安全基準，推薦對應的 security workflows。
5. 使用者可直接修改 plan，或要求 Claude 修訂。
6. plan 穩定後，Claude 產生/更新四個 subagents。
7. 四個 subagents 依 plan 建立 workflows。

### 產生的 Subagents

每個 subagent 都是 Claude custom subagent，必須以 Markdown 檔案建立在專案層 `.claude/agents/` 目錄內。

```text
.claude/agents/{agent-name}.md
```

`project-architect` 產生或更新的 subagent 檔案固定為：

```text
.claude/agents/plan-agent.md
.claude/agents/build-agent.md
.claude/agents/test-agent.md
.claude/agents/release-agent.md
```

### Subagent 檔案結構

每個 subagent 檔案由兩部分組成：

| 區塊 | 格式 | 用途 |
|------|------|------|
| YAML frontmatter | `---` 包住的 metadata | 定義 name、description、tools、model 等 Claude Code subagent metadata |
| Markdown body | 一般 Markdown instructions | 定義此 agent 的角色、責任、輸入、輸出與限制 |

frontmatter 至少包含：

| 欄位 | 必填 | 說明 |
|------|------|------|
| `name` | Yes | subagent 名稱，需與檔名語意一致 |
| `description` | Yes | Claude Code 用來判斷何時委派給此 subagent 的描述 |
| `tools` | No | 此 subagent 可使用的工具；建議明確指定，若省略則繼承主 session 工具 |
| `model` | No | 指定模型，例如 `sonnet` |

### Subagent Description Routing Rules

Claude Code 會高度依賴 YAML frontmatter 的 `description` 判斷是否自動委派任務給 subagent。
因此四個 subagents 的 `description` 必須明確、互斥，避免使用過度通用的描述。

Rules:

- 每個 `description` 必須聚焦該 agent 的 workflow domain。
- 避免在多個 agents 中重複使用 generic phrases，例如 `analyze codebase`、`generate workflows`。
- `description` 應包含該 agent 負責的具體 workflow 類型。
- `description` 不應宣稱負責其他 agent 的 domain。

Recommended descriptions:

| Agent | Recommended `description` |
|-------|---------------------------|
| `plan-agent` | Creates architecture documents, requirements specifications, threat models, and software design workflows. |
| `build-agent` | Defines project build, compilation, packaging, artifact generation, and dependency tree workflows. |
| `test-agent` | Defines unit test, integration test, SAST, DAST, SCA, secret scanning, and code review workflows. |
| `release-agent` | Defines release, deployment, smoke test, rollout, rollback, and delivery verification workflows. |

### Subagent Capability Reuse Rules

Subagents 應在相關時優先重用既有 Claude skills、MCP servers、templates 與 reference knowledge base，而不是重新發明已存在的能力。

Rules:

- 優先使用既有 project/user/plugin skills，再考慮產生新的 workflow logic。
- 只有當 skill 與該 subagent domain 直接相關時，才透過 frontmatter `skills` 預載。
- 只有當 MCP server 是該 subagent 必要能力時，才透過 frontmatter `mcpServers` 指定。
- 不要預設給所有 subagents 廣泛 MCP/tool access；採最小權限。
- 若必要 MCP servers 或 skills 不存在，subagent 應回報 missing capability，並產生 safe fallback 或 draft。
- 不得將 secrets、tokens、MCP credentials 或 local-only settings 寫入 generated files。

建議：

| Agent | Capability reuse guidance |
|-------|---------------------------|
| `plan-agent` | 優先使用 project docs、architecture references、NIST SSDF/security planning skills；通常不需要外部 MCP。 |
| `build-agent` | 可重用 package manager、build system、dependency analysis 相關 skills/MCP，但不得自動取得 deployment 權限。 |
| `test-agent` | 可重用 security scanner、test framework、issue tracker、GitHub 相關 skills/MCP，但必須限制 credentials 與寫入範圍。 |
| `release-agent` | 可重用 CI/CD、GitHub、deployment verification 相關 skills/MCP，但 release/deploy 類操作必須有明確使用者意圖。 |

body 至少定義：

- 此 subagent 負責的 workflow domain。
- 必須讀取 `.claude/project-plan.md`。
- 必須產生 `.claude/workflows/{flow}.js`。
- 必須產生 `.claude/pipelines/{flow}/pipeline.yaml`。
- 必須產生 `.claude/pipelines/{flow}/pipeline_config.local.yaml` skeleton。
- 視需要產生 `.claude/pipelines/{flow}/scripts/*.py` deterministic executors。
- 不得 hardcode credentials、tokens、server URLs 或 local-only paths。
- workflow metadata 必須放在 `pipeline.yaml`。
- local secrets 與 machine-specific settings 必須放在 `pipeline_config.local.yaml`。

Subagent 範例：

```md
---
name: test-agent
description: Builds Testing, Security, and Review workflows for the current project.
tools: Read, Write, Edit, Glob, Grep, Bash
model: sonnet
---

You are the Testing, Security, and Review workflow architect for this project.

Inputs:
- .claude/project-plan.md
- Existing source code, CI files, and project conventions
- Reference knowledge base provided by project-architect or pipeline-architect

Responsibilities:
- Generate Claude workflows for testing and security flows.
- Create .claude/workflows/{flow}.js.
- Create .claude/pipelines/{flow}/pipeline.yaml.
- Create .claude/pipelines/{flow}/pipeline_config.local.yaml skeletons.
- Create deterministic Python scripts when a workflow needs repeatable CLI/API execution.

Rules:
- Never hardcode credentials, tokens, server URLs, or local-only paths.
- Keep workflow metadata in pipeline.yaml.
- Keep local secrets and paths in pipeline_config.local.yaml.
- Generated workflows must use agent() for filesystem or shell work.
```

### 四個 Subagents 的領域

| Agent | Domain | Workflow Examples |
|-------|--------|-------------------|
| `plan-agent` | Design / Planning | requirements-spec, architecture-doc, threat-model |
| `build-agent` | Build / Code Gen | code-gen, compile, package |
| `test-agent` | Testing / Security / Review | unit-test, sast-scan, dast-scan, sca-scan, secret-scan, code-review |
| `release-agent` | Delivery / Release | release, smoke-test, deployment-check |

每個 subagent 會依 `.claude/project-plan.md` 產生自己領域的 Claude workflows。

---

## 3. Subagents 產生 Claude Workflows

四個 subagents 的主要產出是 Claude workflows，而不是直接執行完整 pipeline。

### 產出結構

```text
{project}/
├── .claude/
│   ├── project-plan.md
│   ├── agents/
│   │   ├── plan-agent.md
│   │   ├── build-agent.md
│   │   ├── test-agent.md
│   │   └── release-agent.md
│   ├── workflows/
│   │   ├── threat-model.js
│   │   ├── sast-scan.js
│   │   └── release.js
│   └── pipelines/
│       └── {flow}/
│           ├── pipeline.yaml
│           ├── pipeline_config.local.yaml
│           └── scripts/*.py         # optional deterministic executors
```

`.claude/workflows/{flow}.js` 是主要可執行單位。`.claude/pipelines/{flow}/scripts/*.py` 不是每個 workflow 都必須有；只有當該 workflow 需要 repeatable CLI/API execution 時才產生。

例如：

| Workflow | 是否通常需要 scripts | 說明 |
|----------|----------------------|------|
| `threat-model.js` | 通常不需要 | 可由 workflow 內的 agents 讀取 codebase、架構文件與 project plan 後產生 threat model |
| `architecture-doc.js` | 通常不需要 | 多半是分析與文件生成 |
| `sast-scan.js` | 通常需要 | 需要穩定呼叫 scanner CLI 並解析結果 |
| `release.js` | 視情況需要 | 若要呼叫 build/deploy CLI 或 API，應放在 deterministic scripts |

### Metadata 與 Local Config 拆分

不要把 stage 順序與 workflow metadata 放在 gitignored config 內。

| 檔案 | 是否 commit | 用途 |
|------|-------------|------|
| `pipeline.yaml` | Yes | workflow metadata、stages、schema、script list、預設行為 |
| `pipeline_config.local.yaml` | No | credentials、server URLs、local paths、tokens |
| `scripts/*.py` | Yes | deterministic executors |
| `.claude/workflows/{flow}.js` | Yes | Claude workflow orchestration |

`.gitignore` 應包含：

```gitignore
.claude/pipelines/**/pipeline_config.local.yaml
.claude/pipelines/**/.draft/**/pipeline_config.local.yaml
```

### Workflow 產生規範

每個 workflow 應包含：

- `export const meta`
- 清楚的 phases
- 對應 stage 的 `agent()` 呼叫
- 若 workflow 需要 local config，第一個 stage 應做 config validation
- 需要資料傳遞時使用 schema
- 失敗時回報可重跑方式
- 不在 workflow script 中直接讀寫檔案或跑 shell

### Config Validation Stage

若 workflow 需要讀取 `pipeline_config.local.yaml`，第一個 stage 應透過 `agent()` 做 config validation。

此 validation agent 負責：

- 讀取 `.claude/pipelines/{flow}/pipeline_config.local.yaml`。
- 確認 required keys 是否存在。
- 確認必要 local paths 是否存在或格式合理。
- 判斷哪些 stages 可以執行，哪些 stages 因缺少設定應 skip。
- 以 schema 回傳 redacted config status。

validation agent 不得回傳 secrets 給 workflow JS，包括 tokens、passwords、API keys、private keys。
後續執行 Python/CLI/API 的 agent 或 script 應自行讀取 `pipeline_config.local.yaml`。

建議回傳格式：

```javascript
const configStatus = await agent({
  prompt: `Validate .claude/pipelines/sast-scan/pipeline_config.local.yaml.
Return only redacted status. Do not return tokens, passwords, API keys, or private keys.`,
  schema: CONFIG_STATUS_SCHEMA,
})
```

後續 stage 可依 `configStatus` 決定 run / skip，但不得把 secret value 拼入 prompt。

### Workflow Failure Handling

每個 generated workflow 應明確處理 stage failure，避免背景 workflow 失敗後只留下不完整的輸出。

Rules:

- 每個 stage 的 `agent()` 回傳應盡量包含 `success`、`stage`、`exitCode`、`stdoutSummary`、`stderrSummary`、`logPath`。
- 若某個 stage 失敗，workflow 應停止後續 dependent stages。
- workflow 應呼叫最後一個 reporter agent，產生 `.claude/pipelines/{flow}/reports/error-report.md`。
- error report 不得包含 secrets、tokens、passwords、API keys 或 private keys。

error report 應包含：

- workflow name
- failed stage
- script path
- exit code
- failing command / file / line if available
- stdout/stderr summary
- local log path
- suggested fix
- suggested rerun command

Failure handling 範例：

```javascript
try {
  const configStatus = await agent({
    prompt: `Validate .claude/pipelines/sast-scan/pipeline_config.local.yaml.
Return redacted config status only.`,
    schema: CONFIG_STATUS_SCHEMA,
  })

  const findings = await agent({
    prompt: `Run .claude/pipelines/sast-scan/scripts/analyze.py.
Return success, exit code, stdout/stderr summary, and log path.`,
    schema: STAGE_RESULT_SCHEMA,
  })

  if (!findings.success) {
    throw new Error(JSON.stringify(findings))
  }
} catch (error) {
  await agent({
    prompt: `Create .claude/pipelines/sast-scan/reports/error-report.md.
Use this failure context, redact secrets, and include rerun guidance:
${String(error)}`,
  })
  throw error
}
```

Workflow 範例：

```javascript
export const meta = {
  name: 'sast-scan',
  description: 'Run SAST analysis and summarize findings.',
  phases: [{ title: 'Prepare' }, { title: 'Analyze' }, { title: 'Report' }],
}

await agent({
  prompt: `Read .claude/pipelines/sast-scan/pipeline.yaml and pipeline_config.local.yaml.
Run the prepare stage using the configured script.
Report exit code and important output only.`,
})

const findings = await agent({
  prompt: `Run the SAST analyzer script and return normalized findings.`,
  schema: FINDINGS_SCHEMA,
})

await agent({
  prompt: `Summarize these findings and write the report requested by the workflow:
${JSON.stringify(findings)}`,
})
```

### Python Script 規範

每個 generated script：

- 使用 `_load_config()` 讀取 `pipeline_config.local.yaml`
- 透過 `_get()` 取得設定，必要時 fallback 到環境變數
- exit code `0` 表示成功，非 `0` 表示失敗
- stdout 輸出進度與摘要
- stderr 輸出錯誤
- 不 hardcode credentials、server URLs、local machine paths

---

## 4. `pipeline-architect` Skill

`pipeline-architect` 也是 Claude skill，不是 workflow。

它負責新增使用者想補充的 workflows，或修改四個 subagents 產生的 workflows。

### 主要用途

- 新增 plan 裡沒有，但使用者後來想要的 workflow
- 修改既有 workflow 的 stages、行為、config skeleton 或 scripts
- 修復 subagent 產生但不符合需求的 workflow
- 管理 draft → confirm 的人工確認流程

### 三種模式

| 情況 | 模式 |
|------|------|
| 使用者描述新需求，且沒有對應 workflow | 新建 workflow draft |
| 使用者指定既有 workflow 並提出修改 | 修訂 workflow draft |
| 使用者確認 draft | 將 draft 移到正式 workflow |

### 使用者指令範例

```text
使用 pipeline-architect 建立 Coverity 掃描 workflow，並將 HIGH impact issue 轉成 JIRA

使用 pipeline-architect 修改 sast-scan，只轉 HIGH impact，並啟用 dedup

使用 pipeline-architect 確認 sast-scan draft
```

### Draft 結構

```text
.claude/pipelines/{name}/.draft/
├── {name}.js
├── pipeline.yaml
├── pipeline_config.local.yaml
└── scripts/*.py
```

確認後移動到：

```text
.claude/workflows/{name}.js
.claude/pipelines/{name}/pipeline.yaml
.claude/pipelines/{name}/pipeline_config.local.yaml
.claude/pipelines/{name}/scripts/*.py
```

### 新建流程

1. 解析使用者意圖，決定 workflow 名稱、目的與 stages。
2. 查閱 reference knowledge base 或現有 workflows。
3. 產生 draft workflow JS。
4. 產生 `pipeline.yaml`。
5. 產生 `pipeline_config.local.yaml` skeleton。
6. 視需要產生 deterministic Python scripts。
7. 做跨 stage 一致性審查。
8. 顯示 Stage Summary，等待使用者確認或修訂。

### 修訂流程

1. 讀取正式 workflow 或既有 draft。
2. 分析修改意見。
3. 判斷受影響 stages。
4. 只重寫必要檔案。
5. 重新做跨 stage 一致性審查。
6. 更新 draft。
7. 顯示更新後 Stage Summary。

### Stage Summary 格式

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pipeline: sast-scan [DRAFT]
Stages: prepare → analyze → report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Stage 2 / analyze
  行為：對原始碼執行 SAST 掃描並輸出 normalized findings

  必填（未填無法執行）
    - scanner_path     pipeline_config.local.yaml
    - project_root     pipeline_config.local.yaml

  行為決策（有預設值，請確認符合需求）
    - impact 篩選      HIGH
    - dedup            enabled
    - timeout          60 minutes

  產出
    - .claude/pipelines/sast-scan/reports/findings.json

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
修改：使用 pipeline-architect 修改 sast-scan <修改意見>
確認：使用 pipeline-architect 確認 sast-scan draft
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 5. Reference Knowledge Base

`project-architect` 與 `pipeline-architect` 都可以使用 reference knowledge base 作為生成 workflows/scripts 的知識來源。

建議安裝位置：

```text
~/.claude/skills/project-architect/reference/
~/.claude/skills/pipeline-architect/reference/
```

或在本 repo 開發期保留：

```text
reference/
reference_cache/pipeline_scripts/
```

reference knowledge base 可包含：

- CLI 語法
- API endpoint 範例
- Jenkins / Coverity / JIRA / Gerrit / CodeTek / AISight 範例
- stage script templates
- config key conventions

產生的 workflow/script 可以參考這些內容，但不得複製 credentials 或 local-only path。

---

## 6. 安裝規劃

安裝腳本應安裝 skills 與可選 reference knowledge base。

### 安裝項目

```text
skills/project-architect/   → ~/.claude/skills/project-architect/
skills/pipeline-architect/  → ~/.claude/skills/pipeline-architect/
reference/                  → ~/.claude/skills/pipeline-architect/reference/
```

### 更新流程

```bash
# 1. 更新 reference knowledge base
python scripts/update_reference_cache.py

# 2. 安裝 / 更新 skills 與 reference knowledge base
bash install.sh
```

---

## 7. 結論

修正後的設計重點：

- `project-architect` 是 Claude skill，負責 project plan 與 4 個 subagents。
- `plan-agent`、`build-agent`、`test-agent`、`release-agent` 是 Claude subagents。
- 4 個 subagents 會依領域產生 Claude workflows。
- `pipeline-architect` 是 Claude skill，負責新增或修訂 workflows。
- deterministic execution 放在 generated Python scripts。
- orchestration 放在 Claude workflows。
- credentials/local config 與 workflow metadata 必須拆分。
