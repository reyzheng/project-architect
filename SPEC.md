# Jenkins Pipeline System — Specification Document

## 1. 系統架構

### 整體資料流

```
使用者
  │
  ▼
/project-architect               ← 規劃模式：問答 → .claude/project-plan.md
  │
  ▼ /project-architect run
  │
  ├─ plan-agent    (parallel)    → .pf-plan/{flow}/scripts/*.py
  ├─ build-agent   (parallel)    → .pf-build/{flow}/scripts/*.py
  ├─ test-agent    (parallel)    → .pf-test/{flow}/scripts/*.py
  └─ release-agent (parallel)    → .pf-release/{flow}/scripts/*.py
         │
         │  （人工介入：新增或修改某個 pipeline）
         ▼
/pipeline-architect              ← 人工工具：迭代式設計迴圈
```

### Workflows 分工

| Workflow | 定位 | 說明 |
|----------|------|------|
| `/project-architect` | 主要入口 | SDLC 規劃師 + 執行引擎：問答生成計劃，並行執行所有 pipelines |
| `/pipeline-architect` | 人工介入工具 | 新增 agents 未建立的 pipeline，或修改現有 pipeline |
| `/pipeline-runner` | 本地執行工具 | 依序執行指定 pipeline 的所有 stage scripts |

### 產出結構

```
{project}/
├── .claude/
│   └── project-plan.md          ← /project-architect 規劃模式輸出
├── .pf-plan/
│   └── {flow}/
│       ├── scripts/*.py         ← plan-agent 生成並執行（commit ✓）
│       └── pipeline_config.yaml ← credentials（gitignored）
├── .pf-build/
│   └── {flow}/
│       ├── scripts/*.py
│       └── pipeline_config.yaml
├── .pf-test/
│   └── {flow}/
│       ├── scripts/*.py
│       └── pipeline_config.yaml
└── .pf-release/
    └── {flow}/
        ├── scripts/*.py
        └── pipeline_config.yaml
```

`.gitignore`：`.pf-*/**/pipeline_config.yaml`

### 兩個角色分工（每個 pipeline）

| 產出物 | 角色 | 說明 |
|--------|------|------|
| `scripts/*.py` | Executor | Deterministic 執行 CLI 指令、API 呼叫 |
| `pipeline_config.yaml` | Config | Credentials 與路徑（使用者填寫） |

---

## 2. `/project-architect` — SDLC 執行引擎

`/project-architect` 是一個全域 workflow（安裝於 `~/.claude/workflows/project-architect.js`），提供兩種模式：

### 兩種模式

| 模式 | 指令 | 說明 |
|------|------|------|
| 規劃模式 | `/project-architect` | 問答收集專案資訊，輸出 `.claude/project-plan.md` |
| 執行模式 | `/project-architect run` | 讀取 plan，並行啟動 4 個 SDLC agents |

### 規劃模式

問答流程（新專案 / 既有專案兩種路徑），由單一 `project-planner` agent 執行。輸出 `.claude/project-plan.md`，包含推薦 pipelines 與手動執行指令。

**新專案路徑**：問答 6 題（名稱、語言、類型、安全等級、部署目標）→ 依優先度規則決定 pipelines。

**既有專案路徑**：問答 3 題 + agent 自動掃描 codebase（語言偵測、現有 Jenkinsfile）→ 確認後決定補充哪些 pipelines。

### 執行模式

```
/project-architect run
/project-architect run --only sast-scan,threat-model
```

讀取 `.claude/project-plan.md` 後，並行啟動 4 個 SDLC agents：

| Agent | Phase | 負責 flows | 產出目錄 |
|-------|-------|-----------|---------|
| `plan-agent`    | Phase 1 — Design        | requirements-spec, architecture-doc, threat-model | `.pf-plan/`    |
| `build-agent`   | Phase 2 — Code Gen      | code-gen                                          | `.pf-build/`   |
| `test-agent`    | Phase 3 — Testing       | compile-test, unit-test, sast-scan, dast-scan, sca-scan, secret-scan, code-review | `.pf-test/`    |
| `release-agent` | Phase 4 — Delivery      | release, smoke-test                               | `.pf-release/` |

每個 agent 對每個 flow：
1. 讀取 `~/.claude/workflows/pipeline-architect/reference/` 參考模組
2. 產生 `.pf-{agent}/{flow}/scripts/*.py` 與 `pipeline_config.yaml`
3. 執行腳本，回報 pass / fail / skip
4. 失敗不影響其他 agents（各自獨立）

