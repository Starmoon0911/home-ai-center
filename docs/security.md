# Security model

家庭內網不是授權邊界。所有 client input、Manifest、模型輸出、tool arguments 與 service responses 都需驗證；使用者 identity、service identity、Node credential 與 runtime ownership 分開處理。本文是 [Control Plane](control-plane.md) 的安全契約，沿用 [architecture](architecture.md#trust-boundaries) 的信任邊界。

## Identity 與 trust boundaries

| Boundary | 強制控制 |
| --- | --- |
| Browser → Edge | 使用者外部存取經 Traefik 與 authentication；服務 backend、PostgreSQL、Docker API、Ollama/vLLM 不直接暴露。Browser 不持有 Node、LiteLLM 或 backend service credential。 |
| Edge → Registry／Orchestrator | Backend 驗證受信任 identity，再以 resource/action 授權；不接受 client 可自行填寫的 identity headers。Edge 清除 client identity headers，只由受信任 adapter 建立 principal。 |
| Agent → Registry | TLS server verification 加獨立 Node bearer credential；每個 request 重新檢查到期／撤銷、nodeId ownership、session、generation。Node credential 不能呼叫 management APIs。 |
| Agent → Docker | Agent 是具 host 控制風險的 daemon；使用受限操作 allowlist、平台 labels 與 durable local runtime mapping，不能代替使用者執行任意 shell。 |
| Orchestrator → MCP／資料 | 每次 tool call 授權、arguments validation、service-side authorization；RAG retrieval 強制 owner/scope/ACL。模型輸出沒有授權效力。 |

M0 不啟動服務；M1–M3 開發可由明確的 dev authentication adapter 使用環境配置的個別 bearer credentials，映射穩定 principalId 與 server-side grants。Digest 與 grants 分開保存；不能有通用 anonymous admin 或以 LAN IP 免驗證。僅綁 loopback 的開發入口可用 HTTP；任何跨主機、LAN/VPN 或外部路徑都必須 HTTPS 並驗證 server certificate，不能以 production mTLS 延後為由關閉 TLS。瀏覽器後期採用 session cookie 時必須 HttpOnly、Secure、SameSite 與 CSRF 防護，不把長期 bootstrap token 存 localStorage。

M0–M3 固定 permission catalog 至少 `registry:read` 與 `registry:manage`：manage 明確包含 read；Node 不取得兩者。MCP permission keys 由管理員登記，依 Manifest `service-name:action` 格式且全部 permissions 採 AND。Grants 綁 principal，可由受限管理設定撤銷；每次 request 讀取現行有效授權，改權限立即使相關 discovery/delegation cache 失效。Caller 無權直接寫自己的 roles、grants、Node labels 或 ACL。

## Enrollment、rotation 與 revoke

管理員以受限的本機管理流程建立一次性 enrollment token，至少 256 bits cryptographic randomness，以 configurable implementation policy 設定短期 expiresAt；只允許一個指定 Node identity，並固定允許的 name、roles/labels 或既有 nodeId。Token 與 node credential 在 Registry 僅保存帶 server-side key 的 digest、credentialId、owner、expiresAt、revokedAt 等 metadata；raw values 不進 PostgreSQL、Git、logs 或 audit。Token 交付走受信任通道，不放 URL／query string。

Agent 在本機生成另一個至少 256 bits 隨機 nodeCredential，先以 owner-only 權限持久保存，再透過 HTTPS 的 `POST /api/v1/agent/enroll` 送出 enrollmentToken、唯一 enrollmentId、nodeCredential、name 與 inventory。Enrollment transaction 驗證 token 的範圍與到期、唯一 enrollmentId、credential digest，原子消耗 token、建立或重新綁定 Node，保存無 secret 的 response 與 audit；Node 尚未 heartbeat 時 online=false。Registry 配發不重用 nodeId，角色不是由 inventory 自我授權。

為處理 response 遺失，消耗後的 token 只允許在原 expiresAt 之前，以同 enrollmentId 及相同 body fingerprint 重取原本 nodeId/expiry response；不再次建立 Node、延長期限或回傳 secret。Fingerprint 使用 keyed digest，不保存 raw request。其他重用／到期 token 401。Node credential 有效期限及到期前 rotation 窗口採 configurable implementation policy；Registry 在 enrollment response 明確回傳 credentialExpiresAt，Agent 在期滿前啟動管理員核准的 rotation 流程，期滿後不能默默延用。

Rotation 由管理員簽發綁定既有 nodeId 的新一次性 token，Agent 先 durable 保存新 credential，再重新 enrollment。在同一交易啟用新 credential、撤銷舊 credential、清除舊 protocol session 與 availability；沒有兩組永久共用 credential 或無期限 grace period。Response 遺失按原 enrollmentId 重試；新 Agent heartbeat 及 observations/discovery 通過後才恢復可用。Registry 拒絕未知 token 對既有 Node name 進行接管；一般 enrollment 不可自行指定已存在的 nodeId。

Revoke 是管理員對 credentialId 或整個 nodeId 的受限內部操作，必須 audit 並立即阻止該 Node 的 heartbeat/pull/result 與所有後續派工；Node 有效投影立刻 offline，capabilities 撤銷，未完成 command 不再派送。使用者 bearer 或 MCP service credential 的撤銷也必須在各 backend 生效，不能只登出 Portal。遺失本機 journal、懷疑 credential 洩漏或 host 被接管時，撤銷後由管理員確認主機、隔離舊 runtime，再對受信任 Agent 重新 enrollment。

Credential revoke 無法遠端保證停止已失聯／已遭接管主機上的 container。平台不把 revoked 當成 stopped，不直接釋放資源或自動另起副本；Edge／discovery 阻斷服務入口，由受信任的本機管理、network isolation 或主機重建完成處置。控制面不因此取得未授權的 SSH 能力。

## Docker execution policy

Docker daemon 的管理權限可能導致 host privilege，因此 Node Agent 與 daemon socket 必須隔離於 workload 與公開網路；rootless Docker 在環境支援時可縮小風險，但不能取代平台 authorization。[Docker Engine security](https://docs.docker.com/engine/security/)

v1 runtime 僅 `docker`。Agent 直接使用 Docker API／結構化參數，不拼接 shell；allowlist 只有對核准 image 的 pull、受控 container create/inspect/start/stop/remove，以及以 stop/start 實現的一次性 restart。Remove 僅針對已停止、平台擁有且被目前 intent 明確取代的 container，不刪 volume 或 host files。沒有 exec、任意 command/script、build、system prune、host process kill、任意 network 或 volume delete API。

Image policy 在 deployment resolution 檢查管理員允許的 registry/repository，將 tag 解析並固定 image digest；本機 M3 `hello-service` 可使用管理員核准的 local image ID，不能讓遠端 caller 任意拉取 image。Registry 將核准結果持久綁定 generation，透過受限 Agent intent 的 `resolvedImage: {kind, identity}` 傳遞完整 repository@sha256 digest reference 或該 Node 的 approved local image ID；Agent pull/create 只使用這個 identity，不重新解析 Manifest mutable tag。Local image 不存在或既有 container image 不符時拒絕啟動，不能回退到 tag。`resolvedImage` 是獨立 protocol 欄位，不增加 domain resolvedBindings 的 endpoints/storage/secrets keys。

Runtime 由已驗證 Manifest、resolvedImage 及受限 bindings 生成：禁止 privileged、host network、Docker socket mounts、host PID/IPC、arbitrary flags、額外 host capabilities 及 unmanaged host mounts；logical storage target 仍須通過 canonical path／敏感目錄／overlap 檢查，詳見 [Manifest storage contract](service-manifest.md#routesplacement-與-bindings)。CPU/memory limits 與 GPU device allocation 由允許的 placement/bindings 解析，不直接傳遞使用者提供的 Docker HostConfig。任何 pull/create/start/restart 另須目前 generation 的 `executionAuthorization.runAllowed=true` 與非空 reservationId；Registry 只有在 current reservation committed 且 resolution/policy 通過才授權。`desiredState=running` 或 queued command 本身不授權執行；stopped intent 的受控 stop 不要求重新保留已釋放的資源。

每個受控 container 必須同時有以下 reserved labels，由 Agent 建立，Manifest annotations 不能覆寫：

| Label | 值與用途 |
| --- | --- |
| `home-ai.managed` | 固定 `true`，表示受控 runtime。 |
| `home-ai.node-id` | 目前 authenticated Node 的 nodeId。 |
| `home-ai.deployment-id` | 目前 assignment 的 deploymentId。 |
| `home-ai.service-definition-id` | 驗證過的 immutable revision ID。 |
| `home-ai.generation` | 建立此 container 時的 generation 整數字串。 |

Agent 在 create 前驗證 assignment；每次 inspect/start/stop/restart/remove 前，以本機 durable deployment-to-container-ID mapping 配合全部 labels 核對 ownership。Container name、Compose profile、單一 managed label 或字串 prefix 都不足以證明歸屬。Homebrew Ollama、平台 Compose core containers 與其他 unmanaged containers 不在 lifecycle 管理範圍。Docker object labels 是識別 metadata，並非可防惡意 host 管理員偽造的安全邊界；平台仍信任經 enrollment 的主機管理者。[Docker object labels](https://docs.docker.com/engine/manage-resources/labels/)

現行 generation 的 stopped／replacement intent 可以停止此 deployment 在同 Node 的較舊 container：要求 local mapping、nodeId/deploymentId/definition provenance 全部符合，label generation 不得高於當前 intent；不能以「舊 label」否定必要的清理，也不能跨 Node 操作。Docker labels 在 container 建立時固定，保留原 creation generation，不假裝每次 observation 都改 label；Agent journal 另記 applied generation。若 definition 被更新，只允許清理 journal 中已知的前 revision，不能 adopt 任意相似 container。

Mapping 遺失、labels 不符、重複 container identity 或未知的較高 generation 都回 `OwnershipMismatch`，停止 mutation 並 audit，交由管理員確認，不自動接管或刪除。Agent 安裝與 Docker daemon hardening 是主機管理責任，不開放平台 caller 修改 Agent binaries 或 credential files。

## MCP permission enforcement

Registry discovery 使用 [Manifest](service-manifest.md#mcp-discovery-與-permissions) 固定的 Streamable HTTP 與 protocol revision，僅從已驗證 deployment endpoint 初始化並完整讀完 tools/list 分頁。Allowed tools、實際集合、每個 tool 的完整有效 permission mapping 與 healthy 四重 gate 必須同時通過，且 Node/intent/generation 新鮮。任何缺漏、額外 tools 或未知 mapping 讓整個 deployment 的 MCP capabilities unavailable；不能靠 Portal 隱藏按鈕補救。

Orchestrator 在提供 tools 給模型前，依使用者目前 grants 過濾；在每次 `tools/call` 前重新查 availability、current generation、全部 required permissions，並用目前 discovery 的 inputSchema 驗證 arguments。呼叫時只帶目的 MCP service 專用的短效、audience-bound delegation，包含經驗證 user identity、scope、到期時間與 trace context；不把 Node credential、LiteLLM key 或其他 resource token 轉送給 MCP。Service 端再次驗證 delegation audience/expiry、permission 與 resource scope；只有 Orchestrator 的 shared service identity 而沒有 user context，不能執行 user-scoped tools。Token passthrough 與錯誤 audience 會破壞信任邊界，MCP 官方安全指引也明確要求防範此模式。[MCP security best practices](https://modelcontextprotocol.io/docs/2025-11-25/tutorials/security/security_best_practices)

MCP metadata、annotations、description、tool result、LLM 所提議的 tool name/arguments 都是不可信資料，不會建立 permission grant。Server 的 Streamable HTTP endpoint 須驗證 Origin、authentication 與 session；session ID 不替代 authorization。Network policy 限制可連目的地，discovery/probe 不跟隨跨 endpoint redirect；不能利用自報 URL 觸及 metadata service 或其他內部 admin API。實際 delegation token 格式與 issuer adapter 在 MCP milestone 實作；這些強制驗證邊界不能因產品選擇而省略。

## RAG ACL 邊界

RAG 於 M7 交付；M0–M3 不需要 Qdrant 或 embedding alias。現在固定原始文件、chunks、embeddings 與 metadata 必須連回不可混淆的 documentId、ownerId、scopeId、ACL revision；向量 index 是衍生資料，不是 permission authority。

Ingestion 先檢查 caller 對目的 scope 的寫入權，所有 chunks 繼承文件 ACL。Retrieval 由 server 根據 authenticated principal 計算可用 scopes，將 ACL filter 套在向量／全文查詢層，再於內容載入前以權威 ACL 檢查候選；未授權內容不得進入 reranker、prompt、citation、cache 或 model response。不能先全庫 top-k、讓模型自行隱藏。Client／模型提供的 scope、documentId 或 metadata filter 只能縮小權限，不能擴大。

ACL 撤銷、文件刪除或搬移後立即失效相關 cache；index 尚未同步時以權威 ACL fail closed。Cache key 包含 principal／grants、scope 與 ACL revision，不能跨使用者分享含私密內容的 retrieval 結果。後期整合驗收必須包含兩個使用者／兩個 scopes、直接 document ID、向量 search、citations、cache 與 ACL 撤銷後重試的負向測試。Qdrant 的 collection/payload layout 與 embedding 選型留給 M7，不能改變 retrieval 層強制 ACL 的責任。

## Secret policy 與 audit

Manifest、examples、API errors、一般 response、logs、traces、audit、Git 與測試 fixtures 不得含真實 secret、token、model credential 或 private document content。Manifest 只包含 logical secret ref；resolvedBindings.secrets 也是受限 store reference。M0–M3 由管理員在授權 Node 預先提供 owner-only 的 secret binding store，Agent 執行時解析 ref 並注入指定 env；不存在或未授權 binding 時拒絕啟動，不讀任意 host path 或回傳 secret。ENV 可能被具 Docker 管理權的人讀取，因此 host 管理者在此信任模型內；API 和一般 support bundles 仍不輸出 ENV。

Credential 使用最小 scope、明確 expiresAt 與可撤銷 identity。Secret rotation 更新受限 binding revision，Registry 遞增受影響 deployment generation、撤銷 availability，經受控重建／重新注入後才恢復；不靠改 Manifest 存值。備份同樣需存取控制與加密，restore 後保留 revocation/idempotency/audit 語意，不復活已撤銷 credentials 或重播 restart。自動 log redaction 覆蓋 Authorization、cookies、enrollment body、secret env 及已知敏感欄位，不能只靠開發者小心。

M2 起 audit 為 PostgreSQL durable append-only event contract，一般 application 操作沒有 update/delete audit 的權限。每筆 event 至少有 eventId、occurredAt（server UTC）、requestId、actorType/actorId、action、resourceType/resourceId、outcome、reasonCode；適用時附 nodeId、deploymentId、generation、commandId 與變更前後的非敏感 state。只保存 Idempotency-Key digest，不記 token/raw arguments/raw Manifest/private file content。

必記錄 registration accept/reject、deployment intent/placement、start/stop/restart、command terminal／expiry、stale/ownership 拒絕、enrollment/token consumption、credential rotate/revoke、permission/ACL 變更、MCP discovery mismatch、tool invocation allow/deny/result。大量有效 heartbeat 僅更新狀態，online/offline 轉換與拒絕事件才 append audit；避免每 15 秒保存完整 inventory。Tool invocation audit 記 tool key、scope ID、結果分類與 latency，不記任意 arguments 或 output。

接受的管理 mutation、enrollment、command/state 轉換與 audit 必須同 transaction 提交；audit 寫入失敗即 rollback，不回成功。外部 tool call 採先持久記錄 intent/authorization decision，再呼叫，之後記錄 outcome；結果未知需可追蹤，不能宣稱 exactly-once。DB 故障時不放行依賴 audit 的新 privileged action；安全拒絕仍立即生效，另輸出已 redacted 的本機 security log 供復原後補記。Audit 讀取／匯出僅限受信任管理流程，M0–M3 不提供公開 audit route，也沒有由普通 API 清空 audit 的操作。

## 後期產品邊界與驗收

| 項目 | M0–M3 邊界 | 啟用時的可觀測驗收 |
| --- | --- | --- |
| Authentik | 延後正式 SSO，保留 authentication adapter、穩定 principalId 與 server-side grants。 | 偽造 identity header、失效 session、無 permission API 與 revoked grant 均被 backend 拒絕；SSO 不替代 object authorization。 |
| Qdrant／RAG | 延至 M7；不依賴向量資料庫才能完成 enrollment/lifecycle。 | 未授權 chunks/citations 不進 prompt，ACL 撤銷對 cache 及 retrieval 即時有效。 |
| 完整 observability | 延後 metrics/traces/dashboard 產品；M2 durable audit、requestId、redacted errors 不延後。 | 可追蹤 request → intent → command → observation，logs/traces 不含 secrets。 |
| Production mTLS | 延後 certificate automation／service mesh；跨主機 TLS server verification、per-node credential、revoke 現在固定。 | 換成 mTLS 時 certificate identity 綁同 nodeId，撤銷／rotation／跨 Node 拒絕測試仍通過。 |

M3 安全驗收至少覆蓋：single-use enrollment、撤銷 credential、跨 Node report/result、stale generation／session／sequence、Docker ownership mismatch、非法 runtime/host mount、secret redaction 及 audit rollback。MCP 與 RAG 的實際整合負向測試在各自 milestone 執行，契約與 fail-closed 邊界從現在保持一致。
