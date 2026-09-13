# Home AI Platform Architecture Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建立一套可實作、可驗證且彼此一致的 Home AI Platform 正式架構文件，並以 `docs/` 取代 v0.1 DOCX 成為 source of truth。

**Architecture:** 文件依整體拓撲、平台契約、控制與安全、交付路線拆分；ADR 記錄已固定且會影響後續實作的決策。M0–M3 完整定義 public contracts，M4–M9 只固定邊界、依賴與驗收，不提前設計內部實作。

**Tech Stack:** Markdown、Mermaid、YAML、JSON、TypeScript interface examples、Bun、Docker Compose、PostgreSQL、LiteLLM、Ollama、MCP Streamable HTTP

**Spec:** `Home_AI_Platform_Spec_v0.1.docx`（歷史背景）；本計畫的 Global Constraints 與各 Task 是核准後的新架構要求。

## Global Constraints

- `docs/` 是正式 source of truth；`Home_AI_Platform_Spec_v0.1.docx` 只保留為歷史文件，不修改、不雙向同步。
- 文件使用繁體中文；API、schema、程式識別字與專有名詞保留英文。
- M0–M3 做到決策完整；M4–M9 固定邊界、依賴、non-goals 與可觀測 acceptance criteria，不過早設計內部細節。
- `ServiceDefinition` 不得包含固定 node、完整 runtime URL、host IP 或直接 host path；執行位置屬於 `Deployment`。
- desired state 是 `running | stopped`；observed state 是 `pending | deploying | running | stopping | stopped | failed | unknown`；health state 是 `unknown | starting | healthy | unhealthy`。
- Node Agent 採 outbound pull；每 15 秒 heartbeat，45 秒未收到即視為 offline；stale generation 不得覆寫新 intent。
- MCP transport 採 Streamable HTTP；只有宣告 tools、實際 `tools/list` 與完整 permission mapping 相符且服務 healthy 時，capability 才可用。
- 本機只註冊真實 Node `dev-macbook`；Compose profiles/labels 僅表達 core、gpu、worker、storage roles。
- Host Ollama 固定使用 `phi3:3.8b`；LiteLLM 由 `OLLAMA_API_BASE` 連線，`home-large` 與 `home-fast` 均映射到該模型；embedding alias 延至 M7。
- Portal 與 Orchestrator 只可使用 LiteLLM logical aliases，不得直接連 Ollama、vLLM 或實體 GPU node。
- PostgreSQL 自 M2 起為權威持久層；Redis 不在 M0–M3 無條件加入。
- 禁止 `TODO`、`TBD`、未決 placeholder 與互相矛盾的敘述。

---

### Task 1: 文件索引與整體架構

**Files:**
- Create: `docs/README.md`
- Create: `docs/architecture.md`
- Modify: `AGENTS.md`

**Interfaces:**
- Consumes: Global Constraints 與歷史 DOCX 的專案定位。
- Produces: 全文件閱讀順序、四 plane 定義、拓撲與下游文件使用的術語。

- [ ] **Step 1: 建立正式文件索引**

  在 `docs/README.md` 說明 source-of-truth 政策、推薦閱讀順序、每份文件責任、歷史 DOCX 狀態與一致的術語表。

- [ ] **Step 2: 描述四 plane 與拓撲**

  在 `docs/architecture.md` 定義 Edge、Control、Execution、Data planes，描述 Mac 單節點開發拓撲與未來多實體節點拓撲；加入可解析的 Mermaid component、trust-boundary、LLM request flow 圖。

- [ ] **Step 3: 修正 repository 指引**

  將 `AGENTS.md` 的 source-of-truth 敘述改為 `docs/`，並明確標示 DOCX 為歷史參考。

- [ ] **Step 4: 驗證與提交**

  執行 Markdown 相對連結與 Mermaid fence 基本檢查，確認未出現 placeholder，提交 `docs: establish architecture source of truth`。

### Task 2: Domain model 與 Manifest contract

**Files:**
- Create: `docs/domain-model.md`
- Create: `docs/service-manifest.md`
- Create: `docs/adr/0003-separate-manifest-and-deployment.md`

**Interfaces:**
- Consumes: Task 1 的四 plane、術語與 source-of-truth 政策。
- Produces: `Node`、`ServiceDefinition`、`Deployment`、`Capability`、`AgentCommand` 與 `home-ai/v1` Manifest contract。

- [ ] **Step 1: 定義 domain entities 與 state machines**

  記錄 entity ownership、identity、relationships、desired/observed/health state、generation fencing 與 capability availability rules；TypeScript interface examples 必須使用固定欄位名稱與 enums。

- [ ] **Step 2: 定義 versioned Manifest**

  記錄 validation、forward compatibility、named ports、routes、placement constraints、logical storage/secrets、permissions 與 MCP Streamable HTTP discovery 規則。

