# Framework Selection Matrix

This is the definitive reference for choosing the right framework at Knovik. Use the decision tree first, then refer to the comparison table.

---

## Decision Tree

```{mermaid}
flowchart TD
    A[New agentic feature to build] --> B{Is the workflow stateful?\nDoes it loop, branch, or retry?}
    B -->|Yes| C{Privacy / on-device\nrequired?}
    B -->|No - linear pipeline| D{Data / retrieval heavy?}
    
    C -->|Yes - GDPR / EU| E[SmolAgents + Ollama]
    C -->|No| F[LangGraph]
    
    D -->|Yes - RAG, knowledge base| G[LlamaIndex]
    D -->|No| H{Multiple agents\nwith distinct roles?}
    
    H -->|Yes - like a team| I{Azure / Microsoft\ncompliance required?}
    H -->|No| J[Agno]
    
    I -->|Yes| K[AutoGen / Agent Framework]
    I -->|No| L[CrewAI]
    
    J --> M{Dev workflow automation\nor code generation?}
    M -->|Yes| N[MetaGPT]
    M -->|No| O[Agno — ship fast]
```

---

## Full Comparison Table

| | LangGraph | CrewAI | AutoGen | LlamaIndex | SmolAgents | Agno | MetaGPT |
|--|-----------|--------|---------|------------|------------|------|---------|
| **Best for** | Stateful workflows | Role-based teams | Conversational multi-agent | RAG / retrieval | Local/edge/GDPR | Fast prototyping | Dev automation |
| **Learning curve** | Medium | Low | Medium | Low-Medium | Low | Low | High |
| **Token efficiency** | ✅ High | ⚠️ Low | Medium | High | ✅ High (local) | ✅ High | Medium |
| **State management** | ✅ Native | Manual | Conversation history | Index-based | Minimal | Built-in | Role memory |
| **Human-in-the-loop** | ✅ Native | Manual | ✅ Native | No | No | No | No |
| **Local models** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Designed for it | ✅ Yes | ✅ Yes |
| **GDPR fit** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅✅ Best | ✅ Yes | ✅ Yes |
| **Production maturity** | ✅ High | ✅ High | ✅ High | ✅ High | Medium | Medium | Medium |
| **Knovik product** | NicheXpert | Marketing Platform | AIHR | Answee | EU Products | R&D | Internal tooling |

---

## Cost Benchmark

Running the same niche research task across frameworks (approximate, Claude Sonnet pricing):

| Framework | Avg tokens per run | Relative cost |
|-----------|-------------------|---------------|
| LangGraph | ~3,000 | 1× (baseline) |
| Agno | ~3,200 | 1.1× |
| LlamaIndex | ~3,500 | 1.2× |
| AutoGen | ~4,000 | 1.3× |
| CrewAI | ~9,000 | 3× |
| MetaGPT | ~7,000 | 2.3× |
| SmolAgents (local) | **$0** | **0×** |

```{admonition} Cost in context
:class: tip
CrewAI's 3× cost is justified when output quality drives revenue (Marketing Platform).
It is not justified for internal tooling or high-volume batch processing.
SmolAgents running locally on OVH VPS has zero marginal API cost — ideal for EU products with privacy requirements.
```

---

## Combining Frameworks

Frameworks are not mutually exclusive. The most powerful production patterns at Knovik combine them:

**Pattern 1: LangGraph orchestrates, CrewAI executes**  
LangGraph handles the state machine and routing. CrewAI agents are nodes within the graph, handling content generation tasks.

**Pattern 2: LlamaIndex feeds LangGraph**  
LlamaIndex handles all document retrieval and RAG. LangGraph handles the reasoning and action loop around it. Used in Answee.

**Pattern 3: SmolAgents + LlamaIndex for EU**  
All data stays local. SmolAgents runs on Ollama. LlamaIndex indexes documents locally (ChromaDB). Full GDPR compliance.
