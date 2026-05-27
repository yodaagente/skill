# Otimização de Custo de LLM — Guia Completo

## Os 5 maiores desperdiçadores de custo (e como resolver)

### 1. Modelo errado para a tarefa
```
Problema: usar Claude Sonnet para classificar sentimento de texto
Custo: $3.00/1M tokens de input

Solução: Claude Haiku para classificação simples
Custo: $0.80/1M tokens → redução de 73%

Regra: teste o modelo mais barato primeiro.
Escale apenas se a qualidade for insuficiente.
```

### 2. Prompts inflados sem necessidade
```
Problema: system prompt de 2.000 tokens em toda requisição
Custo em 100k req/dia: 100k × 2.000 = 200M tokens/dia

Soluções:
a) Prompt compression — remover redundâncias, exemplos excessivos
   → redução típica de 20-40% no tamanho do prompt
   
b) Cached prompts (Anthropic Prompt Caching)
   → System prompts > 1.024 tokens podem ser cacheados
   → Cache hit: 90% de desconto no custo de input
   → Condição: 5min de TTL, prefixo idêntico entre requests
   
c) Instrução concisa > instrução verbosa
   Ruim:  "Por favor, analise cuidadosamente o texto a seguir e me diga..."
   Bom:   "Analise o texto:"
```

### 3. Output tokens excessivos
```
Problema: LLM gerando 800 tokens quando a resposta útil tem 100
Custo: output tokens custam 3-5x mais que input tokens

Soluções:
a) Instruir o modelo explicitamente:
   "Responda em no máximo 3 frases." / "Retorne apenas o JSON."
   
b) max_tokens: defina um limite razoável sempre
   response = client.messages.create(max_tokens=256, ...)
   
c) Structured outputs: JSON com schema reduz verbosidade
   → Anthropic: response_format com JSON Schema
   → Substitui outputs descritivos por campos concisos
```

### 4. Chamadas sem cache
```
Problema: mesmas queries sendo processadas repetidamente
Exemplo: 70% das queries de suporte são variações de 20 perguntas frequentes

Solução: cache em camadas
  L1: Cache exato (Redis, TTL 1h)      → hit rate ~15%
  L2: Cache semântico (cosine ≥ 0.92)  → hit rate adicional ~30-50%
  L3: Resposta do LLM                  → apenas queries genuinamente novas

Resultado: 30-68% de redução de chamadas à API em workloads de suporte/FAQ
```

### 5. Processamento síncrono onde batch é possível
```
Problema: processar 10.000 documentos um a um via API síncrona
Custo: preço normal por token

Solução: Batch API (Anthropic e OpenAI)
  → 50% de desconto automático no preço
  → Ideal para: análise de documentos, categorização, geração de conteúdo em lote
  → Limite: SLA de 24h (não para uso interativo)

Código com Anthropic Batch API:
import anthropic

client = anthropic.Anthropic()

batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"doc-{i}",
            "params": {
                "model": "claude-haiku-3-5",
                "max_tokens": 256,
                "messages": [{"role": "user", "content": f"Classifique: {doc}"}]
            }
        }
        for i, doc in enumerate(documentos)
    ]
)
# Resultado disponível em até 24h → 50% mais barato
```

## Prompt Caching (Anthropic) — implementação

```python
import anthropic

client = anthropic.Anthropic()

# System prompts grandes (> 1024 tokens) se beneficiam enormemente de cache
# Cache hit = 90% de desconto no custo de input daquele bloco

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": """Você é um especialista em suporte técnico da N&DC Consultoria.
            
            [... 2.000 tokens de contexto da empresa, produtos, políticas ...]
            """,
            "cache_control": {"type": "ephemeral"}  # ← habilita cache por 5min
        }
    ],
    messages=[{"role": "user", "content": user_question}]
)

# Primeira chamada: paga input normal → escreve o cache
# Chamadas seguintes (5min): paga apenas 10% do input do bloco cacheado
# → Se o system prompt tem 2.000 tokens: economia de 1.800 tokens/request
# → Em 100k req/dia: 180M tokens economizados × $3/1M = $540/dia de economia
```

## Dashboard de custo — o que monitorar

```python
# Estrutura de log por request (salvar no seu banco de dados/observabilidade)
{
    "request_id": "uuid",
    "timestamp": "2026-05-12T10:30:00Z",
    "model": "claude-haiku-3-5",
    "prompt_version": "v2.3",
    "feature": "support-chat",
    "user_id": "user_123",
    "tenant_id": "tenant_abc",
    
    # Tokens e custo
    "input_tokens": 1250,
    "output_tokens": 312,
    "cache_read_tokens": 900,      # tokens lidos do cache (custo 90% menor)
    "cache_write_tokens": 1000,    # tokens escritos no cache
    "estimated_cost_usd": 0.00187,
    
    # Performance
    "ttft_ms": 380,                # time to first token
    "total_latency_ms": 1240,
    "cache_hit": true,             # hit no semantic cache
    
    # Qualidade
    "quality_score": 4.2,          # LLM-as-judge
    "user_feedback": null,         # rating do usuário se disponível
    "fallback_used": false
}

# Queries de alerta:
# custo_dia > budget_diario × 0.8 → alerta preventivo
# custo_por_request > media_7d × 2 → spike inesperado
# cache_hit_rate < 20% → oportunidade de caching perdida
# quality_score < 3.5 → degradação de qualidade
```

## Estimativa de custo por funcionalidade

```
Calculadora rápida (Sonnet $3 input / $15 output por 1M tokens):

Chatbot de suporte (turno médio: 800 input + 300 output tokens):
  → Custo por turno: 800/1M × $3 + 300/1M × $15 = $0.0024 + $0.0045 = $0.0069
  → 10.000 usuários × 5 turnos/mês = 50.000 turnos/mês
  → Custo mensal sem otimização: 50.000 × $0.0069 = $345/mês
  → Com semantic cache (50% hit): $172/mês
  → Com Haiku + cache: ~$55/mês (84% de redução)

Análise de documentos (2.000 tokens médios input + 500 output):
  → Custo por doc: $0.006 + $0.0075 = $0.0135
  → 5.000 docs/mês = $67.50/mês
  → Com Batch API (50% desconto): $33.75/mês
  → Com Haiku: ~$10/mês
```
