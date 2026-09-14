# Service Manifest contract

Manifest 是 portable `ServiceDefinition` 的版本化宣告輸入；Registry 驗證並保存不可變 revision，建立 `Deployment` 時才選擇 Node、解析實際 endpoint 與 bindings。此處是 documentation contract，不建立 executable JSON Schema、runtime launcher 或 SDK。Entity 與 state 定義見 [domain model](domain-model.md)，分離理由見 [ADR 0003](adr/0003-separate-manifest-and-deployment.md)。

## home-ai/v1 envelope 與 validation

Root 必須有 `apiVersion: home-ai/v1`、`kind: ServiceDefinition`、`metadata`、`spec`。YAML 與 JSON 輸入先正規化為相同 JSON-compatible value；拒絕重複 mapping keys、自訂 YAML tags、aliases/merge keys、多文件輸入及非有限數值，避免不同 parser 解讀不一致。字串不做環境變數插值，也不執行模板。

| 欄位 | v1 contract |
| --- | --- |
| `metadata.name` | 必填，1–63 字元，lowercase kebab-case：`[a-z0-9]+(-[a-z0-9]+)*`。 |
| `metadata.version` | 必填，`major.minor.patch` 三個非負整數，以字串表示、不含 leading zeros、prerelease 或 build suffix；同 name/version 的內容不可變。 |
| `metadata.description` | 選填，人類可讀說明，不參與 authorization。 |
| `metadata.annotations` | 選填，string-to-string map，最多 32 entries、每值最多 1024 字元；只供非執行 metadata。 |
| `spec.runtime` | 必填，`type: docker`、非空 `image`；image 為 OCI-compatible image reference，部署前需通過平台 image policy。沒有 arbitrary flags、host network 或 privileged 選項。 |
| `spec.ports` | 必填非空 named map；每項為 `{containerPort, protocol}`，整數 port 1–65535，protocol v1 僅 `http`。名稱遵守 metadata.name 規則；container ports 不重複。不宣告 hostPort。 |
| `spec.paths` | 必填非空 named map；值為 service-local absolute path，以 `/` 開頭但不能 `//`、URI scheme、query、fragment、反斜線、encoded slash 或 `..` segment。名稱遵守 metadata.name 規則。 |
| `spec.web` / `spec.api` | 選填 `{port, path}`，分別引用 ports 與 paths 的名稱；不接受 URL。省略表示不提供該介面。 |
| `spec.health` | 必填 `{port, path, intervalSeconds, timeoutSeconds, failureThreshold, successThreshold}`，port/path 為名稱引用；其餘必填正整數，timeoutSeconds 小於 intervalSeconds。 |
| `spec.placement` | 必填 `{roles, labels, architectures, resources}`；皆為 constraints，不能加入 nodeId/name/address。roles 與 architectures 為非空、不重複 string arrays；labels 為 string map，可空。 |
| `spec.placement.resources` | `{cpuCores, memoryMiB, gpuCount}`；cpuCores 為正數，memoryMiB 為正整數，gpuCount 為非負整數；是最低需求，不能宣告虛構 inventory。 |
| `spec.storage` | 必填 array，可空；每項 `{name, ref, mountPath, readOnly}`，name 唯一、ref 為 logical storage key，mountPath 必須為 container 內的 canonical POSIX absolute path，依下述順序驗證格式、敏感目錄與重疊；readOnly 為 boolean。 |
| `spec.secrets` | 必填 array，可空；每項 `{name, ref, env}`，name 與 env 各自唯一，ref 為 logical secret key，env 為注入的環境變數名稱。沒有 value 欄位。 |
| `spec.permissions` | 必填 array，可空；列出此服務使用的已由平台管理員登記的 permission keys，不會建立權限或替任何使用者授權。 |
| `spec.ai` | 選填；v1 僅接受下述 `mcp` object。UI-only 或 API-only 服務省略。 |