### 錯誤處理規則

| 情況 | 行為 |
|------|------|
| 單個 flow 失敗 | 繼續跑同 phase 其他 flows，記錄失敗 |
| 整個 agent 失敗 | 其他 agents 不受影響 |
| `.claude/project-plan.md` 不存在 | 中止，提示先執行 `/project-architect` |
| `pipeline_config.yaml` 未填寫 | 標記為 skip，提示使用者填寫後重跑 |

### 執行摘要格式

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Project: my-service  執行完成
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
plan-agent    ✓ requirements-spec  ✓ architecture-doc  ✗ threat-model
build-agent   ✓ code-gen
test-agent    ✓ compile-test  ✓ sast-scan  ✓ sca-scan  ✓ secret-scan  ✗ code-review
release-agent ✓ release  ✓ smoke-test
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
失敗 2 個 pipeline：
  .pf-plan/threat-model/    → 修改：/pipeline-architect
  .pf-test/code-review/     → 修改：/pipeline-architect
重跑：/project-architect run --only threat-model,code-review
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 3. `/pipeline-architect` — Pipeline 設計 Workflow

`/pipeline-architect` 是一個全域 workflow（安裝於 `~/.claude/workflows/pipeline-architect.js`），根據現有 `.draft/` 狀態與 args 內容自動決定模式。

主要用途：
- 新增 `/project-architect run` 未建立的 pipeline
- 修改已建立的 pipeline（調整行為決策）

### 三種模式

| 情況 | 模式 |
|------|------|
| 無 draft，有描述 | 新建 |
| 有 1 個 draft，有修改意見 | 修訂 |
| 有多個 draft，args 含名稱 | 修訂指定 pipeline |
| 有多個 draft，args 無名稱 | 列出選項，提前退出 |
| args 為 "ok / done / confirm" | 確認，.draft/ 移到正式位置 |

**使用者指令範例：**

```
/pipeline-architect "我要 Coverity 掃描流程，並將弱點轉為 JIRA issue"
                                          ← 新建

/pipeline-architect "只轉 HIGH impact，開啟 dedup"
                                          ← 修訂（只有一個 draft）

/pipeline-architect security-scan "只轉 HIGH impact"
                                          ← 修訂（多個 draft，指定名稱）

/pipeline-architect ok                    ← 確認（只有一個 draft）
/pipeline-architect security-scan ok      ← 確認（多個 draft，指定名稱）
```

### 新建模式 Phases

**Phase 1 — 意圖解析**
解析使用者描述，決定 pipeline 名稱與 stages 順序（source、coverity、jira、codetek、gerritsubmit、aisight）。

**Phase 2 — 並行生成**
每個 stage 一個 agent，同時進行：
- 讀取 `~/.claude/workflows/pipeline-architect/reference/` 中對應的 Python 參考模組
- 生成完整的 Python 腳本，包含 `_load_config()` + `_get()` config 讀取 helper
- 所有 credentials 和路徑均從 `pipeline_config.yaml` 讀取，不 hardcode

**Phase 3 — 生成 Workflow JS**
依據已生成的 stages，產出 `{name}.js`：
- `export const meta` 含 pipeline 名稱與 phases
- 每個 stage 以 `await agent()` 呼叫對應 Python script
- 有資料依賴的 stage 間，透過 `schema` 傳遞結構化結果（例如 Coverity 結果傳給 JIRA stage）

**Phase 4 — 跨 stage 審查**
確認 stage 間資料一致性（Python scripts 的輸出路徑、workflow JS 的資料傳遞），生成 `pipeline_config.yaml` skeleton。

