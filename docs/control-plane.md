# Control Plane

Registry 採 modular monolith，以同一程序中的明確 modules 管理平台 intent、placement 與合法 observations。本文固定 M0–M3 可實作的 REST、交易與 Agent protocol；entity 欄位、state enums 依 [domain model](domain-model.md)，輸入宣告依 [Manifest contract](service-manifest.md)。Orchestrator、LiteLLM 與 Node Agent 不併入 Registry；決策見 [ADR 0001](adr/0001-registry-modular-monolith.md)。

## Modules、持久化與交易

| Module | 權威資料與責任 |
| --- | --- |
| Definitions | 驗證 Manifest、保存不可變 revision、維護 name/version 與 MCP namespace 唯一性。 |
| Deployments | 擁有 desiredState、assignment、generation、resolved bindings；管理 start/stop、restart command 與受控 reconciliation。 |
| Nodes | enrollment、credential digest/revocation、管理員設定的 roles/labels、Agent inventory、有效 heartbeat 與 protocol sequence。 |
| Discovery | 驗證 health、endpoint、MCP discovery 與 permission mapping，產生 per-deployment capability projection。 |
| Authorization / audit | 驗證 principal、resource/action permission、保存 idempotency records 與不可由一般操作改寫的 audit events。 |

PostgreSQL 自 M2 起是上述資料的權威持久層。M0 是契約，M1 可用明確標示的非持久 fixture 驗證介面；M2 之後重新啟動不得遺失 definitions、intent、commands、credential revocation、idempotency 或 audit。Module 只能透過 owner 的介面修改資料；Portal、Orchestrator、Node Agent 不能直接寫 Registry tables。其他服務的聊天或業務資料由各服務擁有，不因使用同一 PostgreSQL instance 就共用寫入權。

接受 mutation 時，在同一 PostgreSQL transaction 中完成 authorization 的必要一致性檢查、deployment row lock／version compare、intent 變更、generation 遞增、舊 availability 撤銷、command 建立或 supersession、idempotency response 及 audit append。唯一約束保護 name/version、namespace 與 idempotency scope。Registry 採 Read Committed 搭配 deployment row lock；一次鎖多筆時按 deploymentId 排序，deadlock 或 serialization conflict 只重試整筆未提交 transaction。PostgreSQL 的 row locks 在 transaction 結束前阻擋衝突寫入；具體平台 mutation 規則由本文定義。[PostgreSQL explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)

Docker、MCP、network probes 均在資料庫交易之外執行。持久 desired state 與 queued commands 就是重試來源；程序在 commit 後 crash，下一輪仍可讀回工作。外部結果入庫前重新鎖定並檢查 generation、ownership、sequence。不能先呼叫 Docker 再假設資料庫會成功。M0–M3 不需要 Redis、外部 message broker 或 distributed transaction；記憶體 wakeup/cache 只加速，遺失後由 PostgreSQL 重建。

## REST 共通契約

以下是完整 `/api/v1` path catalog；未列出的 routes 不屬 v1。Management requests 使用已驗證的 user 或 service principal；Agent routes 使用獨立 enrollment/node credential。[安全模型](security.md)定義身份及授權。所有成功 response 為 `application/json`，除 Manifest registration 外的 request 亦為 JSON。Manifest registration 接受 `application/json` 或 `application/yaml` 的完整 `home-ai/v1` envelope；兩者共用正規化及驗證。

ID 為不透明字串，時間為 UTC RFC 3339。JSON request 拒絕未知欄位與重複 keys；完整 body 限 1 MiB，超過回 413。Collection GET 接受 `limit`（預設 50、1–100）與 opaque `cursor`，以 ID 穩定排序，回 `{items, nextCursor}`，最後一頁 nextCursor 為 null。Cursor 綁定 caller、filters 與 ordering；無效 cursor 回 400。分頁不是跨交易 snapshot，caller 以 ID 去重；授權必須在每頁重新檢查。

Definition GET 回 ServiceDefinition；deployment GET 回 domain model 的 public projection，完全省略 `resolvedBindings`；node GET 回 Node，不含 credential/protocol state；capability GET 回 Capability 加上已驗證的 `inputSchema`（尚無有效 discovery 時為 null），不含 backend endpoint。Management principal 即使是管理員，也不從一般 GET 取得 secret value、host storage 細節或內部 runtime URL。

