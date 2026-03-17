# Conditional Edges

---

## What conditional edges do

A conditional edge reads the current state after a node completes, and returns the name of the next node to run. This is how LangGraph implements branching logic — loops, validation gates, error recovery, and multi-path routing.

Without conditional edges, every workflow is a straight line. With them, you can build any branching pattern.

---

## The routing function pattern

A routing function takes state and returns a string — the name of the next node to run (or `END`).

```python
from langgraph.graph import END

def route_after_validate(state: NicheResearchState) -> str:
    """
    Routing function: called after validate_node completes.
    Returns the name of the next node to run.
    """
    if state["status"] == "rejected":
        return "reject"

    if state["data_score"] >= 60:
        return "report"

    if state["loop_count"] >= 3:
        # Looped 3 times and still not sufficient — give up
        return "reject"

    # Score too low — loop back to research
    return "research"
```

---

## Adding a conditional edge to the graph

Use `add_conditional_edges` to attach a routing function to a node:

```python
workflow.add_conditional_edges(
    "validate",              # The node this routing function runs after
    route_after_validate,    # The routing function
    {
        # Map return values → node names
        # (optional: makes the mapping explicit and validatable)
        "report": "report",
        "research": "research",
        "reject": "reject",
    }
)
```

The third argument (the dict) is optional but recommended. It makes LangGraph validate at compile time that every possible return value from your routing function maps to a known node.

---

## A complete graph with conditional routing

Building on the previous chapter's state and nodes:

```python
from typing import TypedDict, Annotated, Optional
from langchain_core.messages import BaseMessage, HumanMessage, SystemMessage
from langchain_anthropic import ChatAnthropic
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
import json

# ── State ────────────────────────────────────────────────────────────────────

class NicheResearchState(TypedDict):
    niche: str
    messages: Annotated[list[BaseMessage], add_messages]
    raw_research: str
    data_score: int
    validation_notes: str
    loop_count: int
    report: str
    status: str

# ── LLM ──────────────────────────────────────────────────────────────────────

llm = ChatAnthropic(model="claude-3-5-sonnet-20241022", temperature=0.1)

# ── Nodes ────────────────────────────────────────────────────────────────────

def research_node(state: NicheResearchState) -> dict:
    prompt = f"Research the '{state['niche']}' product niche. Provide market size, top 5 competitors, growth rate, and customer segments."
    response = llm.invoke([HumanMessage(content=prompt)])
    return {
        "messages": [HumanMessage(content=prompt), response],
        "raw_research": response.content,
        "loop_count": state.get("loop_count", 0) + 1,
    }

def validate_node(state: NicheResearchState) -> dict:
    prompt = f"""Score this market research on completeness (0-100).

Research:
{state['raw_research']}

Required fields: market size (USD), growth rate, 3+ competitors, customer segments.
Respond with JSON: {{"score": <int>, "notes": "<what's missing>"}}"""

    response = llm.invoke([HumanMessage(content=prompt)])

    try:
        # Parse the JSON response
        text = response.content
        start = text.find("{")
        end = text.rfind("}") + 1
        data = json.loads(text[start:end])
        score = int(data.get("score", 0))
        notes = data.get("notes", "")
    except Exception:
        score = 0
        notes = "Could not parse validation response"

    return {
        "data_score": score,
        "validation_notes": notes,
        "messages": [response],
    }

def generate_report_node(state: NicheResearchState) -> dict:
    prompt = f"""Format this research into a clean market report for: {state['niche']}

Research data:
{state['raw_research']}

Format as:
# Market Report: [Niche Name]
## Market Overview
## Market Size & Growth
## Key Competitors
## Customer Segments
## Key Risks
## Opportunity Assessment"""

    response = llm.invoke([HumanMessage(content=prompt)])
    return {
        "report": response.content,
        "status": "complete",
        "messages": [response],
    }

def reject_node(state: NicheResearchState) -> dict:
    reason = f"Data quality score {state.get('data_score', 0)}/100 after {state.get('loop_count', 0)} research attempts."
    return {
        "report": f"Research rejected. {reason}\nNotes: {state.get('validation_notes', '')}",
        "status": "rejected",
    }

# ── Routing function ──────────────────────────────────────────────────────────

def route_after_validate(state: NicheResearchState) -> str:
    if state.get("status") == "rejected":
        return "reject"
    if state.get("data_score", 0) >= 60:
        return "report"
    if state.get("loop_count", 0) >= 3:
        return "reject"
    return "research"

# ── Graph ─────────────────────────────────────────────────────────────────────

workflow = StateGraph(NicheResearchState)

workflow.add_node("research", research_node)
workflow.add_node("validate", validate_node)
workflow.add_node("report", generate_report_node)
workflow.add_node("reject", reject_node)

workflow.set_entry_point("research")

# Static edges
workflow.add_edge("research", "validate")
workflow.add_edge("report", END)
workflow.add_edge("reject", END)

# Conditional edge from validate
workflow.add_conditional_edges(
    "validate",
    route_after_validate,
    {"report": "report", "research": "research", "reject": "reject"}
)

graph = workflow.compile()
```

