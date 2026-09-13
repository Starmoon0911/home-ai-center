# ADR 0001：Registry-centered modular monolith

- Status：Accepted
- Date：2026-09-13
- Scope：M0–M3 Registry、PostgreSQL ownership、reconciliation 與 API。

## Context

家庭平台的首要需求是可驗證地保存服務宣告、部署 intent、真實 Node 狀態與受權限限制的 discovery。Start/stop、assignment、generation fencing、restart command、idempotency 與 audit 密切相關；初期拆成多個獨立服務會使單次 intent mutation 需要跨服務協調，而目前沒有必須獨立擴縮的證據。

## Decision

Registry 採 modular monolith，同一程序內分成 Definitions、Deployments、Nodes、Discovery、Authorization/audit modules，透過 owner 介面合作。PostgreSQL 自 M2 起為權威持久層；deployment intent、generation、availability 撤銷、command、idempotency 與 audit 在同一 transaction 提交。外部 Docker／MCP side effects 在交易外執行，入庫時重新驗證 ownership/generation，細節依 [Control Plane](../control-plane.md)。

以持久 desired state 與 commands 驅動 reconciliation，失去記憶體 queue/wakeup 後仍可重建工作；M0–M3 不無條件加入 Redis 或 message broker。公開 API 固定 `/api/v1`，一致使用 `application/problem+json`；start/stop 改 desiredState，restart 使用一次性 AgentCommand 與 Idempotency-Key。安全邊界依 [security model](../security.md)。

Orchestrator、LiteLLM、Node Agent 與業務 services 保持各自程序／資料責任，不因 Registry modular monolith 而併成全平台單一程序。模組邊界是未來拆分的契約，但現在不預建 distributed transaction 或多 Registry leader election。

## Consequences

可在單台 `dev-macbook` 完成 M0–M3，減少部署元件並以 PostgreSQL transaction 保護 generation/idempotency/audit 一致性。測試可直接驗證重送、競爭 intent、重啟及 stale report；後期加入 MCP、RAG、SSO 不改變既有 ownership。

Registry/DB 暫時不可用會停止新 intent 與派工，discovery fail closed；既有 runtime 不會因此自動停止。Monolith 需要嚴格 module API，不能任意跨 module 寫表。未來若量測顯示獨立 scale 或 failure isolation 的需要，拆分時另立 ADR 並定義一致性遷移，不把可拆分誤當成現成高可用。

## Rejected alternatives

| Alternative | 拒絕理由 |
| --- | --- |
| 初期 Registry microservices | 為同一 intent 加入 network failure、分散式 idempotency 與跨服務交易，超出 M0–M3 的交付需要。 |
| 只用記憶體或 YAML 作 runtime authority | 無法可靠保存重啟後的 generation、command 去重、credential revoke 與 audit。 |
| Redis queue 作唯一命令來源 | queue 遺失／重送不能代表 intent；先採 PostgreSQL 持久契約足以完成初版。 |
| Portal 直接操作 Docker／資料庫 | 繞過 server-side authorization、ownership 與 audit，破壞控制／執行邊界。 |

本決策與 [ADR 0002](0002-node-agent-outbound-pull.md) 的 execution boundary 及 [ADR 0003](0003-separate-manifest-and-deployment.md) 的 declaration/deployment 分離互補。