### Management paths

| Method / path | Request 與授權 | 成功及拒絕條件 |
| --- | --- | --- |
| `POST /api/v1/service-definitions` | 完整 Manifest；`registry:manage`；必須 Idempotency-Key。 | 201 ServiceDefinition 與 Location。相同 name/version 及正規化內容回 200 既有 revision；不同內容 409 `DefinitionRevisionConflict`；validation 422。 |
| `GET /api/v1/service-definitions` | `registry:read`；可選 `name` 精確 filter。 | 200 collection。 |
| `GET /api/v1/service-definitions/{serviceDefinitionId}` | `registry:read`。 | 200 revision；不存在 404。 |
| `POST /api/v1/deployments` | `{serviceDefinitionId, desiredState}`；`registry:manage`；必須 Idempotency-Key。 | 201 Deployment public projection 與 Location；desiredState 必須明確為 running 或 stopped。初始 generation=1、observedGeneration=0、pending/unknown、assignedNodeId=null；非同步 placement。 |
| `GET /api/v1/deployments` | `registry:read`；可選 `serviceDefinitionId`、`assignedNodeId` 精確 filters。 | 200 collection。 |
| `GET /api/v1/deployments/{deploymentId}` | `registry:read`。 | 200 public projection，包含 generation、observedGeneration、stateReason，供查詢收斂。 |
| `PATCH /api/v1/deployments/{deploymentId}` | `{desiredState, expectedGeneration}`；兩欄皆必填，desiredState 僅 running 或 stopped；`registry:manage`；必須 Idempotency-Key。 | 改變 desiredState 回 202 public projection 並遞增 generation、撤銷 availability；前置版本符合且 desiredState 已相同則回 200，不遞增、不新增操作或隱含 restart。離線仍接受 stopped intent，不能聲稱實際停止。 |
| `POST /api/v1/deployments/{deploymentId}/restart` | `{expectedGeneration}`；`registry:manage`；必須 Idempotency-Key。 | 202 AgentCommand；只接受 desiredState=running、已指派且 online 的 Node、有效 reservation 與 resolvedImage，並且沒有 queued/running restart；否則 409 `RestartNotAllowed`。新 command、generation 遞增與 reservation 綁定新 generation 在同一 transaction 提交；action 固定 restart，expiresAt 由有界的 command expiry policy 計算並在 response 明確回傳。 |
| `GET /api/v1/deployments/{deploymentId}/commands` | `registry:read`；共用 pagination。 | 200 該 deployment 的 AgentCommand collection，供追蹤 restart completion，不以 HTTP 202 當作成功執行。 |
| `GET /api/v1/nodes` | `registry:read`。 | 200 collection；只有真實 enrolled Nodes。 |
| `GET /api/v1/nodes/{nodeId}` | `registry:read`。 | 200 Node；roles/labels 不是一般 caller 可寫欄位。 |
| `GET /api/v1/capabilities` | user principal 或帶經驗證 caller delegation 的 Orchestrator；可選 `deploymentId`。 | 200 caller 具備全部 required permissions 的 available capabilities；不包含不可用項目。 |
| `GET /api/v1/capabilities/{capabilityId}` | 同上；逐次重新授權。 | available 且 caller 全部 permissions 通過才回 200；不存在、不可用或無 permission 均回 404，避免洩漏工具目錄。 |

v1 不提供 definition 覆寫／刪除、deployment 刪除、任意 assignment/host binding patch 或任意 command creation API。Definition revision/binding/assignment 的後續管理變更仍須經 owner module 並遞增 generation，不能直接改資料庫。M0–M3 placement 是自動 constraints matching，沒有 caller 指定固定 Node 的捷徑。Enrollment token、roles/labels、secret/storage binding catalog 及 credential revoke/rotation 由受限管理操作維護，沿用 transaction、authorization 與 audit 規則，不增加其他公開 top-level paths。

### Idempotency 與競爭更新

