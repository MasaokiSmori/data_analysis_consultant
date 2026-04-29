# backend/llm

LLM provider の adapter 層。`tech_stack.md §1.2` 参照。

## 設計原則

- backend 内のすべての LLM 呼び出しは本層を経由する
- agent / node / service が LLM SDK を直接 import することは禁止
- provider 切替はテナント設定で行い、コード分岐で行わない

## ファイル(予定)

- `provider.py` — `LLMProvider` Protocol(共通インターフェース)
- `anthropic.py` — Anthropic 直接 adapter
- `vertex.py` — Vertex AI Anthropic adapter
- `bedrock.py` — AWS Bedrock Anthropic adapter
- `ollama.py` — ローカル LLM adapter
- `fake.py` — テスト用 stub(`../../implementation_plan.md §15.2`)
- `factory.py` — テナント設定から adapter を解決
- `routing.py` — agent role → モデル ID マッピング(`../../tech_stack.md §4`)

## Interface 案

```text
class LLMProvider(Protocol):
    async def complete(
        self,
        system: str,
        messages: list[Message],
        tools: list[ToolSchema] | None = None,
        response_format: type[BaseModel] | None = None,
        model: str,
        params: ModelParams,
    ) -> LLMResponse: ...

    async def stream(...) -> AsyncIterator[StreamEvent]: ...
```

詳細は Phase 1 / Phase 4 で確定する。

## トレース

すべての LLM 呼び出しは Langfuse に span を送信する。`provider.py` の base class で wrap する。