Logical storage/secret key 使用 metadata.name 的格式；permission key 使用 `<service-name>:<action>`，action 使用 lowercase kebab-case。`env` 遵循 `[A-Z_][A-Z0-9_]*`。陣列 entries 不得重複；所有 port/path references 必須存在。CPU architecture 使用 `arm64` 或 `amd64`；roles 必須是平台支援的 role labels。所有 roles 及 label constraints 為 AND，architectures 為 OR，最低 resources 同時滿足；沒有候選 Node 時 definition 仍有效，但 deployment 保持 pending，reason `AwaitingPlacement`。

Health probe 使用 HTTP GET；2xx 為成功，timeout、redirect 與其他 status 為失敗，不跟隨 redirect。Container 初次執行時 health 為 starting，連續 successThreshold 次成功才 healthy，連續 failureThreshold 次失敗變 unhealthy；任一反向結果重設相應連續計數。健康證據的新鮮度與 probe 中斷處理交由 implementation policy；失去可信健康證據時不能繼續提供 capability。Restart、stop 或 generation 變更清除計數與舊結果。Probe 不經公開 UI route，而由受限 backend 解析同一 deployment 的 port/path。

Validation 次序為 envelope/version → 結構與型別 → references/duplicates → portability/runtime security policy → permission catalog → definition identity/namespace ownership；任一步失敗均拒絕保存，回報穩定的 error code、field path 及不含 secrets 的原因。不 echo 原始輸入值，尤其是疑似 literal secret。