上述 management mutations（POST 與 PATCH）必須帶 `Idempotency-Key`：16–128 個 ASCII letters、digits、`-` 或 `_`，由 caller 為一次意圖產生唯一值。Scope 為 `(principalId, HTTP method, canonical path, key)`，request fingerprint 為正規化 body 的 digest，包含 request 中的 desiredState 與 expectedGeneration。已驗證同 scope/key/body 重送，回相同原 status、body、Location 與 `Idempotency-Replayed: true`；不重新執行，即使目前 generation 已改變。仍先檢查身份與現行授權，不把 cached response 當作權限。相同 key 不同 body 回 409 `IdempotencyConflict`。

第一筆未完成的相同 key 回 409 `RequestInProgress` 與依 retry policy 計算的 Retry-After；client 保留原 key 重試。Mutation 與 response record 原子提交；提交前 crash 可安全重試，commit 後 response 遺失會重播。完整 response 的保留期採 configurable implementation policy，response header `Idempotency-Key-Expires-At` 回傳其 UTC 到期時間；之後保留 scope/key/fingerprint/resource ID tombstone，舊 key 回 409 `IdempotencyKeyExpired`，永不當成新 intent。Client 要新 restart 必須明確產生新 key；timeout 後不得改 key 猜測補發。Validation、authentication、authorization 與未提交的 DB failure 不占用 key。

對新的 deployment PATCH 或 restart POST 請求，expectedGeneration 為必填正整數；與鎖定的當前值不符回 409 `GenerationConflict`，含 currentGeneration。比對後才判斷 no-op。同時兩個不同 key 對同一 generation 改 intent，只有先提交者成功。Start/stop 由 PATCH 將 desiredState 設為 running/stopped，reconciler 可重複套用；restart 是一次性、以 commandId 去重的 POST 操作，不是第三種 desired state。

## Errors

所有 HTTP error 使用 `application/problem+json` 與 [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html) 的 type、title、status、detail、instance；status 與實際 HTTP status 相同。平台 extensions 為 `code`、`requestId`、選用 `errors`（每項 `pointer`、`code`、`message`）與 `currentGeneration`。Type 使用 `urn:home-ai:problem:<kebab-case-code>`；instance 使用不含請求內容的 request UUID URN，不要求新增 error endpoint。訊息不含 stack、raw input、tokens 或 resolvedBindings。

```json
{
  "type": "urn:home-ai:problem:generation-conflict",
  "title": "Deployment generation conflict",
  "status": 409,
  "detail": "讀取目前 deployment 後，以新意圖重新提交。",
  "instance": "urn:uuid:2f035420-311c-4c96-b186-fba75472e488",
  "code": "GenerationConflict",
  "requestId": "2f035420-311c-4c96-b186-fba75472e488",
  "currentGeneration": 4
}
```

| Status | 語意 |
| --- | --- |
| 400 | malformed JSON/YAML、invalid cursor、缺漏或格式錯誤的 Idempotency-Key。 |
| 401 / 403 | credential 無效／過期／撤銷；或 principal 已識別但 action 不允許。不可跨 Node 存取，即使 body 宣稱另一 nodeId。 |
| 404 | resource 不存在；capability unavailable/unauthorized 亦用 404。 |
| 409 | revision、generation、idempotency、sequence 或 command lifecycle conflict。 |
| 413 / 415 / 422 | body 過大、unsupported content type、結構或 semantic validation 失敗。 |
| 429 / 503 | rate limited／權威 DB 暫時不可用；附 Retry-After。不回假成功或以記憶體暫存 intent。 |

## Reconciliation 與 placement

Registry 在 mutation commit 後喚醒 reconciler，另依 configurable implementation policy 週期掃描未收斂 deployments、過期 commands 與失效 observation；喚醒遺失不影響正確性。M0–M3 單一 Registry instance 執行，仍以 row locks 防止 request 與 background worker 競爭。重啟時清除可用快照，Nodes 需新的有效 heartbeat 才恢復 online。權威 DB 不可用時停止派工與 mutation，discovery fail closed。

