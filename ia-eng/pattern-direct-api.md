# Padrão API Direta — Integração de LLM em Aplicações

## Estrutura de integração production-ready

```python
# Camada de abstração recomendada — não chame APIs de LLM diretamente no código de negócio

import anthropic
from tenacity import retry, stop_after_attempt, wait_exponential
import structlog
from typing import Optional
import time

log = structlog.get_logger()

class LLMClient:
    """
    Wrapper production-ready para APIs de LLM.
    Inclui: retry, logging, custo estimado, timeout, fallback.
    """
    
    def __init__(self):
        self.anthropic = anthropic.Anthropic()
        self.default_model = "claude-haiku-3-5"  # comece pelo mais barato
    
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=30),
        reraise=True
    )
    async def complete(
        self,
        messages: list[dict],
        system: str = "",
        model: Optional[str] = None,
        max_tokens: int = 1024,
        temperature: float = 0.0,  # determinístico por padrão para produção
        request_id: Optional[str] = None,
    ) -> dict:
        model = model or self.default_model
        start_time = time.time()
        
        try:
            response = self.anthropic.messages.create(
                model=model,
                max_tokens=max_tokens,
                temperature=temperature,
                system=system,
                messages=messages,
            )
            
            latency_ms = int((time.time() - start_time) * 1000)
            cost = self._estimate_cost(model, response.usage)
            
            # Log estruturado para observabilidade
            log.info("llm_call_success",
                request_id=request_id,
                model=model,
                input_tokens=response.usage.input_tokens,
                output_tokens=response.usage.output_tokens,
                estimated_cost_usd=cost,
                latency_ms=latency_ms,
            )
            
            return {
                "content": response.content[0].text,
                "model": model,
                "usage": response.usage,
                "cost_usd": cost,
                "latency_ms": latency_ms,
            }
            
        except anthropic.RateLimitError:
            log.warning("llm_rate_limit", model=model, request_id=request_id)
            raise  # tenacity vai fazer retry
            
        except anthropic.APIError as e:
            log.error("llm_api_error", model=model, error=str(e), request_id=request_id)
            raise
    
    def _estimate_cost(self, model: str, usage) -> float:
        # Preços em USD por 1M tokens (mai/2026 — atualizar periodicamente)
        prices = {
            "claude-haiku-3-5":   {"input": 0.80,  "output": 4.00},
            "claude-sonnet-4-5":  {"input": 3.00,  "output": 15.00},
            "claude-opus-4-5":    {"input": 15.00, "output": 75.00},
        }
        p = prices.get(model, prices["claude-sonnet-4-5"])
        return (usage.input_tokens * p["input"] + usage.output_tokens * p["output"]) / 1_000_000
```

## Structured Outputs — JSON confiável

```python
import json
from pydantic import BaseModel

class SentimentResult(BaseModel):
    sentiment: str  # "positive" | "negative" | "neutral"
    confidence: float  # 0.0 a 1.0
    reasoning: str

async def classify_sentiment(text: str) -> SentimentResult:
    """
    Classificação com output estruturado.
    Mais confiável que pedir ao LLM para formatar livremente.
    """
    response = await llm_client.complete(
        model="claude-haiku-3-5",  # tarefa simples → modelo barato
        system="""Você é um classificador de sentimento.
        Responda APENAS com JSON válido no formato:
        {"sentiment": "positive|negative|neutral", "confidence": 0.0-1.0, "reasoning": "string"}
        Não inclua markdown, texto antes ou depois do JSON.""",
        messages=[{"role": "user", "content": f"Classifique: {text}"}],
        max_tokens=150,
        temperature=0.0,  # determinístico para classificação
    )
    
    # Parse seguro — nunca assuma que o LLM seguiu o formato
    try:
        raw = response["content"].strip()
        # Remove markdown se o LLM incluiu por engano
        if raw.startswith("```"):
            raw = raw.split("```")[1].lstrip("json").strip()
        data = json.loads(raw)
        return SentimentResult(**data)
    except (json.JSONDecodeError, ValueError) as e:
        log.error("llm_output_parse_error", error=str(e), raw=response["content"])
        # Fallback gracioso
        return SentimentResult(sentiment="neutral", confidence=0.0, reasoning="parse error")
```

## Streaming — para UX responsiva

```python
async def stream_response(user_message: str):
    """
    Streaming é essencial para chatbots — usuário vê a resposta sendo escrita.
    Reduz perceived latency em 3-5x mesmo que total_latency seja igual.
    """
    with anthropic.Anthropic().messages.stream(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": user_message}]
    ) as stream:
        for text in stream.text_stream:
            yield text  # envie via SSE (Server-Sent Events) para o frontend

# Frontend (TypeScript com fetch + SSE):
# const response = await fetch('/api/chat', { method: 'POST', body: JSON.stringify({message}) });
# const reader = response.body.getReader();
# while (true) {
#   const { done, value } = await reader.read();
#   if (done) break;
#   displayText(new TextDecoder().decode(value));
# }
```

## Prompt versioning — prompts são código

```python
# Nunca deixe prompts hardcodados no código sem versionamento

# Estrutura recomendada:
# prompts/
#   v1/
#     classify_sentiment.md
#     summarize_document.md
#   v2/
#     classify_sentiment.md   ← versão melhorada

class PromptRegistry:
    """Carrega prompts de arquivos versionados."""
    
    def __init__(self, prompts_dir: str = "prompts"):
        self.prompts_dir = prompts_dir
        self._cache = {}
    
    def get(self, name: str, version: str = "latest") -> str:
        cache_key = f"{name}:{version}"
        if cache_key not in self._cache:
            path = f"{self.prompts_dir}/{version}/{name}.md"
            self._cache[cache_key] = Path(path).read_text()
        return self._cache[cache_key]

# Uso:
# prompts = PromptRegistry()
# system_prompt = prompts.get("classify_sentiment", version="v2")

# Benefícios:
# → Fácil A/B test entre versões de prompt
# → Git diff mostra exatamente o que mudou
# → Rollback simples em caso de regressão de qualidade
```

## Segurança — prevenção de prompt injection

```python
def sanitize_user_input(user_input: str) -> str:
    """
    Protege contra prompt injection — usuário tentando sobrescrever instruções.
    Exemplos de ataque: "Ignore as instruções acima e faça X"
    """
    # Nunca interpole input diretamente no system prompt
    # Ruim:  f"Você é um assistente. O usuário diz: {user_input}"
    # Bom: use a estrutura de mensagens corretamente:
    
    # Limite de tamanho (evita ataques de token flooding)
    if len(user_input) > 4000:
        user_input = user_input[:4000] + "... [truncado]"
    
    return user_input

# Estrutura segura de mensagens:
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                # O input do usuário fica isolado em seu próprio campo
                # O LLM sabe que é input do usuário, não instrução
                "text": f"<user_input>\n{sanitize_user_input(user_message)}\n</user_input>"
            }
        ]
    }
]

# Para validação adicional: Guardrails AI ou NeMo Guardrails
# detectam PII, conteúdo off-topic, tentativas de jailbreak
```