`home-ai/v1` 對 root 與所有 contract objects 的未知欄位 fail closed；只有 metadata.annotations 保留非執行擴充空間。未知 apiVersion/runtime/transport 直接拒絕，不降級猜測。新增執行語意或 optional 欄位需要明確的新 apiVersion 與 migration，舊 consumer 不會靜默忽略安全欄位。HTTP API 的版本與 Manifest apiVersion 分開演進。未來 schema validator 應選定 JSON Schema dialect，並實作欄位驗證以外的 semantic policy；JSON Schema 的 `$schema` 用來識別 dialect，不能單獨證明 runtime policy 合規。[JSON Schema Core 2020-12](https://json-schema.org/draft/2020-12/json-schema-core)

## Routes、placement 與 bindings

`ports` 的數值是 container listener；`paths` 的值是 service-local path。`web`、`api`、`health`、`ai.mcp` 同時引用 port name 與 path name，Registry 從 deployment 的受控 network identity 解析 endpoint。Public route 由 Edge/Traefik 配置並強制 authentication；Manifest 不能指定外部 hostname、完整 runtime URL、直接 backend address 或繞過 Edge。Web path 的 prefix forwarding 規則保留 service-local prefix，不隱式 strip/rewrite；需不同公開 prefix 時由部署側明確設定並驗證相容性。

Placement 只表達能力與資源需求。不得使用唯一主機 selector 假裝 constraint：node name、ID、IP、hostname 及等價的固定位置 label 都不允許；角色與一般能力標籤由管理員控管。`assignedNodeId` 只能存在 Deployment。Manifest 中的 image repository 位址不是 runtime endpoint，允許 image references，不因此允許 service URL。

Storage 的 `ref` 由管理員建立的 storage binding catalog 解析，且必須授權給該服務及 Node；`mountPath` 是 container 內部路徑，不是 host path。Host-specific source 只可存在受限 deployment binding，不能從 Manifest 偷渡；placement 前驗證 binding 可達性。

`mountPath` 採 canonical POSIX absolute path，依下列順序檢查；validator 不得先 normalize 再接受原始輸入：

1. 先驗證原始字串以單一 `/` 開頭；拒絕相對路徑、`.` 或 `..` segments、重複分隔符 `//`、除根路徑外的 trailing `/`、反斜線及 control characters（包含 NUL），code `NonCanonicalMountPath`。例如 `/data/../etc`、`//etc`、`/data/./child` 均直接拒絕，不能先化簡成其他路徑；`/data/child` 才是 canonical 表示。這是字串層級的 path contract，不做 URL decoding 或 shell expansion。
2. 所有 mountPath 通過格式檢查後，以 `/` 切成大小寫敏感的完整 path segments。拒絕根路徑 `/`，以及與系統敏感目錄 `/proc`、`/sys`、`/dev`、`/etc`、`/var/run` 相同、在其下方或為其祖先的 target，code `RuntimePolicyDenied`；祖先 mount 也可能遮蔽敏感目錄。以 segment 關係判定，不能只用字串 prefix：`/etc/config` 不允許，`/etc-data` 不因此被當成 `/etc` 子目錄。
3. 再逐對檢查本 Manifest 的 mount targets；segments 完全相同，或任一路徑的 segments 是另一路徑的完整前綴，皆以 `OverlappingMountTargets` 拒絕。例如 `/data` 與 `/data/child` 重疊，`/data` 與 `/database` 不重疊。必須先拒絕非 canonical 輸入，不能讓 `/data/./child` 與 `/data/child` 以不同字串逃過相同 target 檢查。

通過上述檢查不豁免 Docker socket 或 unmanaged host mount 禁令；runtime 必須使用驗證過的原始 target，不得在後續 decoding、插值或 normalization 中改變其意義。

Secret ref 由管理員建立的 secret catalog 解析，部署時以指定 env 注入授權的 service instance。Manifest 不提供 generic env values、inline credentials 或 secret value；錯誤、logs、API 與 audit 都不輸出解析值。缺少或未授權 logical ref 時 deployment 不可啟動；不能退回空值或讀取 host 任意檔案。平台 runtime policy 強制拒絕 privileged、host network、Docker socket、arbitrary runtime flags，以及未管理的 host mounts；未知欄位本身也會被 validation 拒絕。

## MCP discovery 與 permissions

`spec.ai.mcp` 必填欄位如下；僅提供 MCP 的服務仍須宣告 health：

| 欄位 | 規則 |
| --- | --- |
| `transport` | 固定 `streamable-http`；不接受 legacy HTTP+SSE 或 stdio launcher。 |
| `port` / `path` | 引用 spec.ports/spec.paths 名稱，解析單一 deployment 的 MCP endpoint。 |
| `namespace` | 使用 metadata.name 格式；由 Registry 檢查跨服務唯一，相同服務各 revision 維持一致。 |
| `tools` | 非空 allowed tools array；每項 `{name, permissions}`。name 使用 `[A-Za-z0-9_-]+`、最多 128 字元且不重複。permissions 必須是非空、不重複 permission keys，且每一項都存在 spec.permissions 及平台 permission catalog。 |

Home AI v1 採 MCP protocol revision `2025-11-25` 的 Streamable HTTP；初始化時協商該 revision，不相容則拒絕 discovery。Streamable HTTP 的單一 MCP endpoint 支援 POST/GET，GET 若未提供 SSE 可回覆 405；客戶端仍須處理 POST 回傳 JSON 或 SSE，不能把 `streamable-http` 誤當成只支援 JSON。初始化後的 HTTP requests 必須攜帶協商的 `MCP-Protocol-Version`；server 若配發 `MCP-Session-Id`，後續 requests 亦須帶上。這是專案固定基準，升級 protocol revision 須更新契約與相容性驗證。[MCP Streamable HTTP transport](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)

Registry 在 Node online、generation 已確認且服務 healthy 後，以受限 backend credential 連到解析出的 endpoint，執行 initialize、initialized notification，再呼叫 `tools/list` 並讀完所有分頁。拒絕跨 endpoint redirect；server 提供的 metadata 不可覆寫已解析 endpoint。Tool name set 必須與 Manifest allowed tools 完全相等，inputSchema 必須是有效 JSON Schema object，重複 names、缺漏、額外 tools 或不合法 schema 都令 discovery 失敗。實際 schema 供 arguments validation，Manifest 不重複維護 schema 副本。MCP 定義了 tool schema、`tools/list` 的 pagination 與 `tools/list_changed` notification；平台在此之上增加完整集合匹配與 permission policy。[MCP tools specification](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)

四重 gate 為 Manifest allowed tools、實際 `tools/list`、完整 permission mapping、healthy service instance；另須符合 [domain model](domain-model.md#capability-availability) 的 Node、intent 與 generation 條件。MCP 不提供此平台的使用者 permission grants，完整 mapping 是平台驗證：每個實際 tool 必須恰有一筆 Manifest entry，其所有 required permissions 都有效；多個 permissions 採 AND。spec.permissions 可包含 API/Web 使用的額外有效 permission，不能用此額外集合默認授權未宣告 tool。

成功快照綁定 deploymentId、generation、namespace、tool schemas 與 mapping。Discovery refresh timing、cache 有效期限與無通知 server 的重新驗證時機交由 implementation policy；可用性必須有當前成功的完整 tools/list 證據，過期或失敗快照不能提供 capability。收到 `notifications/tools/list_changed` 立即撤銷快照並重新列舉。Offline、unhealthy、健康證據失效、stop/restart、definition/binding 改變或 discovery failure 都撤銷當前能力。Orchestrator 於每次呼叫前確認快照仍有效，驗證 arguments 與 caller 的全部 required permissions，server-side 另行 enforce；server annotations 或模型輸出都不是授權來源。

## Valid：portable hello-service

以下為完整、結構有效的教學 Manifest；`hello-service:0.1.0` 是待本機建置並交由 image policy 接受的 image reference，不表示目前已提供此 image。範例假設管理員事先登記 `hello-service:read`、storage `hello-data` 與 secret `hello-api-token`。這些 catalog bindings、實際 Node 與 runtime URL 均由部署環境決定，範例不包含 secret value。

```yaml
apiVersion: home-ai/v1
kind: ServiceDefinition
metadata:
  name: hello-service
  version: "0.1.0"
  description: "示範可部署的 Web、API 與 MCP service"
spec:
  runtime:
    type: docker
    image: hello-service:0.1.0
  ports:
    http:
      containerPort: 8080
      protocol: http
  paths:
    web: /
    api: /api
    health: /health
    mcp: /mcp
  web:
    port: http
    path: web
  api:
    port: http
    path: api
  health:
    port: http
    path: health
    intervalSeconds: 10
    timeoutSeconds: 2
    failureThreshold: 3
    successThreshold: 1
  placement:
    roles: [worker]
    labels: {}
    architectures: [arm64, amd64]
    resources:
      cpuCores: 0.25
      memoryMiB: 128
      gpuCount: 0
  storage:
    - name: data
      ref: hello-data
      mountPath: /data
      readOnly: false
  secrets:
    - name: api-token
      ref: hello-api-token
      env: HELLO_API_TOKEN
  permissions:
    - hello-service:read
  ai:
    mcp:
      transport: streamable-http
      port: http
      path: mcp
      namespace: hello-service
      tools:
        - name: hello
          permissions: ["hello-service:read"]
```

單節點 Mac 的真實 `dev-macbook` 可帶 worker role；範例沒有建立第二個 Node。解析出的 service instance 若 healthy，`tools/list` 的完整結果須恰為以下集合，才可繼續通過 permission gate：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "hello",
        "description": "Return a greeting",
        "inputSchema": {
          "type": "object",
          "properties": { "name": { "type": "string" } },
          "required": ["name"],
          "additionalProperties": false
        }
      }
    ]
  }
}
```

## Invalid examples 與拒絕原因

以下 YAML 是獨立的欄位覆寫案例，套用到上方 valid example 的對應 field path，不是可獨立提交的完整 Manifest，也不是 executable patch format。每案僅展示觸發拒絕的輸入；validator 不得因為認得其意圖而自動修正。

### 固定 runtime URL／host IP

```yaml
spec.web:
  url: http://192.0.2.10:8080/
