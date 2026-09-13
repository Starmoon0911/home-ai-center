# ADR 0002：Node Agent outbound pull

- Status：Accepted
- Date：2026-09-13
- Scope：Node enrollment、heartbeat、command delivery、runtime ownership 與故障收斂。

## Context

家庭 Node 可能休眠、斷線、位於不同 LAN/VPN 或透過 NAT。Registry 主動對各主機開啟 SSH／RPC 會增加 inbound attack surface、host credentials 與連線管理。另一方面，重送 restart、遲到 healthy report 與離線後的錯誤 failover 都可能使新的 stop intent 失效或造成雙重執行。

## Decision

Node Agent 作為 host daemon 主動經 authenticated outbound HTTPS 向 Registry enrollment、每 15 秒 heartbeat，heartbeat 同時上傳 observations/progress 與拉取 current desired deployments/commands。Registry 以 server 接收時間計算，最後有效 heartbeat 達 45 秒即 offline；result 由 `/api/v1/agent/commands/{commandId}/result` 回報。沒有 inbound Agent 控制 port 或 arbitrary shell RPC；完整 paths 與 envelope 見 [Control Plane](../control-plane.md#agent-outbound-pull-protocol)。

Registry 擁有 desiredState、assignment 與 generation；Agent 只能操作自己的受控 Docker objects。Start/stop 是可重複收斂的 desired state，restart 是一次性、綁 generation 的 AgentCommand。使用 Idempotency-Key 防 management request 重複建立意圖，使用 durable command journal 防 command 重送再 restart，使用 boot/report sequence 排除同 generation 的倒序 observation。失聯不開始新 runtime mutation；每次 mutation 前先取得 fresh heartbeat response，舊 generation 不能覆寫新 intent。

Enrollment 採一次性 token 換取可到期、撤銷與 rotation 的 per-node credential；Docker allowlist、平台 labels 與本機 durable mapping 同時驗證 ownership。Credentials、TLS 與 host 信任模型見 [security model](../security.md)。

## Consequences

Agent 只需連向固定 Registry 入口，支援單節點 `dev-macbook` 與未來多實體 Nodes；credential revoke 與 assignment 限制可統一 enforce。代價是操作傳遞通常需等下一次 15 秒 heartbeat，UI 必須呈現 desired/observed/health 及 pending command，不能把 202 當完成。

離線後撤銷 discovery/capability，不推定實際 container 已停止；M0–M3 不自動跨 Node failover，也不因 offline 釋放已派工 reservation。重連需 current generation report、新健康證據與 MCP discovery 才恢復 availability。Production mTLS automation 延後，但跨主機 TLS server verification、per-node credential 與 revoke 不延後。

Fencing 無法取消最後一次合法 heartbeat 後已開始的 Docker side effect；平台保證舊回報不能覆寫新 intent，並透過下一輪收斂處理競爭。Agent crash 對已開始 restart 的結果若無法證明，回 IndeterminateExecution 並 inspect，不盲目再次 restart；本決策不宣稱外部操作 exactly-once。

## Rejected alternatives

| Alternative | 拒絕理由 |
| --- | --- |
| Registry 透過 SSH/inbound RPC push | 擴大 host 存取權與暴露面，需管理跨 Node 的 inbound connectivity，不能簡化 lifecycle 一致性。 |
| Portal 直接操作 Docker daemon | 將 host 管理權交給 caller，無法維持 per-node ownership、generation fencing 與 audit。 |
| 把 restart 當第三種 desiredState | 永久 intent 無法區分已執行與待重試的一次性行為；重送可能反覆 restart。 |
| Heartbeat 斷線就另起 instance | 未證明原 instance 已停止，可能雙重執行與衝突寫入；初版採保留 assignment 並 fail closed。 |
| 保證 exactly-once 的 transport | transport 去重不能與 Docker side effect 原子提交；採 durable journal、保守失敗與可觀測 reconciliation。 |

此決策使用 [domain model](../domain-model.md) 的 states/generation，並與 [ADR 0001](0001-registry-modular-monolith.md) 的持久 intent／交易邊界一致。
