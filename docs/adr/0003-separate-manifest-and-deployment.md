# ADR 0003：分離 Manifest 與 Deployment

- Status：Accepted
- Date：2026-09-13
- Scope：ServiceDefinition、home-ai/v1 Manifest、Deployment 與 discovery。

## Context

同一服務需要先在 Apple Silicon 單節點開發，再部署到未來的 Core、GPU 或 worker 主機。把 node name、host IP、完整 runtime URL、storage host path 或 literal secret 寫進服務宣告，會讓每次環境切換都需要改服務定義，並混淆使用者 intent、實際運行狀態與 credential ownership。單一服務也可能有多個 instances，不能用一個全域 healthy 旗標代表所有 deployments。

平台需能保存穩定、可驗證的服務契約，並由 Registry 根據真實 Node inventory 與管理員核准的環境 bindings 做 deployment。Node Agent 回報 runtime observation 的權限，不應包含改寫服務宣告或部署 intent。

## Decision

採用 [home-ai/v1 Manifest](../service-manifest.md)作為 `ServiceDefinition` 的 portable 宣告輸入。Definition revision 不可變，只包含 runtime image、named ports/paths、health、介面、placement constraints、logical storage/secrets 與 permissions。禁止固定 node、完整 runtime URL、host IP、直接 host path、literal secret 及不受控 runtime flags。

另以 [Deployment](../domain-model.md)表達具體 service instance，承載 definition revision reference、assignedNodeId、resolvedBindings、desiredState、observedState、healthStatus、generation、observedGeneration 及 stateReason。Registry 擁有 intent 與 generation；Node Agent 僅執行受控工作並回報當前 assignment/generation 的 observation。舊 generation 不得覆寫新 intent 或恢復 capability availability。

Web/API/health/MCP endpoint 由 deployment 解析 named port/path references。MCP capabilities 逐 deployment 通過 allowed tools、實際 tools/list、完整 permission mapping、healthy 四重 gate，並受 Node online 與 generation freshness 約束。可用性不代替 caller authorization。start/stop 改變 desiredState；一次性 restart 使用綁定 deployment generation 的 AgentCommand。

## Consequences

同一份 Manifest 可在符合 constraints 的不同主機使用，服務作者不需要知道實際 IP、storage source 或 secret values。Definition revision 可獨立 review、驗證與重用；每個 deployment 的狀態、故障及操作可被追蹤，不會污染共享宣告。

Registry 必須實作 placement、binding resolution、版本關係、generation fencing 與 per-deployment discovery，UI 也需分開呈現定義與 instance。通過 Manifest validation 不保證可部署：可能沒有符合 constraints 的 Node，或環境缺少授權 binding；這些是 deployment 的 pending/失敗原因。Definition 更新不自動改變既有 deployment，需明確選擇新 revision 並產生新 generation。

Resolved bindings 屬敏感 backend 資料，API 必須控制投影與存取。單節點開發也保留此分離，不為本機建立固定 node shortcut。此決策不承諾 M0–M3 自動跨 Node failover、k3s、進階 scheduler 或 secret manager 特定產品。

## Rejected alternatives

| Alternative | 拒絕理由 |
| --- | --- |
| 一個 Service object 同時包含宣告、node、URL 與 status | 多 instance 時無法表達獨立 intent/health；主機搬移污染宣告，也難以 fence 舊 observation。 |
| 每個環境維護一份完整 Manifest | 重複 metadata 與 permissions，易產生 drift；host/secret 資訊仍可能隨宣告散布。 |
| 直接以 Docker Compose 作為平台 Manifest | Compose 可做開發基礎設施，但完整 runtime 設定面超過 portable、安全受控的平台 contract，亦不能取代平台 permission/discovery model。 |
| 先將 node/IP 寫入 Manifest，日後再拆 | M0–M3 的 API、validation 與資料 ownership 會先固化錯誤邊界，增加遷移與測試成本。 |

四 plane、Node Agent outbound pull 及 source-of-truth 政策沿用[架構](../architecture.md)與[文件索引](../README.md)。