```

拒絕：`UnknownField` / `NonPortableEndpoint`；web 只接受 named port/path reference，不能固定 runtime URL 或 host IP。範例 IP 僅作無效輸入，不是部署設定。

### 固定 Node

```yaml
spec.placement:
  nodeId: dev-macbook
```

拒絕：`UnknownField` / `NonPortablePlacement`；placement 僅 constraints，不能指定 nodeId。透過 `labels: {hostname: dev-macbook}` 或固定 host IP selector 也必須由 portability policy 拒絕。

### Literal secret

```yaml
spec.secrets:
  - name: api-token
    value: NOT_A_REAL_SECRET
    env: HELLO_API_TOKEN
```

拒絕：`UnknownField` / `LiteralSecretForbidden`；secrets 只能 logical ref，不接受 value。錯誤不 echo value。此字串是無效示例，不是 credential。

### Privileged container

```yaml
spec.runtime.privileged: true
```

拒絕：`UnknownField` / `RuntimePolicyDenied`；任何 privileged execution 都超出受控 runtime policy。

### Unmanaged host mount

```yaml
spec.storage:
  - name: data
    hostPath: /Users/example/private
    mountPath: /data
    readOnly: false
```

拒絕：`UnknownField` / `HostMountForbidden`；Manifest 不可指定直接 host path，必須使用預先授權的 logical ref。

### 非 canonical mountPath

```yaml
spec.storage:
  - name: data
    ref: hello-data
    mountPath: /data/../etc
    readOnly: false
