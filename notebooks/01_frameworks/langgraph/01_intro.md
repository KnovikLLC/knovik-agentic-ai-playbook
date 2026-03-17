# Introduction to LangGraph

---

## The core idea

LangGraph models agent workflows as a **directed graph**. Nodes are Python functions. Edges define what runs next — and those edges can be conditional.

This is a fundamentally different mental model from chains (linear) or crews (role-based delegation). With LangGraph, you draw your workflow as a diagram first, then implement each node as a function.

```
START → research_node → validate_node → [conditional] → report_node → END
                                              ↓
                                         reject_node → END
```

The conditional edge reads the current state and decides which node to route to next. This is how LangGraph handles branching logic that would be awkward or impossible with LangChain.

---

## Your first LangGraph program

Before theory, here is a minimal but complete LangGraph workflow:

```python
from typing import TypedDict
from langgraph.graph import StateGraph, END
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage

# 1. Define the state
class AgentState(TypedDict):
    question: str
    answer: str

# 2. Define nodes (functions that transform state)
llm = ChatAnthropic(model="claude-3-5-haiku-20241022")

def answer_node(state: AgentState) -> AgentState:
    response = llm.invoke([HumanMessage(content=state["question"])])
    return {"answer": response.content}

# 3. Build the graph
workflow = StateGraph(AgentState)
workflow.add_node("answer", answer_node)
workflow.set_entry_point("answer")
workflow.add_edge("answer", END)

# 4. Compile and run
graph = workflow.compile()
result = graph.invoke({"question": "What is the capital of France?"})
print(result["answer"])  # Paris
```

Six operations: define state, define node, create graph, add node, add entry point, add edge. That is the complete API for a simple workflow.

---

## Why use a graph instead of a chain?

A LangChain chain is a pipeline: A → B → C → done. Good for simple, linear tasks.

LangGraph adds three things that chains cannot do cleanly:

**1. Loops** — a node can route back to an earlier node

```
research → validate → [if insufficient] → research (loop back)
                    → [if sufficient] → report
```

**2. Conditional routing** — different paths based on state content

```
classify_query → [if product question] → product_agent
              → [if support question] → support_agent
              → [if other] → general_agent
```

**3. Persistent state** — the state dict survives across steps and can be checkpointed to disk, enabling pause/resume and human-in-the-loop patterns.

---

## How LangGraph compares to raw LangChain

```{list-table}
:header-rows: 1
:widths: 30 35 35

* - Feature
  - LangChain (LCEL)
  - LangGraph
* - Execution model
  - Linear pipeline
  - Directed graph
* - Loops
  - Not supported natively
  - First-class
* - Conditional routing
  - Limited
  - Core feature
* - State persistence
  - Not built-in
  - Built-in checkpointer
* - Human-in-the-loop
  - Manual
  - Built-in interrupt
* - Complexity
  - Simple workflows
  - Complex multi-step
```

LangGraph is built on top of LangChain — you can use LangChain tools, prompts, and LLMs inside LangGraph nodes. They are complementary.

---

## The five primitives

Everything in LangGraph reduces to five concepts. Master these and you can build any workflow.

**1. State** — A TypedDict that is shared across all nodes. Nodes read from state and return updates to state.

**2. Nodes** — Python functions that take state as input and return a dict of state updates.

**3. Edges** — Define what runs after a node. Can be static (always go to node X) or conditional (go to X or Y based on state).

**4. StateGraph** — The graph object you add nodes and edges to.

**5. Checkpointer** — Optional persistence layer. Saves state between steps. Required for human-in-the-loop.

---

## The state machine mental model

LangGraph workflows behave exactly like finite state machines:

```
Current state + node function → New state
New state + edge function → Next node
```

Every step:

1. The current node function runs, reading state and returning updates
2. The state is updated (merged, not replaced)
3. The edge from that node determines the next node
4. If the next node is `END`, the graph stops

```python
# Node: reads state, returns partial updates
def my_node(state: MyState) -> dict:
    # Read from state
    current_value = state["some_field"]

    # Do work
    result = do_something(current_value)

    # Return ONLY what changed (partial update)
    return {"result_field": result}
    # Note: you don't return the full state,
    # only the fields that changed
```

---

## When LangGraph is the right choice for Knovik

Use LangGraph when your workflow has any of these:

- **Validation loops** — research a niche, validate the data, loop back if insufficient
- **Approval gates** — a human must approve before the workflow continues
- **Parallel branches** — run competitor research and customer research simultaneously
- **Error recovery** — retry a failed step with different parameters
- **Multi-step pipelines with state** — 4+ steps where each step needs data from multiple previous steps

**Primary Knovik use case:** NicheXpert research pipeline — research → validate → score → generate report, with conditional loops if the data quality is insufficient.

---

## What you will build in this section

The running example across all LangGraph chapters is a **niche research and validation agent**:

```
START
  │
  ▼
research_niche(state)     ← calls web search, gathers market data
  │
  ▼
validate_data(state)      ← scores data completeness (0-100)
  │
  ▼
[conditional edge]
  ├── score < 60  → research_niche  (loop: try again)
  ├── score ≥ 60  → generate_report
  └── max_loops   → reject (data insufficient)
  │
  ▼
generate_report(state)    ← formats final markdown report
  │
  ▼
END
```

By chapter 4, you will add a human-in-the-loop step between `validate_data` and `generate_report`, so a human can review the data quality score before committing to report generation.

---

## Next

In {doc}`02_state_and_nodes` we cover the state definition in depth — TypedDict, Annotated reducers, and how to design state that scales as your workflow grows.