Placement 在 online Nodes 中同時檢查 roles、labels、architecture、最低 CPU/memory/GPU、logical binding 授權與可達性。使用申報 inventory 減去已保留資源；placement transaction 鎖定候選 Node 及 deployment，保留最低資源，避免超賣。M0–M3 以 nodeId 字典序選第一個符合候選，沒有進階負載最佳化。初次 placement 沒有候選或資源不足，維持 pending/unknown、assignedNodeId=null、stateReason=`AwaitingPlacement`；資源更新時再評估，不標示 failed，也不憑空建立 Node。曾指派的 stopped deployment 再 start 時保留原 assignedNodeId，依下述 gate 等待重新取得 reservation。

初次 assignment 與 binding resolution 一筆交易遞增 generation。對尚未指派且 desiredState=stopped 的新 deployment，Registry 可確認從未派工、沒有 runtime identity 後記錄 stopped/unknown 與 observedGeneration=generation，reason=`NeverScheduled`；這是 absence 證據，不是假造 Agent heartbeat。曾派工的 deployment 只能由其 Node 確認停止；只有 desiredState=stopped 且目前 generation 已確認 stopped／runtime absence 時，才釋放資源 reservation。DesiredState 仍為 running 時，runtime 暫時停止、absence 或 failed 都保留已提交 reservation 供收斂，不能單憑 stopped observation 釋放。Stopped deployment 經 PATCH 再 start 時遞增 generation，重新檢查及保留資源；若原 Node 不足，維持 pending/unknown、AwaitingPlacement、executionAuthorization.runAllowed=false，不自動改派。Agent 即使收到 desiredState=running，也不可 pull/create/start/restart；回報仍然停止的 observation 只確認 runtime absence，Registry 的有效投影保留 pending/AwaitingPlacement，不因此當作已完成 running intent。

Registry 擁有 durable reservation，綁定 reservationId、nodeId、deploymentId 與 generation。只有目前 generation 的 reservation 已在 PostgreSQL committed，且 bindings/image policy 通過，才可在 heartbeat intent 中給予 executionAuthorization.runAllowed=true。取得資源可以在稍後交易完成，但必須重新鎖定並比對目前 generation；交易 commit 前不得回傳授權。任何 generation 變更都讓舊授權失效；仍在使用的 reservation 可在同一 intent transaction 重新綁定新 generation，不能先釋放造成超賣。DesiredState=stopped 時 runAllowed=false，保留舊資源占用至目前 generation 確認停止；沒有 reservation 不妨礙依目前 stopped intent 執行受控 stop，這條停止路徑仍須 fresh intent 與 Docker ownership，不能用來 restart。

Agent 比較當前 desired state、executionAuthorization 與受控 Docker observation：running 且 absence 時，只有當前 generation 的 runAllowed=true、reservationId 非空及 resolvedImage 有效才可建立／啟動；缺任一 gate 保持 pending 並只 inspect/report。stopped 且存在時依停止路徑收斂；已符合則只 report。新 generation 不一定要重建 container，但舊 health/discovery 一律失效並重新取得。新 restart generation 帶 active command 時，還須通過相同 run gate 與 command acceptance，再走一次性 command journal，不得讓一般 running reconciliation 額外觸發第二次 restart。

Image pull、Docker create/start/stop 失敗回 failed 與受控 RuntimeError；同 generation 的自動 runtime 重試採有界次數與 backoff 的 configurable implementation policy，超限保持 failed，等新 intent 或管理員處理。Resource shortage 保持 pending，不消耗 runtime retry 次數；health unhealthy 只撤銷 capability，M0–M3 不自動 restart。Start 已為 running 是 no-op，不藉此繞過重試上限；需要再次執行時明確 stop/start 或符合條件的新 restart。離線不等於 stopped，不釋放 reservation、不跨 Node failover。

## Agent outbound pull protocol

Agent 是 host daemon，只主動連向 Registry 的受限 endpoint；Registry 不開 SSH/inbound Agent RPC。Heartbeat 每 15 秒送出，server 以最後一次有效且非重送 heartbeat 的接收時間，距今達 45 秒即 offline。判斷使用 server time，不能讓 Agent 提供的未來時間延長 online。Registry 每次讀取／派工／discovery 都套用 45 秒有效性 gate，不等待背景掃描才撤銷權限。[ADR 0002](adr/0002-node-agent-outbound-pull.md)記錄取捨。