```

拒絕：`NonCanonicalMountPath`；原始 target 含 `..` segment，在敏感目錄與 overlap 檢查之前直接拒絕，不可先 normalize 為 `/etc`。同樣拒絕 `//etc` 與 `/data/./child`；後者不能與 canonical `/data/child` 被當成不同的合法 targets。

### Unknown runtime

```yaml
spec.runtime:
  type: shell
  image: hello-service:0.1.0
```

拒絕：`UnsupportedRuntime`；v1 僅 docker，不能將未知 type 解讀成 host launcher。

### Tool 缺少 permission mapping

```yaml
spec.ai.mcp.tools:
  - name: hello
    permissions: []
```

拒絕：`PermissionMappingIncomplete`；每個 allowed tool 都須非空 mapping。換成不存在於 spec.permissions 或平台 catalog 的 permission 也拒絕，code `UnknownPermission`。

### Discovery 多出未宣告 capability

```yaml
discovery.toolsListNames: [hello, delete_all]
manifest.allowedToolNames: [hello]
```

此段是 discovery 對照資料，不是 Manifest 欄位。拒絕：`ToolSetMismatch`，撤銷整個 deployment 的 MCP capabilities。實際集合若為空、只有 `delete_all` 或重複 hello 也不相符；不得替新增 tool 推導 permission 或僅把它從 UI 隱藏。

### Host network、Docker socket 與 arbitrary flags

```yaml
spec.runtime.networkMode: host
```

拒絕：`UnknownField` / `RuntimePolicyDenied`；不能繞過平台 network boundary。

```yaml
spec.storage:
  - name: docker-socket
    ref: docker-socket
    mountPath: /var/run/docker.sock
    readOnly: true
```

拒絕：`RuntimePolicyDenied`；即使使用 logical ref 及 readOnly，也不能將 Docker socket 暴露給 workload。

```yaml
spec.runtime.extraArgs: ["--cap-add=SYS_ADMIN"]
```

拒絕：`UnknownField` / `RuntimePolicyDenied`；不提供 arbitrary runtime flags 或增加 host capabilities 的 escape hatch。

### 未知版本與懸空 reference

```yaml
apiVersion: home-ai/v999
```

拒絕：`UnsupportedManifestVersion`，不得 fallback 到 v1。

```yaml
spec.health.port: admin
```

拒絕：`UnresolvedPortReference`；valid example 僅宣告 http，不得猜測 admin 的 port number。
