# Padrão RAG — Retrieval-Augmented Generation para Produção

## Quando usar RAG vs. alternativas

```
Use RAG quando:
✅ Base de conhecimento > 200k tokens (não cabe no contexto)
✅ Dados mudam frequentemente (documentos, preços, políticas)
✅ Precisão de citação é obrigatória (compliance, legal, médico)
✅ Múltiplos usuários com bases de conhecimento diferentes (multi-tenant)

Prefira Few-Shot quando:
→ Base de conhecimento pequena e estável (< 20 exemplos)
→ O problema é de formato/estilo, não de conhecimento
→ RAG adicionaria latência inaceitável

Prefira Fine-tuning quando:
→ Comportamento consistente em domínio específico (tom, formato, terminologia)
→ Tarefa bem definida com > 1000 exemplos de qualidade
→ RAG com base pequena não melhora qualidade suficientemente
```

## Arquitetura de produção — pipeline completo

```
INDEXAÇÃO (offline, assíncrono)
─────────────────────────────────────────────────────
Documentos (PDF, Markdown, HTML, DB)
  ↓
Preprocessamento (limpeza, normalização, metadados)
  ↓
Chunking estratégico (ver estratégias abaixo)
  ↓
Geração de embeddings (modelo de embedding)
  ↓
Vector Store (Qdrant / pgvector / Pinecone)
  + índice BM25 (Elasticsearch / Typesense)

QUERY (online, síncrono — path crítico de latência)
─────────────────────────────────────────────────────
Query do usuário
  ↓
[Cache semântico] ← 30-68% hit rate em FAQ/suporte
  ↓ (cache miss)
Query transformation (HyDE, multi-query, decomposição)
  ↓
Hybrid Search (Vector + BM25) → Reciprocal Rank Fusion
  ↓
Reranking (cross-encoder) → top-K chunks
  ↓
Construção do contexto (parent chunks + metadados)
  ↓
LLM → resposta fundamentada
  ↓
Citações + validação de output
```

## Estratégias de chunking

```python
# ✅ Recomendado: Parent-Child Chunking
# Child chunks (128 tokens): busca precisa e específica
# Parent chunks (512 tokens): contexto rico para o LLM

from langchain.text_splitter import RecursiveCharacterTextSplitter

child_splitter = RecursiveCharacterTextSplitter(
    chunk_size=128,
    chunk_overlap=20,
    separators=["\n\n", "\n", ".", " "]
)

parent_splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50
)

# Indexe child chunks → ao recuperar, retorne o parent correspondente
# Resultado: precisão de retrieval + contexto suficiente para o LLM

# Outras estratégias:
# Semantic chunking: divide por mudança de tópico (9% mais recall que fixed-size)
# Sliding window: chunks com overlap alto para documentos técnicos
# Document structure: respeita cabeçalhos, seções, parágrafos (melhor para docs estruturados)
```

## Hybrid Search — implementação

```python
from qdrant_client import QdrantClient
from rank_bm25 import BM25Okapi

class HybridSearchEngine:
    """
    Combina busca vetorial (semântica) com BM25 (keyword).
    Melhora recall em 1-9% vs. busca vetorial pura.
    Essencial para: códigos de produto, nomes próprios, siglas.
    """
    
    def __init__(self, vector_store: QdrantClient, bm25_index: BM25Okapi):
        self.vector_store = vector_store
        self.bm25 = bm25_index
    
    def search(self, query: str, top_k: int = 20) -> list[dict]:
        # 1. Busca vetorial
        query_embedding = embed(query)
        vector_results = self.vector_store.search(
            collection_name="docs",
            query_vector=query_embedding,
            limit=top_k
        )
        
        # 2. Busca BM25 (keyword)
        tokens = query.lower().split()
        bm25_scores = self.bm25.get_scores(tokens)
        bm25_results = sorted(
            enumerate(bm25_scores), key=lambda x: x[1], reverse=True
        )[:top_k]
        
        # 3. Reciprocal Rank Fusion (combina os scores)
        return self._reciprocal_rank_fusion(vector_results, bm25_results, k=60)
    
    def _reciprocal_rank_fusion(self, list1, list2, k=60):
        scores = {}
        for rank, doc in enumerate(list1):
            scores[doc.id] = scores.get(doc.id, 0) + 1 / (k + rank + 1)
        for rank, (doc_id, _) in enumerate(list2):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
        return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

## Cache semântico — 30-68% de redução de custo

```python
import numpy as np
from redis import Redis