| Method / path | Request | Response 與語意 |
| --- | --- | --- |
| `POST /api/v1/agent/enroll` | `{enrollmentToken, enrollmentId, nodeCredential, name, inventory}`；無既有 node credential；token 綁定管理員允許的 identity/roles/labels。 | 201 `{nodeId, credentialExpiresAt, heartbeatIntervalSeconds:15, offlineAfterSeconds:45}`。Agent 在本機預先生成並持久保存高熵 nodeCredential；Registry 僅存 digest，不回傳 credential。相同 token/enrollmentId/body 的短期重送回原 response，見安全模型。 |
| `POST /api/v1/agent/heartbeat` | Node bearer credential；`{nodeId, bootSequence, heartbeatSequence, inventory, deployments, commandProgress}`。 | 200 `{serverTime, acceptedHeartbeatSequence, desiredDeployments, commands, rejectedReports, rejectedCommands}`；以同一受限 response 拉取此 Node 全部 current assignments 與非 terminal、未過期 restart commands。沒有工作或拒絕項目時對應 arrays 為空；空 assignments 列表本身不是停止／刪除未列 container 的授權。 |
| `POST /api/v1/agent/commands/{commandId}/result` | Node bearer credential；`{nodeId, bootSequence, deploymentId, generation, status, result, error}`；status 僅 succeeded 或 failed。 | 200 保存的 AgentCommand；須 command 屬於此 Node/current generation，且 lifecycle 合法。相同 terminal payload 重送回 200 原結果；不同結果或已 expired 回 409 `CommandTerminal`，只留 audit。 |

Agent protocol 不使用 management Idempotency-Key：enrollment 以 token/enrollmentId、heartbeat 以 boot/heartbeatSequence、command result 以 commandId 加固定 payload 去重。Heartbeat 先驗證 envelope（可解析的 body、root 欄位／型別、inventory、deployments/commandProgress arrays）、credential、nodeId ownership、boot session 與 heartbeatSequence；任一失敗則整個 request 拒絕，不更新 lastHeartbeatAt。Body 中 nodeId 必須與 credential owner 相同；重送 heartbeat 也不延長 online。

Envelope/node-session 驗證通過且 heartbeatSequence 是新值時，heartbeat transaction 以 server 接收時間更新 lastHeartbeatAt。Deployment observations 與 commandProgress 的 item schema、ownership、generation、reportSequence/lifecycle 則各自驗證：即使全部 items 都 stale／rejected，合法 heartbeat 仍更新 Node liveness 並取得最新 intent，不能把「舊 observation 不可信」誤當「Node 沒有連線」。Rejected item 不更新 deployment/command projection 或健康證據；只有有效 items 的結果可以入庫。Transaction 失敗（例如 DB/audit 寫入失敗）不提交 liveness 或任何 item 更新。

`bootSequence` 是每次 Agent 程序啟動前在本機 durable journal 遞增的正整數。Registry 持久保存最高值；第一個合法 heartbeat 的更高 bootSequence 建立新 session、撤銷舊 session 的 observations/availability，較低值回 409 `StaleAgentSession`。同一 Node 只允許一個 Agent 程序，本機 process lock 防止雙啟動。Journal 遺失時不能重設 counter 猜值，需管理員撤銷舊 credential 後重新 enrollment。Result endpoint 不能建立新 session。

HeartbeatSequence 在每個 bootSequence 由 1 單調遞增。Registry 對相同 sequence/body 回原 response 與 `Heartbeat-Replayed: true`，較小 sequence 或同 sequence 不同 body 回 409 `ReportOrderConflict`；已接受 sequence 與 fingerprint 持久保存。Agent 必須辨識 replayed response，不用其舊 desired state 開始 runtime mutation；發送新 sequence 取得最新快照。

每項 `deployments` observation 是 `{deploymentId, generation, reportSequence, observedState, healthStatus, stateReason}`。ReportSequence 在同 bootSequence、deploymentId 中遞增；只接受 assignedNodeId 相同、generation 恰等於 current generation 的 observation。不同 item 獨立驗證：不合法 item 不更新 projection，放入 `rejectedReports`，每項包含原 array index、deploymentId（無法解析時為 null）、code；其他有效 items 與 Node liveness 可提交。Stale generation 記錄診斷，不更新 observedGeneration、health 或 capability。未知的更高 generation 也拒絕。同 generation 倒序 report 不得讓 stopped/unhealthy 回到舊 running/healthy。

