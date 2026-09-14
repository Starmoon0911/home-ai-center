# ADR 0004：LiteLLM logical model aliases

- Status：Accepted
- Date：2026-09-14
- Scope：Portal、Orchestrator 與所有 LLM callers 的模型存取邊界；M5 chat/streaming，M7 embedding 擴充。

## Context

本機開發只有 Apple Silicon `dev-macbook`；未來模型可能移到 vLLM 或專用 GPU host。若 Portal／Orchestrator 依賴實際模型名稱、host IP 或 provider endpoint，每次換模型／placement 都會改上層程式，也容易外洩 gateway credentials 或繞過授權。平台需要穩定 caller contract，backend routing 則由受限部署設定管理。

## Decision

所有 LLM callers 只使用 LiteLLM logical aliases。Portal 與 Orchestrator 只能使用已核准的 `home-large`、`home-fast`；不得直接連 Ollama、vLLM 或實體 GPU node。瀏覽器呼叫 Orchestrator，gateway credential 留在 backend；只有 LiteLLM 解析實際 inference endpoint。未知 alias 拒絕，不提供 wildcard、任意 provider passthrough 或失敗後 direct backend fallback。未來增加 alias 需明確納入契約及授權。

本機兩個 alias 固定如下；alias 名稱不構成模型大小、品質或速度保證：

| Caller alias | LiteLLM backend mapping | Backend address |
| --- | --- | --- |
| `home-large` | `ollama_chat/phi3:3.8b` | 由 LiteLLM 的 `OLLAMA_API_BASE` 解析。 |
| `home-fast` | `ollama_chat/phi3:3.8b` | 同一 host Ollama，同一環境變數。 |

`phi3:3.8b` 是本機固定 prerequisite，tag 由 [Ollama Phi-3 tags](https://ollama.com/library/phi3/tags)確認；不使用隱含 latest，不為本文件改選其他模型。`ollama_chat/` 使用 LiteLLM 的 Ollama chat integration，chat/streaming 實測在 M5 完成；不從 provider 支援推論此模型已有可用 tool calling。[LiteLLM Ollama integration](https://docs.litellm.ai/docs/providers/ollama)

LiteLLM `model_name` 提供 caller 名稱，`litellm_params` 保存 backend mapping；本專案透過 `api_base: os.environ/OLLAMA_API_BASE` 注入 endpoint。這是 gateway 部署設定，不能寫進 `ServiceDefinition`，也不交給 Portal 組合 host URL。[LiteLLM proxy configuration](https://docs.litellm.ai/docs/proxy/configs) Host 與 Docker Desktop 的環境值及最小 YAML 見 [development](../development.md#ollama-與-litellm-設定邊界)。

M0–M3 先固定此 access boundary，不要求啟動 inference。M4 先驗證獨立於模型的 MCP discovery/authorization；M5 以固定 Phi-3 實測兩 aliases 的 chat/streaming；M6 再接通 Portal／Orchestrator。Embedding alias 與模型延至 M7，不把 Phi-3 chat mapping 當作 embedding capability。未來 GPU backend 切換保持 callers 的 aliases，改 gateway 的核准 mapping 並重新驗證，順序見 [roadmap](../roadmap.md)。

## Consequences

上層使用穩定名稱，模型搬遷／替換可由部署側完成；backend credentials、位址及 provider 差異集中在 LiteLLM 邊界。測試可分開確認平台授權與實際 inference，不把小模型的生成品質當作控制面正確性。

LiteLLM 成為 inference 路徑的必要元件；它或 backend 失效時 caller 得到受控錯誤，不能繞路。兩個本機 aliases 共用一個模型與 host，因此沒有容量或故障隔離。新增模型仍需驗證 request/stream/error compatibility、授權與容量；logical alias 不會消除模型能力差異。Registry capability availability/permission 與模型 tool proposal 是不同契約，模型輸出永不建立 tool grant。

## Rejected alternatives

| Alternative | 拒絕理由 |
| --- | --- |
| Portal／Orchestrator 直接呼叫 Ollama/vLLM | 固化 provider/host 依賴，繞過 gateway credential 與 routing boundary。 |
| Caller 使用 phi3:3.8b 或 GPU host name 作 model | 將部署細節變成 public contract，之後搬遷須改上層。 |
| 本機為兩 aliases 下載不同模型 | 增加開發 prerequisite，與固定單一 Phi-3 baseline 不符；目前不要求品質或效能分級。 |
| Wildcard model routing 或 backend fallback | Caller 可繞過核准 alias 清單，錯誤時破壞存取邊界。 |
| M0 即要求真實 chat/embedding 通過 | 文件交付被 runtime/download 綁住；embedding 屬 M7，chat/streaming 屬 M5。 |

本決策延續 [架構 LLM flow](../architecture.md#llm-request-flow)、[security model](../security.md)與 [ADR 0003](0003-separate-manifest-and-deployment.md) 的宣告／部署分離。