---

## Running the graph

```python
from dotenv import load_dotenv
load_dotenv()

result = graph.invoke({
    "niche": "Zigbee smart home hubs for Apple HomeKit users",
    "loop_count": 0,
    "status": "in_progress",
    "messages": [],
    "data_score": 0,
    "validation_notes": "",
    "raw_research": "",
    "report": "",
})

print(f"Status: {result['status']}")
print(f"Score: {result['data_score']}/100")
print(f"Loops: {result['loop_count']}")
print()
print(result["report"])
```

---

## Visualising the graph

LangGraph can generate a Mermaid diagram of any compiled graph:

```python
from IPython.display import display, Image

# In a Jupyter notebook
display(Image(graph.get_graph().draw_mermaid_png()))
```

For the graph above, you get:

```
graph LR
    __start__ --> research
    research --> validate
    validate -->|score >= 60| report
    validate -->|score < 60, loops < 3| research
    validate -->|loops >= 3| reject
    report --> __end__
    reject --> __end__
```

Use this to verify your graph structure matches your intended workflow before running it.

---

## Common routing patterns

**Retry with backoff**
```python
def route_with_retry(state: MyState) -> str:
    if state["error_count"] >= 3:
        return "error_handler"
    if state["result"] is not None:
        return "process"
    return "fetch"   # Loop back to retry
```

**Multi-path classification**
```python
def route_by_query_type(state: ChatState) -> str:
    query_type = state["query_classification"]
    if query_type == "product":
        return "product_agent"
    elif query_type == "support":
        return "support_agent"
    elif query_type == "billing":
        return "billing_agent"
    else:
        return "general_agent"
```

**Parallel fan-out** (using `Send`)
```python
from langgraph.types import Send

def fan_out_to_parallel(state: MyState) -> list[Send]:
    # Run the same node in parallel for each item in a list
    return [
        Send("process_item", {"item": item, "config": state["config"]})
        for item in state["items"]
    ]

workflow.add_conditional_edges("prepare", fan_out_to_parallel)
```

---

## Debugging routing issues

If your graph loops unexpectedly or routes to the wrong node:

```python
# Stream with node names to see the routing decisions
for step in graph.stream(initial_state):
    node_name = list(step.keys())[0]
    state_updates = step[node_name]
    print(f"→ {node_name}: score={state_updates.get('data_score', '?')}, loops={state_updates.get('loop_count', '?')}")
```

You can also inspect the full state at any point during streaming:

```python
for step in graph.stream(initial_state, stream_mode="values"):
    print(f"score={step.get('data_score', '?')}, loops={step.get('loop_count', '?')}, status={step.get('status', '?')}")
```

---

## Before you continue

- [ ] Write a routing function for a support ticket workflow: route to `billing_agent`, `technical_agent`, or `general_agent` based on a `ticket_type` field in state.
- [ ] What is the difference between `stream_mode="updates"` (default) and `stream_mode="values"`?
- [ ] Why is the third argument to `add_conditional_edges` useful even though it is optional?

## Next

In {doc}`04_human_in_the_loop` we add a checkpoint between `validate` and `report` so a human can review the data quality score and approve or redirect the workflow before the report is generated.