class SemanticCache:
    """
    Armazena respostas de queries similares.
    Threshold 0.92 de cosine similarity = boa precisão sem falsos positivos.
    Produz 30-68% de hit rate em workloads de FAQ e suporte.
    """
    
    SIMILARITY_THRESHOLD = 0.92
    TTL_SECONDS = 3600  # 1 hora (ajustar conforme staleness aceitável)
    
    def __init__(self, redis: Redis):
        self.redis = redis
    
    def get(self, query: str) -> str | None:
        query_embedding = embed(query)
        
        # Busca em cache por similaridade
        cached_keys = self.redis.keys("cache:embedding:*")
        for key in cached_keys:
            cached_embedding = np.frombuffer(self.redis.get(key))
            similarity = cosine_similarity(query_embedding, cached_embedding)
            
            if similarity >= self.SIMILARITY_THRESHOLD:
                response_key = key.replace("embedding:", "response:")
                return self.redis.get(response_key)
        
        return None
    
    def set(self, query: str, response: str, query_id: str):
        query_embedding = embed(query)
        self.redis.setex(f"cache:embedding:{query_id}", self.TTL_SECONDS,
                         query_embedding.tobytes())
        self.redis.setex(f"cache:response:{query_id}", self.TTL_SECONDS, response)
```

## Seleção de vector database

| Banco | Melhor para | Self-host | Managed | Destaque |
|---|---|---|---|---|
| **pgvector** | Apps já com PostgreSQL, volume moderado | ✅ | ✅ (Supabase, Neon) | Sem infra extra — SQL nativo |
| **Qdrant** | Performance, filtros complexos, Rust | ✅ | ✅ | Melhor custo-performance em 2025 |
| **Pinecone** | Managed sem ops, protótipos rápidos | ❌ | ✅ | Simplicidade máxima |
| **Weaviate** | Multi-tenant, GraphQL, hybrid nativo | ✅ | ✅ | Bom para multimodal |
| **Milvus** | Bilhões de vetores, escala extrema | ✅ | ✅ | Melhor para escala muito grande |

**Recomendação para 90% dos casos**: `pgvector` se já usa PostgreSQL, `Qdrant` se precisa de performance dedicada.

## Avaliação de qualidade do RAG

```python
# Métricas essenciais para monitorar em produção:

# 1. Retrieval Score (offline — golden test set)
# Precision@K: dos K chunks recuperados, quantos são relevantes?
# Recall@K: dos chunks relevantes totais, quantos foram recuperados?

# 2. Generation Quality (online — LLM-as-judge)
async def evaluate_rag_response(question, context, answer):
    judge_prompt = f"""
    Avalie a resposta abaixo em 3 dimensões (score 1-5 cada):
    
    Pergunta: {question}
    Contexto recuperado: {context}
    Resposta gerada: {answer}
    
    1. Faithfulness: a resposta está fundamentada no contexto? (sem alucinações)
    2. Relevance: a resposta responde à pergunta?
    3. Completeness: a resposta está completa dado o contexto disponível?
    
    Responda em JSON: {{"faithfulness": X, "relevance": X, "completeness": X}}
    """
    return await llm.complete(judge_prompt, response_format={"type": "json_object"})

# Alerte se faithfulness < 4.0 em média por dia → possível drift do corpus
```

## Multi-tenant RAG — isolamento de dados

```python
# CRÍTICO: nunca misture dados de tenants diferentes no mesmo namespace

# Opção A (recomendada): filtro por metadata — dados ficam juntos, acesso isolado
results = vector_store.search(
    query_vector=embedding,
    filter={"tenant_id": current_user.tenant_id},  # garante isolamento
    limit=10
)

# Opção B: collections separadas por tenant (melhor performance, mais ops)
collection_name = f"tenant_{tenant_id}_documents"

# Nunca: buscar em toda a coleção e filtrar no código (dados vazam entre tenants)
```
