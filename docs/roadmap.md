# Delivery roadmap

本文件以 M0–M9 作唯一交付順序，取代歷史 DOCX 的 phases；正式契約以 [docs 索引](README.md)為準。每一 milestone 的驗收都建立在其 Dependencies 上，不以文件範例或空成功腳本冒充 runtime 成果。M0–M3 的 public contracts 已在責任文件固定；實作必須保留它們。M4–M9 僅固定交付邊界、依賴與 observable acceptance，內部 schema、實作工具及容量參數在該 milestone 實作時決定並同步責任文件。

## M0：正式架構與契約

- **Dependencies：** 核准的架構要求與歷史 DOCX 背景；不依賴 runtime、模型下載或部署環境。
- **Deliverables：** `docs/` source-of-truth 政策、四 plane／真實 Node 拓撲、domain states、portable `home-ai/v1` Manifest、Registry `/api/v1`、Agent protocol、security boundaries、開發 smoke tests、本 roadmap 與 ADR 0001–0004。M0 不建立實作檔。
- **Non-goals：** 執行服務、建立資料庫、啟動 Docker、拉模型、實測 chat/streaming；DOCX 更新或雙向同步。
- **Observable acceptance criteria：** 正式文件的相對路徑／anchors、Mermaid、YAML/JSON、TypeScript examples 通過靜態驗證；states、API、15/45 秒、generation、MCP gate 與 aliases 一致；M0–M9 每項都有四欄；完成 lifecycle/security walkthrough。[M0 smoke](development.md#m0文件與-contracts)提供檢查方式。

## M1：Bun workspace 與契約驗證

- **Dependencies：** M0；Apple Silicon arm64 Bun 與 Git。
- **Deliverables：** Bun workspace、單一 lockfile、固定工具版本、root dev/build/test/lint scripts、TypeScript shared contracts、Zod 或 JSON Schema 外部設定／Manifest validation、formatter/linter、Registry module interfaces 與明確標示非持久的 fixtures。Repository layout、commands 與版本管理依 [development](development.md#prerequisites-與程序位置)。Dev authentication adapter 使用個別 bearer credential、穩定 principalId 與 server-side grants；read/manage 分離，不提供 anonymous admin。
- **Non-goals：** PostgreSQL 持久保證、真實 Node enrollment、Docker lifecycle、LLM/MCP runtime、Redis、完整 Portal/SSO。
- **Observable acceptance criteria：** Frozen dependency install 不修改 lockfile；lint/build/tests 成功。Valid Manifest 通過，固定 node/URL、literal secret、privileged、host mount、unknown runtime/fields 與 permission mismatch fixtures 被拒絕；generation/ownership/availability fixtures fail closed。Fixture mode 不建立真實 Nodes，重啟重置行為有明確標示。[M1 smoke](development.md#m1bun-workspace-與-deterministic-fixtures)不要求模型或 DB。

## M2：PostgreSQL Registry

- **Dependencies：** M1；Docker Compose `core` 與 PostgreSQL、本機管理員的 principals/grants、permission／binding catalogs。
- **Deliverables：** Bun/TypeScript Registry modular monolith 的 Definitions、Deployments、Nodes、Discovery、Authorization/audit modules；PostgreSQL 自此成為權威持久層，保存 definitions、intent、generation、assignment/reservations、commands、credential digests/revocation、sequence、idempotency 與 audit。交付 migrations、受限管理設定與完整 [REST catalog](control-plane.md#management-paths)，Agent routes 的 server contract 先由 protocol fixtures 驗證。Mutation、idempotency、audit 同 transaction；row locks／constraints 保護競爭更新；外部 side effects 在交易外。
- **Non-goals：** 真實 Agent 或 Docker mutation、跨 Node failover、Redis/message broker、分散式交易、多 Registry leader、MCP/模型 runtime、公開 audit 管理 route。
- **Observable acceptance criteria：** [M2 smoke](development.md#m2postgresql-registry-durability)證明重啟不遺失已提交 intent/idempotency/audit/revocation；無 Node 時 pending/AwaitingPlacement；同 key 重送不重複 mutation，不同 body 回 409；version conflict、unauthorized、DB/audit failure 不產生部分 intent。Public response 不含 resolvedBindings/secrets；PostgreSQL 故障停止新 mutation/派工並讓 discovery fail closed。

## M3：Host Node Agent 與服務 lifecycle

- **Dependencies：** M2；Mac host 的 Bun/TypeScript Agent、Docker daemon、一次性 enrollment token、owner-only credential/journal/store、核准的 hello-service local image ID 及 logical bindings。
- **Deliverables：** 唯一真實 Node `dev-macbook` enrollment；Agent 是 Bun/TypeScript host process，有 process lock 與 durable boot/command journal。Outbound pull 每 15 秒 heartbeat、45 秒 offline；placement 依 constraints、真實 inventory、已提交 reservation 及 bindings，自符合候選按 nodeId 字典序選擇。實作 current generation executionAuthorization/resolvedImage、受控 Docker allowlist/labels/mapping、HTTP health、start/stop intent、一次性 restart、credential rotation/revoke 與 audit。完整契約依 [Control Plane](control-plane.md)及 [security model](security.md)。
- **Non-goals：** Fake Nodes/GPU inventory、inbound Agent RPC、任意 shell/exec、管理 host Ollama 或 Compose core containers、MCP 真實 discovery、LLM、資源不足自動 failover、unhealthy 自動 restart、production mTLS automation。
- **Observable acceptance criteria：** [M3 smoke](development.md#m3唯一真實-node-與-lifecycle)完成真實 hello-service registration → deployment → running/healthy、stop/start/restart。只存在 dev-macbook，roles 不增加 Node 數。資源不足保持 pending 且無未授權 Docker mutation；offline 不冒充 stopped、不釋放未知 runtime 的 reservation。Stale report 不改新 intent，合法 heartbeat 中的 rejectedReports 不影響正確 liveness；同 commandId 不二次 restart。Token single-use、跨 Node/session/sequence 拒絕、ownership mismatch、secret redaction、audit rollback、rotation/revoke 均有證據。

## M4：MCP discovery 與 permission enforcement

- **Dependencies：** M3 的有效 deployment、health、generation、audit；已登記 permissions 與受信任 discovery/user delegation。
- **Deliverables：** 實際 hello-service MCP Streamable HTTP integration，採既定 protocol revision；完整 tools/list discovery、schema validation、permission mapping、capability 投影及撤銷；backend test caller 與 service-side authorization 接通。Grant 及 availability 在每次呼叫重新驗證。
- **Non-goals：** 依賴 LLM 產生工具呼叫、Portal chat、RAG、embedding、任意 MCP server 自動信任；不更換既定 transport 或提前選 workflow engine。
- **Observable acceptance criteria：** [M4 smoke](development.md#m4實際-mcp-discovery-與授權)證明 Manifest allowed tools、完整實際 tools/list、完整 permission mapping 與 healthy 同時通過才可用；多頁不漏 tool。錯 audience／permission／arguments 被拒絕；unhealthy/offline、generation 變更或 tool 集合不符撤銷整份 capability 快照。合法 deterministic tools/call 有結果與 audit，完全不需要 Phi-3。

## M5：LiteLLM 與本機模型

- **Dependencies：** M4 已建立平台服務及授權基線；host Ollama 固定 `phi3:3.8b`、LiteLLM 可達的 `OLLAMA_API_BASE` 與受限 gateway credential。純 chat request 不需呼叫 MCP，交付排序不代表每次 inference 都有 MCP runtime dependency。
- **Deliverables：** LiteLLM gateway、兩個核准 chat aliases `home-large`／`home-fast` 都映射到 `phi3:3.8b`，chat/streaming 與受控錯誤處理；部署設定與 backend address 不進 portable Manifest。Caller boundary 依 [ADR 0004](adr/0004-litellm-logical-model-aliases.md)。
- **Non-goals：** Embedding alias、70B–80B 模型、GPU cluster、模型 quality/throughput SLA、Phi-3 tool-calling 保證、瀏覽器持有 gateway key、Portal/Orchestrator 完整產品流程。
- **Observable acceptance criteria：** [M5 smoke](development.md#m5phi338b-chat-與-streaming)實際證明兩 aliases 各自可取得非空 chat response 及 streaming deltas／正常終止；unknown alias、無 credential 與 backend unavailable 有受控失敗。Caller 無 direct Ollama/vLLM/GPU node 路徑、無 wildcard bypass。M0 文件驗收不以前述真實模型測試為 gate。

## M6：Portal、Orchestrator 與 Authentik

- **Dependencies：** M4 的 MCP/authorization、M5 logical aliases；M2 durable audit 與現有 authentication adapter 邊界。
- **Deliverables：** Portal 經 Traefik／Authentik 入口登入，展示 definitions 與 deployments 的分離狀態；Orchestrator 提供 chat/streaming 並整合可用且已授權 tools。Backend 驗證 identity/resource scope，逐次確認 tools/call；session 與 delegation 落實 [security model](security.md)。
- **Non-goals：** 用 SSO 取代 object authorization、直接 Docker/backend 存取、RAG、任意工具自治、提前承諾特定 conversation schema 或 agent framework。
- **Observable acceptance criteria：** 登入使用者從 Portal 完成 chat/streaming，可看到 desired/observed/health 及 pending command；合法工具流程有 traceable audit。Deterministic model proposal fixture 證明無權／撤銷 grant／stale capability 不會觸發 tool，不把 Phi-3 是否自主選工具當安全驗收。偽造 identity headers、過期 session、CSRF 與錯 audience 被拒絕；browser 不持有 LiteLLM/Node credentials，所有模型呼叫只帶 aliases。

## M7：RAG、embedding 與 Qdrant

- **Dependencies：** M6 已驗證 principal/scope、Orchestrator 邊界；M2 PostgreSQL 的權威 ACL metadata；M5 LiteLLM routing。
- **Deliverables：** 文件 ingestion/retrieval/citations、Qdrant 衍生向量索引、此階段才定義及啟用 embedding logical alias 與適用模型；原始文件／chunks／embeddings 可追溯 owner/scope/ACL revision，retrieval 層強制 ACL。Index layout、chunking 與模型選型在 M7 定義，不改變權限權威。
- **Non-goals：** 以 `phi3:3.8b` 冒充 embedding model、全庫先取回再讓模型隱藏、跨使用者私有 cache、以向量 index 取代 ACL authority。
- **Observable acceptance criteria：** 兩個使用者／兩個 scopes 的正負測試涵蓋 ingestion、vector search、直接 document ID、citations、cache 及撤銷後重試；未授權 chunks 不進 reranker/prompt/response。Index 落後時依權威 ACL fail closed；embedding caller 僅用已核准 alias，模型不可達時有可追蹤失敗與受控恢復。

## M8：語音與家庭自動化整合

- **Dependencies：** M6 的 authenticated Portal/Orchestrator、M4 tool permission/audit；只有需要文件知識的流程才額外依賴 M7。
- **Deliverables：** STT/TTS 與語音輸入輸出、家庭自動化 tools 的整合邊界；服務依 portable Manifest、logical bindings、既有 lifecycle/authorization 接入。具物理副作用的 action 保留明確使用者 intent 及必要的確認流程。
- **Non-goals：** 特定麥克風／speaker／家電硬體、workflow engine 選型、全屋無人確認自治、繞過 permissions 的語音捷徑。
- **Observable acceptance criteria：** 測試音訊可依授權流程取得文字／回應音訊，服務失效呈受控錯誤；家庭 action 可先在 simulator 驗證合法觸發、拒絕及 audit，再對經核准的實體裝置驗收。語音／模型輸出不能新增 grants，撤銷權限立即阻止後續 action；具副作用的重試不會無意重做操作。

## M9：多實體節點與運維強化

- **Dependencies：** M3 真實 Node/lifecycle 基線及已部署 M4–M8 服務；實體 Core/GPU/worker 與儲存環境、跨主機 TLS 網路及各主機管理權。
- **Deliverables：** 每個真實 execution host 獨立 enrollment；以 Deployment 改變 placement/bindings，LiteLLM routing 可接核准 GPU backend，NAS 用作授權檔案／備份。交付完整 metrics/traces/dashboard、備份還原演練、production mTLS identity/rotation/revocation；具體容量、模型與工具由量測及環境決定。
- **Non-goals：** 承諾 k3s、自動跨 Node failover、未量測的 70B–80B 效能、把 storage-only NAS 註冊成 execution Node、改寫 Manifest 固定 host、因 mTLS 省略 application authorization。
- **Observable acceptance criteria：** 同一 portable Manifest 可在另一符合 constraints 的真實 Node 部署而不改 node/IP；Portal/Orchestrator aliases 不變。斷線／credential 或 certificate revoke 阻止新派工並撤銷 capability，不錯判 stopped 或另起衝突副本。Request → intent → command → observation 可追蹤且無 secrets；restore 保留 intent、audit、idempotency 與 revoke 語意，不重播 restart；mTLS rotation/跨 Node 拒絕通過驗證。模型 latency/資源數據以實測報告提供。

Authentik 到 M6、Qdrant/embedding 到 M7、完整 observability 與 production mTLS automation 到 M9；這些延後不延後 M1 server-side authorization、M2 durable audit 或 M3 per-node credential/revoke。Redis 只有後續出現明確需求時另行評估，不是 M0–M3 或全 roadmap 的無條件依賴。
