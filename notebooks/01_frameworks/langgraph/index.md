# LangGraph

**The framework for stateful, graph-based agent workflows.**

---

## Why LangGraph?

LangGraph is the Knovik default for any agent workflow that needs:

- **Loops** — the agent needs to retry, reflect, or iterate
- **Conditional branching** — different paths based on intermediate results
- **State persistence** — the workflow must remember what happened in previous steps
- **Human-in-the-loop** — a human needs to approve or redirect mid-workflow
- **Parallel execution** — multiple branches running simultaneously

It was built by the LangChain team specifically because LangChain's linear chain model couldn't handle these patterns well.

```{admonition} LangGraph vs LangChain
:class: note
LangChain is great for linear pipelines: fetch → process → respond.
LangGraph is for everything that needs a loop, branch, or persistent state.
They are complementary, not competing — LangGraph is built on top of LangChain.
```

---

## Core Concepts

| Concept | What it is |
|---------|-----------|
| **StateGraph** | The graph definition — nodes and edges |
| **State** | A typed dictionary shared across all nodes |
| **Nodes** | Python functions that read and update state |
| **Edges** | Connections between nodes; can be conditional |
| **Checkpointer** | Persists state between steps (enables human-in-the-loop) |

---

## Chapters in this section

```{tableofcontents}
```

---

## When NOT to use LangGraph

- Simple single-step tasks (just use `llm.invoke()`)
- Pure RAG with no branching (use LlamaIndex)
- Role-based team orchestration (CrewAI is more natural)
- You need to ship something in 30 minutes (use Agno)

---

## Knovik product fit

**Primary:** NicheXpert research and validation pipeline  
**Secondary:** AIHR workflow routing with approval gates  
**Benchmark task used in this section:** Product niche research → validation → report generation