**Phase 5 — 寫入 .draft/**
寫入 `.claude/pipelines/{name}/.draft/`。

**Phase 6 — 顯示 Stage Summary**

### 修訂模式 Phases

**Phase 1 — 讀取現有 draft**：讀取 `.draft/scripts/*.py`（依 `# pf-stages:` 還原順序）與 `pipeline_config.yaml`。

**Phase 2 — 影響範圍分析**：agent 分析修改意見，判斷哪些 stages 需重生成（考慮 stage 間相依關係）。

**Phase 3 — 重生成受影響 stages**：只重跑受影響的 stages，其餘保留。

**Phase 4 — 重新生成 Workflow JS**：依更新後的 stages 重新產出 `{name}.js`。

**Phase 5 — 重新審查**：確認跨 stage 一致性。

**Phase 6 — 更新 .draft/**

**Phase 7 — 顯示更新後 Stage Summary**

### 確認模式

將 `.draft/` 內容移至正式位置，刪除 `.draft/` 目錄，確認 `.gitignore` 含 `.claude/pipelines/**/pipeline_config.yaml`。

```
.claude/pipelines/{name}/.draft/{name}.js        → .claude/workflows/{name}.js
.claude/pipelines/{name}/.draft/scripts/         → .claude/pipelines/{name}/scripts/
.claude/pipelines/{name}/.draft/pipeline_config.yaml → .claude/pipelines/{name}/pipeline_config.yaml
```

### Stage Summary 格式

每次結束後顯示，使用者依此決定是否需要修改：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pipeline: security-scan [DRAFT]
Stages: source → coverity → jira
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Stage 2 / coverity
  行為：對原始碼執行 Coverity 靜態分析

  ⚠ 必填（未填無法執行）
    • build_command    從 pipeline_config.yaml 讀取
    • server_url       從 pipeline_config.yaml 讀取

  ● 行為決策（有預設值，請確認符合需求）
    • impact 篩選      ALL（HIGH / MEDIUM / LOW 全轉）
    • 分析模式         incremental
    • checker 範圍     全部啟用
    • 逾時設定         60 分鐘

  ○ 次要設定（安全預設值，通常不需修改）
    • 輸出報告路徑     /tmp/{name}/coverity_results.json

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
修改  /pipeline-architect "{name}" "<修改意見>"
確認  /pipeline-architect ok
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 4. 產出物規範

### Python 腳本規範

每個生成的腳本：
- 使用 `_load_config()` 讀取 `pipeline_config.yaml`，透過 `_get()` 取值，fallback 到環境變數
- `_load_config()` 路徑：`os.path.join(os.path.dirname(__file__), '..', 'pipeline_config.yaml')`（scripts/ 上一層）
- exit code 0 = 成功，非 0 = 失敗
- stdout 輸出進度，stderr 輸出錯誤
- 不 hardcode 任何 URL、路徑、credentials

### Workflow JS 規範

每個生成的 workflow：

```javascript
export const meta = {
  name: '{name}',
  description: '...',
  phases: [{ title: 'Source' }, { title: 'Coverity' }, { title: 'JIRA' }],
}

// 各 stage 依序執行（有資料依賴，不並行）
await agent({
  prompt: `Run python3 .claude/pipelines/{name}/scripts/source.py
           Read pipeline_config.yaml for config. Report exit code and output.`,
})

const defects = await agent({
  prompt: `Run python3 .claude/pipelines/{name}/scripts/coverity.py ...`,
  schema: DEFECTS_SCHEMA,   // 有資料需傳遞給下一 stage 時使用
})

await agent({
  prompt: `Run python3 .claude/pipelines/{name}/scripts/jira.py
           Defects: ${JSON.stringify(defects.items)}`,
})
```

### pipeline_config.yaml 的 Stage 順序 Metadata

新建模式寫入 `pipeline_config.yaml` 時，第一行固定加入：

```yaml
# pf-stages: source coverity jira
```

修訂模式 Phase 1 讀取此行來還原正確的 stage 順序。若此行缺失，fallback 至字母排序。

---

## 5. 安裝 (install.sh)

### 安裝流程

`bash install.sh` 執行以下三步：

**[1/3] 安裝 workflow scripts**
```
workflows/pipeline-architect.js   →  ~/.claude/workflows/pipeline-architect.js
workflows/pipeline-runner.js      →  ~/.claude/workflows/pipeline-runner.js
workflows/project-architect.js    →  ~/.claude/workflows/project-architect.js
```
安裝後，任何 repo 都能呼叫 `/pipeline-architect`、`/pipeline-runner`、`/project-architect`。

**[2/3] 安裝參考模組**
```
rfp_cache/pipeline_scripts/  →  ~/.claude/workflows/pipeline-architect/reference/
```
參考模組是 `/pipeline-architect` Phase 2 agents 的知識來源（CLI 語法、API endpoint、server URL）。
`/project-architect run` 的 SDLC agents 也使用相同的 reference 目錄。

**前置條件**：`rfp_cache/pipeline_scripts/` 必須存在，否則 install.sh 會報錯並中止。

### 更新流程

```bash
# 步驟 1：從 Gerrit clone Pipeline Framework，填充 rfp_cache/
python scripts/update_knowledge_base.py

# 步驟 2：重新安裝 workflows + 參考模組
bash install.sh
```
