# Home AI Platform 正式文件

Home AI Platform 是家庭內網優先的自架 AI 基礎設施。核心交付是可擴充的服務平台：以宣告式 Manifest 註冊服務，經部署、健康檢查與授權後，將 Web UI 與 AI Capability 提供給 Portal 與 Orchestrator。聊天、RAG、語音及家庭自動化共用這些平台契約。

## 文件權威與變更政策

`docs/` 下的正式架構、契約、開發、路線與 ADR 文件是 architecture source of truth；實作與 review 均以這些文件為準。[Home_AI_Platform_Spec_v0.1.docx](../Home_AI_Platform_Spec_v0.1.docx) 僅保存專案歷史背景，不再修改，也不做雙向同步。正式文件取代 DOCX 中衝突的技術、部署與階段決策。

`docs/superpowers/plans/` 是文件工作的執行紀錄，不是另一套 runtime contract。變更正式決策時，先更新責任文件與受影響的交叉引用；若更改 ADR 決策，新增取代決策並標示關係，保留原有脈絡。若正式文件之間出現衝突，應修正一致後再依其實作，不能退回 DOCX 選擇另一版本。

文件採繁體中文；API、schema、identifier 與專有名詞保留英文。M0–M3 契約須達到可直接實作的決策完整度；M4–M9 先固定邊界、依賴、non-goals 與可觀測驗收，不將後期元件誤列為前期必要依賴。

## 推薦閱讀順序與責任

下表列出完整正式文件集的責任與推薦閱讀順序；各契約以對應責任文件為準。

| 順序 | 文件 | 責任 |
| --- | --- | --- |
| 1 | [architecture.md](architecture.md) | 專案邊界、四 plane、單節點與多節點拓撲、信任邊界及 LLM request flow。 |
| 2 | [domain-model.md](domain-model.md) | `Node`、`ServiceDefinition`、`Deployment`、`Capability`、`AgentCommand` 的 identity、ownership、關係及 state machines。 |
| 3 | [service-manifest.md](service-manifest.md) | `home-ai/v1` 宣告契約、validation、可攜性、named ports、logical storage/secrets、permissions 與 MCP discovery。 |
| 4 | [control-plane.md](control-plane.md) | Registry modules、PostgreSQL ownership、`/api/v1`、reconciliation、idempotency 與 Node Agent protocol。 |
| 5 | [security.md](security.md) | 身份、enrollment、credential lifecycle、server-side authorization、runtime 限制、RAG ACL、secrets 與 audit。 |
| 6 | [development.md](development.md) | Apple Silicon、Bun、Docker Compose、host Ollama、LiteLLM 設定與分階段 smoke tests。 |
| 7 | [roadmap.md](roadmap.md) | M0–M9 的 dependencies、deliverables、non-goals 與 observable acceptance criteria。 |
| 8 | [adr/0001-registry-modular-monolith.md](adr/0001-registry-modular-monolith.md) | Registry-centered modular monolith 的決策及取捨。 |
| 9 | [adr/0002-node-agent-outbound-pull.md](adr/0002-node-agent-outbound-pull.md) | Node Agent outbound pull 的決策及取捨。 |
| 10 | [adr/0003-separate-manifest-and-deployment.md](adr/0003-separate-manifest-and-deployment.md) | 宣告 metadata 與具體 deployment 分離的決策及取捨。 |
| 11 | [adr/0004-litellm-logical-model-aliases.md](adr/0004-litellm-logical-model-aliases.md) | LLM logical aliases 與 backend access boundary 的決策及取捨。 |

## 共用術語

| 術語 | 本文件集定義 |
| --- | --- |
| Plane | 邏輯責任及信任邊界，不代表必須有獨立主機、程序或 Compose project。 |
| Edge plane | 使用者入口、Traefik、Auth/SSO 與 Portal UI；承接身份驗證，後端仍須自行授權。 |
| Control plane | Registry 的 intent、placement、discovery 與 reconciliation；也包含 Orchestrator 的請求協調及 LiteLLM 的模型路由。各元件可獨立執行。 |
| Execution plane | Node Agent、受控 container runtime、實際服務與模型 backend；執行控制面允許的工作並回報觀測。 |
| Data plane | 權威持久資料、ACL metadata、向量與檔案儲存；保留各資料服務的 ownership 與存取邊界。 |
| `Node` | 真實可執行 workload 的主機，由 Node Agent 登錄及回報 inventory。Mac 開發環境只有 `dev-macbook`。 |
| Role | `core`、`gpu`、`worker`、`storage` 等角色標籤；一個 Node 可有多個 role，role 不建立新 Node identity。 |
| `ServiceDefinition` | 可部署服務的宣告 metadata；不包含固定 node、完整 runtime URL、host IP 或直接 host path。 |
| Manifest | `ServiceDefinition` 的版本化宣告輸入；runtime endpoint 與 host-specific binding 在部署時解析。 |
| Service / service instance | Service 是泛稱；描述具體部署或運行狀態時使用 service instance / `Deployment`，不可將其當作 `ServiceDefinition` 的同義詞。 |
| `Deployment` | 特定 `ServiceDefinition` 的具體部署；承載執行位置、resolved bindings、desired/observed/health state 與 generation。 |
| `Capability` | 提供給 AI 或其他服務的能力；可用性由有效 deployment、健康、discovery 與 permission mapping 共同決定。 |
| `AgentCommand` | 控制面發出的有型別、可追蹤執行命令；Node Agent 主動拉取，命令不提供任意 shell execution。 |
| Desired state | 使用者或管理操作要求的 intent：`running` 或 `stopped`。 |
| Observed state | 觀測到的部署狀態：`pending`、`deploying`、`running`、`stopping`、`stopped`、`failed`、`unknown`。 |
| Health state | 與 observed state 分開記錄：`unknown`、`starting`、`healthy`、`unhealthy`；`running` 本身不代表可用。 |
| Generation | 對 deployment intent 的遞增版本；stale generation 的回報不得覆寫新 intent 或使舊狀態重新可用。 |
| Logical model alias | Caller 傳給 LiteLLM 的模型名稱；本機 `home-large` 與 `home-fast` 都指向 host Ollama 的 `phi3:3.8b`。 |
| MCP | 以 Streamable HTTP 暴露的服務介面；宣告 tools、實際 `tools/list` 與完整 permission mapping 必須相符。 |
| Source of truth | 本節政策中的正式架構權威。執行期的權威持久層另有定義：PostgreSQL 自 M2 起承擔此責任。 |

## 全文件固定約束

- 執行位置由 `Deployment` 決定；Manifest 只表達 portable 宣告、placement constraints 與 logical bindings。
- Node Agent 採 outbound pull；每 15 秒 heartbeat，45 秒未收到即視為 offline。Offline、unhealthy 或 discovery/permission mismatch 的 service instance 不可提供可用 capability。
- Portal 與 Orchestrator 只使用 LiteLLM logical aliases，禁止直接連 Ollama、vLLM 或實體 GPU node。
- 本機透過 `OLLAMA_API_BASE` 設定 LiteLLM 到 host Ollama 的連線。Embedding alias 延至 M7，不列為 M0–M3 必要契約。
- PostgreSQL 自 M2 起為權威持久層；Redis 不在 M0–M3 無條件加入。後期的 RAG、向量資料庫、語音與監控能力按 roadmap 交付。