每項 `commandProgress` 為 `{commandId, deploymentId, generation, status:"running"}`，在實際 restart 之前送出；Registry 原子驗證未過期、queued/running、assignment、generation 與當前 run gate，接受後才把 command 標為 running、填 startedAt。拒絕項目放入 `rejectedCommands`，每項包含原 array index、commandId（無法解析時為 null）、code；Agent 不執行該 command，但不影響合法 heartbeat 的 Node liveness 更新。Heartbeat response 的 desiredDeployments 為受限 Agent intent，包含 deploymentId、serviceDefinitionId、validated manifest、assignedNodeId、generation、desiredState、必要的 resolvedBindings、executionAuthorization、resolvedImage 與 `activeCommandId`（無則 null）。Secret bindings 僅 reference，不是 value；只有被授權 Node 可透過本機預先佈署的 secret binding 取得值。

`executionAuthorization` 固定為 `{generation, reservationId, runAllowed}`：generation 必須等於同一 intent 的 generation，reservationId 是已提交且綁同 node/deployment/generation 的不透明 ID，尚未取得時為 null；runAllowed 是 Registry 計算的 boolean。只有 desiredState=running、Node online、current reservation 已提交且 bindings/resolvedImage 通過 policy 才為 true。Agent 不得從 desiredState、過去 reservation 或 command presence 自行推定此值；false／缺欄位／版本不符都拒絕 pull/create/start/restart。Registry 接受 running/healthy report 與建立 capability 時也重新檢查此 gate，未授權啟動的 observation 只能作異常診斷。

`resolvedImage` 為 null 或 `{kind, identity}`，獨立於 domain model 的 resolvedBindings；後者維持僅有 endpoints/storage/secrets。Registry 在 deployment resolution 依 image policy 核准並持久保存每個 generation 的 image identity：kind=`registry-digest` 時 identity 是完整 repository@sha256 digest reference，kind=`local-image-id` 時 identity 是該 assigned Node 上經管理員核准的完整 Docker image ID。尚未解析時為 null 且 runAllowed=false。相同 generation 不可重新解析 tag 來改 identity；改用另一 image 必須經新的受控 generation mutation。Agent pull/create 只用 resolvedImage.identity；不得自行重解 Manifest 的 mutable tag，local-image-id 不存在時直接回受控 RuntimeError，不退回 tag 拉取。Agent start/restart 既有 container 前也須核對其 image 與此 identity 的解析紀錄一致，不符則拒絕啟動並交由受控 replacement 收斂。這兩個欄位是受限 Agent protocol metadata，不新增 public Deployment 欄位或 API family。

Agent 在每個新的 Docker mutation 前另發新 sequence heartbeat，重新確認當前 assignment/generation；pull/create/start/restart 額外確認 executionAuthorization、resolvedImage 與適用的 command acceptance。Stop 依現行 stopped／replacement intent，或下述已接受 active restart command 的 stop 子步驟授權，並驗證 ownership；remove 仍僅限目前 intent 明確取代的已停止受控 container。快照的有界有效窗口由 configurable implementation policy 定義，自本機 monotonic send time 起計算，逾時就再拉一次。固定 15 秒 heartbeat 照常，長操作不阻塞 heartbeat。失聯、401、503、過期快照或 generation 被取代時，不開始新的 mutation；既有 container 可繼續運行，先完成已開始且無法取消的 runtime operation，重連後回報並收斂。

這是 generation fencing，不承諾跨 HTTP、資料庫與 Docker 的原子 exactly-once：stop 可能在最後一次 fresh heartbeat 後到達。Registry 立即撤銷 availability，拒絕舊結果改寫當前 intent；Agent 下一次取得新 generation 必須收斂到 stopped。執行中的 restart 即使 command expired 也不能被當作已取消。

### Restart journal 與觀測新鮮度

