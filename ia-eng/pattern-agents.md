# Padrão Agentes — LLM + Ferramentas + Autonomia

## Quando usar agentes vs. alternativas

```
Use agentes quando:
✅ A tarefa requer múltiplos passos com decisões intermediárias
✅ Ferramentas externas precisam ser chamadas dinamicamente
✅ O caminho até a solução não é conhecido com antecedência
✅ Human-in-the-loop é necessário em pontos específicos

NÃO use agentes quando:
❌ Pipeline simples e previsível resolve (use chains simples)
❌ Confiabilidade > 99% é exigida (agentes completam 30-35% de tarefas complexas)
❌ Latência < 500ms é necessária (agentes têm múltiplos round-trips)
❌ Auditoria detalhada de cada decisão é obrigatória sem infraestrutura de tracing
```

## Framework — LangGraph (recomendado para produção)

LangGraph emergiu como líder em produção em 2025, superando AutoGen e CrewAI em adoção e confiabilidade.

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_anthropic import ChatAnthropic
from typing import TypedDict, Annotated
import operator

# 1. Defina o estado do agente
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    user_id: str
    context: dict
    attempts: int

# 2. Defina as ferramentas disponíveis
from langchain_core.tools import tool

@tool
def buscar_pedidos(user_id: str, status: str = None) -> list[dict]:
    """Busca pedidos do usuário no banco de dados."""
    return order_repository.find_by_user(user_id, status=status)

@tool
def cancelar_pedido(pedido_id: str, motivo: str) -> dict:
    """Cancela um pedido específico. Requer confirmação humana para pedidos > R$500."""
    pedido = order_repository.find(pedido_id)
    if pedido["valor"] > 500:
        raise ValueError("HUMAN_APPROVAL_REQUIRED")  # interrupt para revisão humana
    return order_repository.cancel(pedido_id, motivo)

tools = [buscar_pedidos, cancelar_pedido]

# 3. Configure o modelo
llm = ChatAnthropic(model="claude-sonnet-4-5").bind_tools(tools)

# 4. Defina os nós do grafo
def agent_node(state: AgentState) -> AgentState:
    """Nó principal: LLM decide qual ferramenta chamar."""
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

def should_continue(state: AgentState) -> str:
    """Roteamento: continuar com ferramentas ou finalizar."""
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"
    return END

# 5. Monte o grafo
graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", ToolNode(tools))

graph.set_entry_point("agent")
graph.add_conditional_edges("agent", should_continue)
graph.add_edge("tools", "agent")  # após executar ferramenta, volta ao agente

app = graph.compile()

# 6. Execute
result = await app.ainvoke({
    "messages": [{"role": "user", "content": "Quero cancelar meu último pedido"}],
    "user_id": "user_123",
    "context": {},
    "attempts": 0
})
```

## Human-in-the-Loop (HITL) — para ações irreversíveis

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import interrupt

# Adicione persistência de estado para pausar e retomar
checkpointer = MemorySaver()

def cancelar_pedido_com_aprovacao(pedido_id: str, valor: float) -> dict:
    """Cancela pedido — exige aprovação humana se valor > R$500."""
    if valor > 500:
        # Pausa o agente e aguarda input humano
        human_decision = interrupt({
            "type": "approval_required",
            "message": f"Cancelar pedido {pedido_id} no valor de R${valor:.2f}?",
            "actions": ["aprovar", "rejeitar"]
        })
        if human_decision != "aprovar":
            return {"status": "cancelado_pelo_operador"}
    
    return order_repository.cancel(pedido_id)

# No frontend: receba o interrupt → mostre ao operador → retome com a decisão
# await app.ainvoke(Command(resume="aprovar"), config={"thread_id": thread_id})
```

## Multi-Agent — padrão Supervisor

```python
# Para tarefas complexas que requerem especialistas diferentes

from langgraph_supervisor import create_supervisor

# Agentes especializados
research_agent = create_agent(tools=[web_search, read_url], 
                              system="Especialista em pesquisa")
analysis_agent = create_agent(tools=[run_code, query_db],
                              system="Especialista em análise de dados")
writer_agent   = create_agent(tools=[save_document],
                              system="Especialista em redação")

# Supervisor coordena qual agente usa para cada subtarefa
supervisor = create_supervisor(
    agents=[research_agent, analysis_agent, writer_agent],
    model=ChatAnthropic(model="claude-sonnet-4-5"),
    prompt="Coordene os especialistas para completar a tarefa do usuário."
)

# Atenção: cada nível de agente adiciona latência e custo
# Benchmark CMU (2025): agentes completam 30-35% de tarefas multi-passo
# → Use supervisor apenas quando complexidade justifica
```

## MCP (Model Context Protocol) — integração padronizada

```python
# MCP é o protocolo aberto adotado por Anthropic, OpenAI, Google, Microsoft
# Permite que agentes se conectem a qualquer ferramenta via interface padronizada

# Exemplo: conectar agente a servidor MCP do Google Drive
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    tools=[
        {
            "type": "mcp",
            "server": {
                "type": "url",
                "url": "https://drivemcp.googleapis.com/mcp/v1",
                "name": "google-drive"
            }
        }
    ],
    messages=[{
        "role": "user",
        "content": "Liste os arquivos da pasta 'Projetos' no meu Drive"
    }]
)
```

## Observabilidade de agentes — o que rastrear

```python
# Agentes são difíceis de debugar sem tracing adequado
# Use LangSmith, Langfuse, ou Arize Phoenix desde o dia 1

# O que rastrear por execução de agente:
{
    "trace_id": "uuid",
    "agent_name": "support-agent",
    "total_steps": 4,
    "total_tokens": 3200,
    "total_cost_usd": 0.048,
    "total_latency_ms": 4500,
    "success": true,
    "steps": [
        {
            "step": 1,
            "type": "llm_call",
            "model": "claude-sonnet-4-5",
            "tokens": 800,
            "decision": "call_tool:buscar_pedidos"
        },
        {
            "step": 2,
            "type": "tool_call",
            "tool": "buscar_pedidos",
            "latency_ms": 120,
            "success": true
        },
        # ...
    ]
}

# Alertas críticos para agentes em produção:
# - task_completion_rate < 80% por janela de 1h
# - steps_per_task > max_steps_esperado (loop detectado)
# - custo_por_tarefa > 2x média histórica
# - tool_error_rate > 5% (dependência externa instável)
```

## Guardrails — controles de segurança

```python
# Nunca execute ações irreversíveis sem guardrails

class AgentGuardrails:
    # Lista de ferramentas que requerem aprovação humana
    REQUIRE_HUMAN_APPROVAL = {
        "cancelar_pedido",
        "enviar_email_em_massa", 
        "deletar_registro",
        "executar_pagamento"
    }
    
    # Limites de custo por sessão de agente
    MAX_COST_PER_SESSION_USD = 0.50
    MAX_STEPS_PER_SESSION = 20
    
    def check_tool_call(self, tool_name: str, args: dict, session: dict) -> bool:
        if tool_name in self.REQUIRE_HUMAN_APPROVAL:
            return False  # bloqueia → interrupt para aprovação humana
        if session["cost_usd"] > self.MAX_COST_PER_SESSION_USD:
            raise SessionCostExceeded()
        if session["steps"] > self.MAX_STEPS_PER_SESSION:
            raise SessionStepsExceeded()
        return True
```
