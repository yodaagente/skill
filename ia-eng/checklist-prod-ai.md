# Checklist de Produção — Sistemas de IA

Execute este checklist antes de qualquer feature de IA ir a produção.

## Arquitetura e Design

```markdown
- [ ] Problema definido com clareza: IA é realmente necessária aqui?
- [ ] Alternativas avaliadas: ML clássico, regras, busca tradicional
- [ ] Modelo selecionado com justificativa de custo (menor modelo suficiente)
- [ ] Padrão de integração definido: API direta / RAG / agente / fine-tuning
- [ ] Fallback implementado: o que acontece se a API de LLM estiver down?
- [ ] SLA definido: latência máxima, disponibilidade mínima, qualidade mínima aceitável
```

## Custo e Escalabilidade

```markdown
- [ ] Estimativa de custo mensal calculada para volume esperado
- [ ] Budget alert configurado (80% e 100% do limite mensal)
- [ ] Prompt caching habilitado para system prompts > 1024 tokens (Anthropic)
- [ ] max_tokens definido em todas as chamadas (sem outputs ilimitados)
- [ ] Semantic cache implementado para workloads com queries repetitivas
- [ ] Batch API avaliada para processamentos assíncronos (50% de desconto)
- [ ] Break-even calculado: API vs. self-hosting para volume atual e projetado
```

## Qualidade e Avaliação

```markdown
- [ ] Golden test set criado (mínimo 50 exemplos representativos)
- [ ] Métricas de qualidade definidas antes do launch (não depois)
- [ ] LLM-as-judge implementado para avaliação contínua em produção
- [ ] Prompt versionado em repositório (tratado como código)
- [ ] A/B test planejado para validar melhorias de prompt
- [ ] Threshold de qualidade mínima definido para alerta/rollback
```

## Observabilidade

```markdown
- [ ] Logging de tokens e custo por request habilitado
- [ ] Distributed tracing configurado (LangSmith / Langfuse / Helicone)
- [ ] Dashboard com: cost/day, error_rate, latency_p95, quality_score
- [ ] Alerta: cost spike > 2x média 7 dias
- [ ] Alerta: error_rate > 2% por 5 minutos
- [ ] Alerta: quality_score < threshold por 1 hora
- [ ] Alerta: latência p95 > SLO definido
```

## Segurança e Privacidade

```markdown
- [ ] API keys em Secrets Manager / variáveis de ambiente (nunca no código)
- [ ] Rate limiting implementado antes de chamar API de LLM
- [ ] Input sanitization contra prompt injection
- [ ] Output validation: nunca usar output de LLM sem verificação
- [ ] PII mapeado: dados pessoais enviados ao LLM têm base legal e DPA?
- [ ] Para dados sensíveis: modelo local avaliado (Ollama)
- [ ] Structured outputs com schema validado (evita uso de dados malformados)
- [ ] Guardrails configurados para conteúdo off-topic / harmful
```

## Confiabilidade

```markdown
- [ ] Retry com backoff exponencial implementado (3 tentativas)
- [ ] Circuit breaker para falhas do provedor de LLM
- [ ] Timeout configurado em todas as chamadas (não deixar requests travados)
- [ ] Queue implementada para picos de tráfego (evitar explosão de custo)
- [ ] Degradação graciosa: o que o usuário vê se o LLM falhar?
- [ ] Incident runbook escrito para falhas de IA
```

## Para Agentes Especificamente

```markdown
- [ ] max_steps por sessão definido (evitar loops infinitos)
- [ ] max_cost por sessão definido
- [ ] Human-in-the-loop configurado para ações irreversíveis
- [ ] Todas as ferramentas têm tratamento de erro explícito
- [ ] Tracing de multi-step configurado (LangSmith / Langfuse)
- [ ] Task completion rate monitorado (benchmark: 30-35% em tarefas complexas)
- [ ] Guardrails de segurança implementados por ferramenta
```

## Para RAG Especificamente

```markdown
- [ ] Chunking strategy definida e testada (parent-child recomendado)
- [ ] Hybrid search implementado (vector + BM25) — não só semântico
- [ ] Reranker configurado (cross-encoder para top-K final)
- [ ] Semantic cache implementado (threshold cosine: 0.92)
- [ ] Retrieval score monitorado (precision@K e recall@K no golden set)
- [ ] Multi-tenant: isolamento de dados por tenant verificado
- [ ] Freshness do corpus monitorado (alertar se dados > SLA sem atualização)
```