Restart 以同一 command 的 stop/start 子步驟執行，全程 desiredState=running。Stop 前的 fresh heartbeat 必須確認 commandProgress 已接受、command.status=running 且未過期、activeCommandId=commandId，commandId/deploymentId/current generation 與本機 journal 一致，assignment 仍屬此 Node，並有同 generation 的 executionAuthorization.runAllowed=true、已提交且非空 reservationId、有效 resolvedImage 及相符的 container image。通過 Docker ownership 與 durable journal 檢查後，這個 active accepted restart command 才授權 command-bound stop，不需要將 desiredState 改為 stopped；queued command 或一般 running intent 都不授權此 stop。

Restart 中途回報的 stopping／stopped 是同一 command 的 execution phase，沿用既有 observedState 值，不新增 state enum；health 為 unknown。Registry 可接受 current generation observation，但不把它當作 desiredState=stopped、running intent 已收斂或 command succeeded，也不釋放 reservation。中途 heartbeat response 繼續提供 current running intent、仍有效的 reservation/runAllowed 與同一 active command；一般 reconciler 不搶先啟動，start 子步驟仍由該 command journal 控制。

Start 前必須另取新 sequence 的 fresh heartbeat，重新通過上述 command、generation、runAllowed、reservation、resolvedImage 及 ownership gates，不能沿用 stop 前的授權。Stop 後失聯時不開始 start；重連後只有 current command/generation 仍有效、且 journal 能證明 stop 已完成而 start 尚未開始，才接續同一 command 的 start 子步驟，不重做 stop。已被新 intent 取代或已 terminal／expired 的 command 不續行；依 current intent 與下述 journal 不確定結果規則收斂。

Agent 將 commandId、deploymentId、generation 與 execution phase 在 Docker mutation 前 durable 寫入本機 journal。相同 commandId 重送，只回既有 progress/result；不能重新 restart。Crash 後若已保存 terminal result 就重送；若 journal 顯示已開始但不能證明結果，回 failed / `IndeterminateExecution`，inspect 實際 runtime 再 report，不能以新 commandId 或一般 reconciler 自動重做同一 restart。Command succeeded 表示 runtime restart 操作完成，仍須新 report、healthy probes 與 MCP discovery 才可 available。

Agent 依 Manifest health 設定持續 probe；每份 health report 必須來自當前 generation。Health evidence 與 deployment report freshness 採 configurable implementation policy，必須有界、與 probe interval/timeout 相容；probe 中斷或可信證據過期時，Agent／Registry 將 healthStatus 設 unknown 並撤銷 capability。有 Node heartbeat 但缺 deployment report 不延長服務健康。Offline 立即使該 Node deployments 的有效 projection 為 unknown/unknown、reason=NodeOffline，保留最後 observedGeneration 作診斷。

## Discovery 與服務存取

Registry 從 assigned Node 的受控 network identity、validated named port/path 解析 endpoint，不能信任 Agent 自報 arbitrary URL，也不能使用 Manifest 的固定 IP。Public Web/API route 經 Traefik 與 authentication；Portal 只使用 Edge route。Backend discovery 僅供已授權的 Orchestrator／服務呼叫者；同樣不得從一般 capability response 暴露 Docker 或 host 管理介面。