- [ ] **Step 3: 提供 valid 與 invalid examples**

  valid `hello-service` 必須 portable；invalid cases 必須涵蓋固定 URL/node、literal secret、privileged、host mount、unknown runtime 與 capability/permission mismatch，並逐例說明拒絕原因。

- [ ] **Step 4: 記錄 Manifest 與 Deployment 分離 ADR 並提交**

  ADR 包含 context、decision、consequences、rejected alternatives；驗證 YAML 可解析、TypeScript fences 可轉譯檢查，提交 `docs: define platform domain and manifest contract`。

### Task 3: Control Plane 與安全模型

**Files:**
- Create: `docs/control-plane.md`
- Create: `docs/security.md`
- Create: `docs/adr/0001-registry-modular-monolith.md`
- Create: `docs/adr/0002-node-agent-outbound-pull.md`

**Interfaces:**
- Consumes: Task 2 entities、states、generation 與 Manifest restrictions。
- Produces: `/api/v1` REST contract、reconciliation、Agent protocol、authorization 與 audit boundaries。

- [ ] **Step 1: 設計 Registry-centered modular monolith**

  定義 PostgreSQL ownership、modules、transaction boundaries、service discovery、`application/problem+json` errors、`Idempotency-Key` 行為與完整 API paths。

- [ ] **Step 2: 定義 lifecycle 與 outbound-pull Agent protocol**

  說明 start/stop desired-state mutation、restart idempotent command、15/45 秒 timing、resource-shortage pending reason、offline handling 與 stale generation fencing。

- [ ] **Step 3: 固定 security controls**

  定義 trust boundaries、one-time enrollment token、revocable node credential、rotation、Docker allowlist 與 label ownership、MCP permission enforcement、RAG ACL、audit 與 secret policy。

- [ ] **Step 4: 記錄兩份 ADR 並提交**

  ADR 記錄 modular monolith 與 outbound pull 的選擇及取捨；核對 API、enum、timings 與 Task 2 一致後，提交 `docs: specify control plane and security model`。

### Task 4: 開發流程 Roadmap 與 LLM ADR

**Files:**
- Create: `docs/development.md`
- Create: `docs/roadmap.md`
- Create: `docs/adr/0004-litellm-logical-model-aliases.md`

**Interfaces:**
- Consumes: Task 1 拓撲、Task 2 contracts、Task 3 lifecycle 與 security boundaries。
- Produces: Apple Silicon 開發方式、M0–M9 交付順序與 LLM access boundary。

- [ ] **Step 1: 文件化本機開發拓撲與 prerequisites**

  說明 Apple Silicon、Bun、Docker Compose、host Ollama、LiteLLM、`phi3:3.8b`、`OLLAMA_API_BASE`，以及單一 `dev-macbook` Node 與 profiles/labels 的角色模擬。

- [ ] **Step 2: 定義 M0–M5 smoke-test 流程**

  每個流程列出 prerequisites、commands/requests、預期 observable result；`phi3:3.8b` chat/streaming 實測屬 M5，非本次文件交付 gate。

- [ ] **Step 3: 以 M0–M9 取代舊 phases**

  每個 milestone 都列 dependencies、deliverables、non-goals、acceptance criteria；M0–M3 決策完整，M4–M9 僅固定邊界。

- [ ] **Step 4: 記錄 LiteLLM alias ADR 並提交**

  說明所有 callers 只用 `home-large`/`home-fast` 等 logical aliases，禁止 direct backend access；核對 aliases 與 milestone ordering 後，提交 `docs: define development path and roadmap`。

### Task 5: 全文件一致性與驗證

**Files:**
- Modify: `docs/README.md`
- Modify: `docs/architecture.md`
- Modify: `docs/domain-model.md`
- Modify: `docs/service-manifest.md`
- Modify: `docs/control-plane.md`
- Modify: `docs/security.md`
- Modify: `docs/development.md`
- Modify: `docs/roadmap.md`
- Modify: `docs/adr/*.md`

**Interfaces:**
- Consumes: Tasks 1–4 全部文件。
- Produces: 可追蹤、無 contract drift、可由後續實作直接採用的完整文件集。

- [ ] **Step 1: 執行 machine-readable validation**

  驗證 Markdown relative links、Mermaid fences、所有 YAML/JSON examples 與 TypeScript interface snippets；列出實際命令與結果。

- [ ] **Step 2: 執行 contract consistency scan**

  跨文件比對 enums、欄位名稱、API paths、15/45 秒 timing、model aliases、MCP transport 與 source-of-truth 政策，移除 placeholder 或矛盾。

- [ ] **Step 3: walkthrough lifecycle scenarios**

  逐一確認 registration-to-healthy、resource-shortage pending、heartbeat timeout、stale result fencing、unhealthy/offline capability removal、MCP declaration/permission mismatch rejection，以及 idempotent start/stop/restart。

- [ ] **Step 4: 驗證 roadmap completeness 並提交**

  確認 M0–M9 各自具 dependencies、deliverables、non-goals、observable acceptance criteria；提交任何一致性修正為 `docs: verify architecture consistency`。
