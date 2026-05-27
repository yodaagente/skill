# Seleção de Modelos — Guia de Decisão

## Framework de decisão

```
Passo 1: A tarefa realmente precisa de LLM?
  → Classificação simples (< 10 classes): ML clássico é mais rápido e barato
  → Busca: Elasticsearch/BM25 resolve sem LLM
  → Regex/rules: regex ainda bate LLM em precisão para padrões fixos
  → Se sim, continue:

Passo 2: Qual a complexidade da tarefa?
  → Simples (classificar, extrair campo, resumir < 500 palavras): modelo pequeno
  → Médio (analisar, comparar, gerar conteúdo estruturado): modelo médio
  → Complexo (raciocinar em múltiplos passos, código, análise longa): modelo grande

Passo 3: Quais são os requisitos não-funcionais?
  → Latência < 200ms: modelo local ou API com streaming
  → Privacidade total: modelo local obrigatório
  → Volume > 100k req/dia: calcule break-even de self-hosting
  → Custo crítico: comece com modelo pequeno e suba apenas se necessário
```

## Tabela comparativa de modelos (mai/2026)

| Modelo | Input ($/1M tk) | Output ($/1M tk) | Contexto | Melhor para |
|---|---|---|---|---|
| Claude Haiku 3.5 | $0.80 | $4.00 | 200k | Tarefas rápidas, classificação, extração |
| Claude Sonnet 4.5 | $3.00 | $15.00 | 200k | Uso geral, código, análise |
| Claude Opus 4.5 | $15.00 | $75.00 | 200k | Raciocínio complexo, decisões críticas |
| GPT-4o mini | $0.15 | $0.60 | 128k | Tarefas simples, alto volume |
| GPT-4o | $2.50 | $10.00 | 128k | Uso geral, multimodal |
| Gemini 2.0 Flash | $0.10 | $0.40 | 1M | Alto volume, contexto longo barato |
| Gemini 2.5 Pro | $1.25 | $10.00 | 1M | Contexto muito longo, raciocínio |
| Llama 3.3 70B (local) | ~$0 | ~$0 | 128k | Alto volume, privacidade, sem latência de rede |
| Qwen 2.5 7B (local) | ~$0 | ~$0 | 128k | MVP local, Ollama, VPS modesto |

> Preços aproximados — verificar sempre na documentação oficial do provedor.

## Estratégia de Model Cascade (roteamento por complexidade)

Economize 60-80% roteando requisições para o modelo mais barato que resolve:

```python
class ModelRouter:
    """
    Roteia cada requisição para o modelo mais barato suficiente.
    Estratégia: tente o pequeno → se falhar, escale para o médio → grande.
    """
    
    async def route(self, task: str, context: dict) -> str:
        # 1. Classifique a complexidade da tarefa (modelo de classificação local)
        complexity = await self.classify_complexity(task)
        
        if complexity == "simple":
            result = await self.call_model("claude-haiku-3-5", task, context)
            if self.is_quality_ok(result):
                return result
            # Se falhou, escale
                
        if complexity in ("simple", "medium"):
            result = await self.call_model("claude-sonnet-4-5", task, context)
            if self.is_quality_ok(result):
                return result
        
        # Fallback final para modelo grande
        return await self.call_model("claude-opus-4-5", task, context)
    
    def is_quality_ok(self, result: str) -> bool:
        # Implemente verificações: comprimento mínimo, JSON válido, sem recusa, etc.
        return len(result) > 10 and "I cannot" not in result
```

## Modelos locais com Ollama — quando e como

```bash
# Quando usar modelo local:
# ✅ Dados sensíveis (PII, dados médicos, financeiros)
# ✅ Volume > 500k req/mês (break-even vs. API)
# ✅ Latência de rede inaceitável (< 100ms necessário)
# ✅ Sem acesso à internet no ambiente de deploy

# Modelos recomendados para Ollama (mai/2026):
# qwen2.5:7b    → melhor custo-benefício para tarefas gerais (4GB RAM)
# qwen2.5:14b   → qualidade próxima a modelos médios da API (8GB RAM)  
# llama3.2:3b   → ultra-leve para tarefas simples (2GB RAM)
# llama3.3:70b  → qualidade de modelo grande, exige GPU (40GB VRAM)
# mistral:7b    → bom para código e inglês (4GB RAM)

# Instalação e uso básico
ollama pull qwen2.5:7b
ollama run qwen2.5:7b

# Integração via LiteLLM (API compatível com OpenAI)
import litellm
response = litellm.completion(
    model="ollama/qwen2.5:7b",
    messages=[{"role": "user", "content": "Classifique este texto: ..."}],
    api_base="http://localhost:11434"
)
```

## Cálculo de break-even: API vs. self-hosting

```python
def calcular_breakeven(
    req_por_mes: int,
    tokens_por_req: int,
    custo_api_por_1m_tokens: float,  # ex: $3.00 para Sonnet
    custo_servidor_mes: float,        # ex: $500 para VPS com GPU
) -> dict:
    custo_api_mes = (req_por_mes * tokens_por_req / 1_000_000) * custo_api_por_1m_tokens
    
    return {
        "custo_api_mes": custo_api_mes,
        "custo_self_host_mes": custo_servidor_mes,
        "break_even": custo_servidor_mes < custo_api_mes,
        "economia_mensal": max(0, custo_api_mes - custo_servidor_mes),
    }

# Exemplo: 500k req/mês, 1000 tokens médios, Sonnet ($3/1M)
# custo_api = 500k * 1000 / 1M * $3 = $1.500/mês
# custo_servidor GPU (A10G) = ~$600/mês
# → self-hosting economiza $900/mês → break-even atingido
```