Capability 須符合 current intent/generation、online Node、running 且 healthy，以及 Manifest allowed tools、完整實際 tools/list、完整 permission mapping。MCP 是 Streamable HTTP；初始化與 protocol revision 規則依 [Manifest contract](service-manifest.md#mcp-discovery-與-permissions)。Discovery snapshot 有效期限、refresh 與無通知 server 的重驗時機採 configurable implementation policy，需有界並重新列舉所有分頁；正在 refresh 不能延長舊快照。失敗、tools/list_changed、permission catalog 改變、generation 變更、unhealthy/offline 或 report 過期立即撤銷整個 deployment 的 MCP capabilities。這些 gate 在每次查詢及 invocation 前檢查，不只依賴 cache eviction timer。

MCP/Orchestrator 實際整合是後期 milestone；M0–M3 固定 contracts 並使用 deterministic fixtures 驗證 fail-closed gates，不要求部署整套 MCP、RAG 或模型服務。當它們啟用時，仍使用相同 ownership、authorization 與 audit 邊界。

## Lifecycle failure walkthrough

| Scenario | 可觀測結果與恢復方式 |
| --- | --- |
| Registration → healthy | Definition 201；Deployment 201 初始 pending；placement 新 generation；Agent heartbeat 拉取並回 deploying/running；新 probe healthy；MCP 完整 gate 成功後 capability GET 才可見。 |
| 資源不足 | pending / AwaitingPlacement、無新 runtime mutation；inventory 或 reservation 變更後重評；不建立虛構 gpu Node。 |
| Stopped → 資源被占用 → start | 確認 stopped 後釋放 reservation；其他 deployment 取得資源；PATCH desiredState=running 產生新 generation，但原 Node 資源不足，故 pending/AwaitingPlacement、reservationId=null、runAllowed=false。Heartbeat 仍傳 running intent，Agent 只 inspect/report，不 pull/create/start/restart；待 current generation reservation committed 且 resolvedImage 有效，後續 fresh heartbeat 授權才啟動。 |
| Heartbeat timeout | 自最後有效 heartbeat 達 45 秒，online=false、unknown/unknown、NodeOffline、capability 不可用；不宣稱 container 停止、不自動 failover。 |
| 合法 heartbeat 含 stale report | Credential/node/session 與新 heartbeatSequence 有效，lastHeartbeatAt 更新、Node online；stale item 列入 rejectedReports，不改 observedGeneration/health/capability。Response 仍提供最新 intent，讓 Agent 收斂；整個 envelope 無效或 heartbeat 重送則不延長 liveness。 |
| Stop 與舊 running report 競爭 | Stop 遞增 generation 並撤銷 availability；舊 report 被拒絕；Node 重連取得 stopped intent，確認 absence 後才 stopped。 |
| 同代倒序／舊 Agent session | reportSequence 或 bootSequence 不符合即拒絕；不能把較新的 unhealthy/stopped 覆蓋成 healthy/running。 |
| Start/stop 重送 | 同 key 回原 response；新 key 配當前 generation 且 desired 已相同則 no-op；HTTP timeout 不新增 generation。 |
| Restart 重送／crash | 同 key 同 commandId、Agent journal 去重；不確定已執行結果為 IndeterminateExecution；不再次 restart。 |
| Restart 中途 heartbeat／失聯 | running deployment 接受 restart，generation 由 4 升至 5，reservation 同交易綁定 5 並撤銷 capability。Agent 以 commandId/deploymentId/5 送 commandProgress，fresh response 接受 running command 並通過 run/image/ownership gates 後 journal 記錄並執行 stop。中途新 heartbeat 回報 generation=5、observedState=stopped、healthStatus=unknown；Registry 保持 desiredState=running、同一 active command 及 reservation，不視為完成。若此時失聯，start 不開始、reservation 仍保留；重連以新 sequence heartbeat 確認同 command/5 仍有效且 journal 證明 start 未開始，重新通過 gates 才 journal 記錄並 start。若先收到新的 stopped generation，改依該 intent 確認 stopped/absence 才釋放。Restart succeeded 後 capability 仍撤銷，直到 generation=5 的新 running report、fresh healthy probes 與完整 MCP discovery 全部通過。 |
| Restart 被 stop 取代／過期 | 舊 command 變 failed / StaleGeneration，或到期 expired；遲到結果只 audit；新 stopped intent 優先收斂。 |
| Docker／image failure | failed / RuntimeError、capability 不可用；受限退避重試，超限等待新 intent／管理處理。 |
| Unhealthy／discovery mismatch | 健康失敗可保持 observed running，但 unavailable；tools 缺漏、額外或 permission mismatch 撤銷整份快照；恢復需新健康與完整 discovery。 |
| Registry／DB 重啟或斷線 | 已提交 intent/commands 保留，拒絕新派工直到權威資料可讀；Node 取得新快照後恢復；不憑 heartbeat 或 command success 推定 healthy。 |
| Credential 撤銷 | Agent routes 401，Node 立即不可派工且 capability 撤銷；既有 runtime 需受信任的本機管理操作隔離，詳見安全模型。 |
