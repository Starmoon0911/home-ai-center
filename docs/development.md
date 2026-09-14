# 本機開發與 smoke tests

本文件定義 Apple Silicon Mac 的開發路徑，交付順序見 [roadmap](roadmap.md)。目前是 M0 文件交付；下列 M1–M5 指令、測試入口與設定檔是各 milestone 實作時必須提供的開發契約，並非宣稱 repository 已有可執行服務。M0 只做文件靜態驗證，不下載模型、不啟動 Compose、不執行 inference；`phi3:3.8b` 真實 chat/streaming 是 M5 gate。

## Prerequisites 與程序位置

| 起始階段 | Prerequisites | 用途與限制 |
| --- | --- | --- |
| M0 | Git、Markdown/YAML/JSON/TypeScript/Mermaid 靜態檢查工具。 | 閱讀與驗證正式契約；不要求 Docker、Bun runtime 或模型服務運行。 |
| M1 | Apple Silicon Mac、原生 arm64 Bun、Git。 | Bun 管理 workspace dependencies、TypeScript 程序與測試；M1 固定實際工具版本及 lockfile，CI 採相同版本。安裝及版本確認依 [Bun installation](https://bun.sh/docs/installation)。 |
| M2 | Docker Desktop for Mac、Docker Compose v2、PostgreSQL 開發 volume 與本機管理設定。 | Compose 啟動 PostgreSQL；Registry 是 Bun/TypeScript modular monolith。預設只提供 loopback 開發入口。 |
| M3 | 可用的 Docker daemon、Bun host process、owner-only Agent credential/journal/binding store、已核准的 arm64 hello-service image。 | Node Agent 直接在 Mac host 運行，取得該 host Docker context 的受控存取；不放進帶 Docker socket 的 workload container。 |
| M4 | M3 deployment healthy、hello-service 的實際 MCP endpoint 與登記的 tool permissions。 | 驗證 Streamable HTTP discovery 及 server-side authorization，不依賴 LLM。 |
| M5 | Host Ollama、已下載 `phi3:3.8b`、LiteLLM runtime、gateway credential、可達的 `OLLAMA_API_BASE`。 | Ollama 原生執行以使用 Mac inference 能力；兩個 chat aliases 均使用同一模型。固定 tag 存在於 [Ollama Phi-3 tags](https://ollama.com/library/phi3/tags)。 |

M1 建立 `apps/`、`services/`、`agents/node-agent/`、`packages/shared/`、`infrastructure/`、`registry/` 的 workspace 邊界；只為已交付元件加入 package。TypeScript 使用兩空格縮排，外部設定使用 Zod 或 JSON Schema 驗證，formatter/linter 隨首個實作交付。Root `package.json` 提供 `dev`、`build`、`test`、`lint`；使用 `bun install`、`bun run dev`、`bun run build`、`bun run test`、`bun run lint`。`bun run test` 聚合當前已實作範圍，不因 M1 尚無 M5 runtime 而失敗；各服務 README 記錄設定、port、啟動與清理方式。Repository 不混用多套 package-manager lockfiles。

## 一台真實 Node，複數 roles

唯一 enrolled Node 的 name 固定為 `dev-macbook`；nodeId 仍由 Registry 配發，不能直接把 name 當作 ID。`core`、`gpu`、`worker`、`storage` Compose profiles 控制開發元件啟用，labels 提供角色說明；Node roles/labels 由管理員核准，inventory 由真實能力產生。Compose profile 不會自動 enrollment，也不賦予 Node role。Profiles 是 Compose 的選擇啟動機制，沒有 profile 的 service 預設啟用；本專案用明確 profiles 對應開發階段。[Docker Compose profiles](https://docs.docker.com/compose/how-tos/profiles/)

M2 起提供 root `compose.yaml` 作 infrastructure 開發入口；`core` 啟動已交付平台元件及 PostgreSQL，M5 才加入 LiteLLM。`gpu` 只標示 inference 相關開發角色，host Ollama 不由 Compose 建立。`worker`/`storage` 在對應實作存在時才啟用。Compose role labels 不使用 [Agent reserved ownership labels](security.md#docker-execution-policy)，不讓 Agent 接管 Compose core containers。平台 hello-service 由 M3 Agent 建立，不能同時由 Compose lifecycle 管理。

不得註冊 `core-node`、`gpu-node` 等假 Nodes、虛構 GPU inventory 或以多個 containers 冒充多個故障域。Mac host 的 Metal inference 與 Docker 可分配 GPU 資源是不同能力；`gpuCount` 只回報受控 Docker runtime 真能分配的數量，不能因有 `gpu` role 就填非零。資源不足應呈現 `pending` / `AwaitingPlacement`。Host Ollama 是管理員準備的 inference dependency，不另立 Node，也不由 Node Agent 安裝／停止。

## Ollama 與 LiteLLM 設定邊界

以下是 M5 開發設定。`OLLAMA_API_BASE` 是供 LiteLLM 解析 `api_base` 的環境變數，不是 Manifest 欄位，也不會自動修改 Ollama listener。Host LiteLLM 使用 `http://127.0.0.1:11434`；Docker Desktop 中的 LiteLLM 使用 `http://host.docker.internal:11434`，container 內的 `localhost` 指向 container 自己。Host DNS 與連線方式見 [Docker Desktop networking how-tos](https://docs.docker.com/desktop/features/networking/networking-how-tos/)。

Ollama 預設綁 `127.0.0.1:11434`；先驗證 Docker Desktop 到 host listener 的可達性。若本機配置不能轉送到 loopback，可將 LiteLLM 暫時作為受限 host process，沿用同一 aliases 與 API contract。若需改 `OLLAMA_HOST`，依官方方式調整並重啟 Ollama，且先建立只允許本機／Docker Desktop 路徑的 firewall 或受控代理；不能無保護地對 LAN 開放 `0.0.0.0:11434`。跨主機連線必須 TLS server verification。`OLLAMA_HOST` 是 server listen 設定，與 client-side `OLLAMA_API_BASE` 分開。[Ollama FAQ：server configuration](https://docs.ollama.com/faq)

M5 建立 `infrastructure/litellm/config.yaml`，以下 YAML 是其最小設定範例；環境變數必須注入實際 LiteLLM 程序／container，不能只停留在啟動 Compose 的 shell。Gateway credential 由本機受限設定供應，不寫入範例或 Git。

```yaml
model_list:
  - model_name: home-large
    litellm_params:
      model: ollama_chat/phi3:3.8b
      api_base: os.environ/OLLAMA_API_BASE
  - model_name: home-fast
    litellm_params:
      model: ollama_chat/phi3:3.8b
      api_base: os.environ/OLLAMA_API_BASE
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

`model_name` 是 caller 名稱，`litellm_params.model` 是 gateway 的 backend mapping；`os.environ/` 由 LiteLLM config 讀取環境值。[LiteLLM proxy configuration](https://docs.litellm.ai/docs/proxy/configs) `ollama_chat/` 使用 Ollama chat integration；不在此設定 `supports_function_calling: true` 來宣稱 Phi-3 已驗證 tool calling。[LiteLLM Ollama integration](https://docs.litellm.ai/docs/providers/ollama)

Portal／Orchestrator 只能使用 `home-large`、`home-fast` 等已核准 logical aliases，不得直接連 Ollama、vLLM 或實體 GPU node；未知 alias 拒絕，不採 wildcard/backend passthrough。Gateway credential 留在 backend，瀏覽器經 Orchestrator。兩個 aliases 本機都映射 `phi3:3.8b`，不承諾不同品質或速度；embedding alias 延至 M7。完整決策見 [ADR 0004](adr/0004-litellm-logical-model-aliases.md)。

## Smoke tests 的共通約定

以下各節列出執行階段的 prerequisites、commands/requests 與 observable result。M1 提供 `bun run test` 的 deterministic contract fixtures；M2–M4 逐步提供 `bun run smoke:m2`、`bun run smoke:m3`、`bun run smoke:m4`，由測試 runner 建立隔離資料、讀取回傳 ID/generation、產生合法 Idempotency-Key，並執行各節 request 序列。這些是待相應 milestone 交付的指令契約，不是在 M0 建立空成功腳本。

Runner 從受限開發設定載入 endpoint、分別具 manage/read/無 tool grant 的 principals 及測試 binding；不輸出 credentials，亦不依賴固定 node IP 或 production secrets。API requests 遵循 [Control Plane](control-plane.md#rest-共通契約)：management mutations 帶 `Idempotency-Key`，PATCH/restart 帶最新 `expectedGeneration`。表中的 `{deploymentId}` 等是從 response 取得的 route parameters，不是可以照字串送出的 ID。Runner 不以 201/202 代替 runtime completion。

### M0：文件與 contracts

**Prerequisites：** 正式 `docs/` 文件集及靜態 parser；尚無 runtime。

**Commands/requests：** 在 repository root 執行：

```sh
git diff --check
rg --files docs
```

另以靜態檢查逐份驗證 Markdown 相對路徑與 anchors、fence 成對、Mermaid 可解析、YAML/JSON 可解析及 TypeScript snippets 可轉譯；依 [roadmap](roadmap.md) 核對 M0–M9 四項必要欄位，逐案 walkthrough registration、資源不足、offline、stale report、MCP mismatch 與 idempotent operations。檢查器及命令結果隨文件 review 提供，不預設 repository 已有檢查 script。

**Observable result：** 無壞連結、語法錯誤或 whitespace errors；Manifest、API、states、15/45 秒、aliases 與 security 邊界一致。只有文件驗證通過，不代表 M1–M5 runtime 驗收通過。

### M1：Bun workspace 與 deterministic fixtures

**Prerequisites：** M0 通過；M1 已建立 root scripts、鎖定工具版本、formatter/linter 與契約 tests。

**Commands/requests：**

```sh
bun --version
bun install --frozen-lockfile
bun run lint
bun run build
bun run test
```

**Observable result：** 安裝不改 lockfile；已交付 packages 可 build/lint；valid hello-service 通過，invalid Manifest、unknown fields、permission mismatch 與 stale generation fixtures 被拒絕。Fixture 的 Node/health/discovery 只存在隔離測試資料，不寫入正式 Node registry。M1 非持久 adapter 重啟可重置資料，介面明確標示 fixture mode；無 PostgreSQL、Ollama 或真實 Docker execution 也能完成本階段。

### M2：PostgreSQL Registry durability

**Prerequisites：** M1 通過；Compose `core` 包含 PostgreSQL、Registry dev 設定與 `registry:manage` 測試 principal；migrations、audit、idempotency storage 及 smoke runner 已交付。

**Commands/requests：**

```sh
docker compose --profile core config --quiet
docker compose --profile core up -d
bun run dev
bun run smoke:m2
```

`bun run dev` 在另一個 terminal 運行 Registry；runner 依序執行下表。M2 不啟動 Node Agent。

| Request / 操作 | Observable result |
| --- | --- |
| `POST /api/v1/service-definitions`，完整 valid Manifest，重送相同 key/body。 | 首次 201；重送相同 response 並有 `Idempotency-Replayed: true`；只存在一個 revision。 |
| `POST /api/v1/deployments`，body 使用該 serviceDefinitionId 及 desiredState=running。 | 初始 generation=1、observedGeneration=0、pending/unknown；無 enrolled Node 時保持 AwaitingPlacement，無 Docker side effect。 |
| 重啟 Registry 後 `GET /api/v1/service-definitions` 與 `GET /api/v1/deployments/{deploymentId}`；重送建立 deployment 的同 key/body。 | ID、intent、generation 與已提交 audit/idempotency record 保留；不重建第二個 deployment。 |
| 同 key 改 body；先用新 key/current expectedGeneration PATCH stopped，再以另一新 key/先前 generation PATCH running；無權 principal mutation。 | 依序得到 IdempotencyConflict、成功 stopped mutation 後的 GenerationConflict、403；被拒絕的 requests 無新 intent 或 command。 |
| Runner 以隔離 fixture 觸發 audit 寫入失敗／DB unavailable。 | 整筆 mutation rollback；DB 不可用回 503，不回記憶體假成功；一般 response 不含 resolvedBindings 或 secrets。 |

**Observable result：** 以上結果均可由 HTTP、隔離 DB 的受限 read-only assertions 及 redacted audit 確認。Enrollment digest/revoke、commands、sequence 等 M3 所需持久契約在 M2 已建置並以 fixtures 驗證；M3 再驗真實 Agent。

### M3：唯一真實 Node 與 lifecycle

**Prerequisites：** M2 通過；管理員為 `dev-macbook` 核准 roles/labels 並簽發一次性 enrollment token；host Agent 的 process lock、durable journal、credential store 已就緒；hello-service image 在 Mac 本機建置並核准完整 local image ID，logical storage/secret bindings 已授權。Runtime fixture 提供 HTTP health，MCP 實際 discovery 留 M4。

**Commands/requests：** 啟動 M3 交付的 `bun run dev:agent` host process，於另一 terminal 執行 `bun run smoke:m3`。Runner 透過受限 Agent 測試介面協調下列序列，不將 raw enrollment body 或 credentials 寫進報告。

| Request / 操作 | Observable result |
| --- | --- |
| Agent `POST /api/v1/agent/enroll`，再 `POST /api/v1/agent/heartbeat`；管理端 `GET /api/v1/nodes`。 | 只有一個真實 dev-macbook，nodeId 為 Registry 回傳值；heartbeatIntervalSeconds=15、offlineAfterSeconds=45。不同 enrollmentId 重用 token 401。 |
| 依 M2 註冊 Manifest、建立 running deployment，查 `GET /api/v1/deployments/{deploymentId}`。 | Placement 新 generation，fresh heartbeat 給 runAllowed=true、已提交 reservationId 與 resolvedImage；Agent 只使用核准 image identity。可觀測 pending → deploying → running 且 health=healthy；尚未真實 MCP discovery 不可發布 capability。 |
| `PATCH /api/v1/deployments/{deploymentId}`，desiredState=stopped；確認 stopped，再以 running 重啟；重送原 key/body。 | generation 只因新 intent 遞增；確認 stopped 後釋放 reservation；同 key 重送不再 mutation。同 desired/new key/current generation 為 200 no-op。 |
| `POST /api/v1/deployments/{deploymentId}/restart`，追蹤 `GET /api/v1/deployments/{deploymentId}/commands`；重送 management request 與 Agent command。 | 只有一個 commandId、一次 restart；command succeeded 後仍需 fresh observation/probe；journal 不確定結果呈 IndeterminateExecution，不盲目重做。 |
| 停止後由另一真實 workload fixture 占滿可用 reservation，再 start 原 deployment。 | pending/AwaitingPlacement、runAllowed=false；Docker inspection 證明未 pull/create/start/restart。釋放資源後才獲 fresh authorization；不建假 Node。 |
| Runner 暫停 host Agent heartbeat 至最後有效 heartbeat 達 45 秒，期間送 stopped intent；恢復 Agent。 | Node offline，部署有效投影 unknown/unknown、NodeOffline、capability unavailable；不假設 runtime 停止。重連取得 current stopped intent 後才確認 stopped。 |
| 受控 protocol fixture 發送舊 generation、同代倒序／舊 boot session、跨 Node report/result、ownership label mismatch，再做 credential rotation/revoke。 | 舊資料不覆蓋 state；合法新 heartbeat 內的 stale item 進 rejectedReports，但 Node liveness 仍更新。Unauthorized/session/envelope 拒絕不延長 liveness；revoke 立即阻派工；其他 containers 不被操作。 |

**Observable result：** HTTP state、Docker container identity/labels、Agent journal 與 audit 相符。每 15 秒 heartbeat、45 秒 offline、generation fencing、單次 restart、secret redaction、audit rollback 均有證據。負向 protocol fixture 不代表另註冊假 Node；跨 Node 測試用隔離的 credential/ownership fixtures。

### M4：實際 MCP discovery 與授權

**Prerequisites：** M3 通過；完整 hello-service Manifest 中的 `hello` tool、`hello-service:read` catalog/grants、Streamable HTTP revision `2025-11-25`、受限 discovery credential 與 user delegation 已實作。

**Commands/requests：** 執行 `bun run smoke:m4`。Runner 等 current generation/healthy，讓 Registry initialize、送 initialized notification、讀完 `tools/list` 所有分頁，再用已授權 principal 請求 `GET /api/v1/capabilities`，附 deploymentId query filter（值由先前 response 綁定）。以受信任 backend test caller 對解析的 MCP endpoint 送 `tools/call`，arguments 為 `{"name":"smoke"}`；測試 caller 執行與未來 Orchestrator 相同的逐次授權邊界，無須模型提議 tool。

**Observable result：** 完整 tool 集合恰為 hello 且 mapping/healthy 通過時 capability 才可見，合法呼叫收到 greeting；無 grant、錯 audience、失效 delegation 或非法 arguments 不會執行 tool。增加／刪除 tool、缺 mapping、unhealthy、offline 或 tools/list_changed 會撤銷該 deployment 全部 MCP capabilities；current generation 變更同樣使舊 discovery snapshot/capabilities 失效，此時 `GET /api/v1/capabilities/{capabilityId}` 回 404。遲到的 stale report 本身被拒絕，不更新或撤銷 current projection。恢復需新完整 discovery，不沿用舊快照；tool allow/deny/result 有 redacted audit。M4 不驗證 Phi-3 的 tool-calling 能力。

### M5：phi3:3.8b chat 與 streaming

**Prerequisites：** M4 通過；host Ollama 服務已啟動，M5 LiteLLM 設定及受限 gateway key 已注入；`LITELLM_BASE_URL` 是測試 backend 可達的 LiteLLM base URL，`LITELLM_API_KEY` 是授權 smoke caller key，均由本機設定載入。此測試由管理員 backend terminal 執行，gateway key 不交給瀏覽器。

**Commands/requests：** 以下指令於 M5 才實際執行：

```sh
ollama --version
ollama pull phi3:3.8b
ollama list
docker compose --profile core config --quiet
docker compose --profile core up -d litellm
curl --fail-with-body --max-time 120 "$LITELLM_BASE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LITELLM_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"home-fast","messages":[{"role":"user","content":"Say hello in one sentence."}],"max_tokens":64,"stream":false}'
curl --fail-with-body --no-buffer --max-time 120 "$LITELLM_BASE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LITELLM_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"home-large","messages":[{"role":"user","content":"Say hello in one sentence."}],"max_tokens":64,"stream":true}'
```

`120` 秒是本次手動 smoke request 的 client timeout，不是平台 heartbeat 或產品 latency SLA。若用 host LiteLLM 排除 Docker host networking 問題，改由該 M5 runtime 的啟動方式載入同一 config 並略過 Compose litellm 啟動；其餘 requests 相同。不得執行 `set -x` 或收集帶 Authorization 的 verbose logs。

**Observable result：** `ollama list` 可見固定 tag；非串流 response 有非空 `choices[0].message.content`，串流逐步收到 SSE `data:` chunks、文字 delta 與正常終止訊號。交換兩個 requests 的 model 再驗一次，使兩 aliases 都涵蓋 chat 與 streaming；不以特定生成文字作 deterministic assertion。記錄版本、模型 ID、aliases、完成／錯誤分類，不保存私密 prompt。

另以不存在的 alias、無 credential、停止 host Ollama 後的同一 chat request 做負向測試：未知 alias 不解析任意 backend，無 credential 被拒絕，backend unavailable 回受控失敗且沒有 direct backend fallback；恢復 host Ollama 後同 alias 可再次成功。這些是真實 M5 runtime acceptance；本次 M0 文件完成不要求它們已執行。
