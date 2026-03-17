# State and Nodes

---

## State: the backbone of a LangGraph workflow

State is a Python `TypedDict` — a dictionary with typed keys. It is shared across every node in the graph. Nodes receive the full state, do work, and return a partial update (only the keys that changed).

```python
from typing import TypedDict

class NicheResearchState(TypedDict):
    niche: str              # Input: the niche to research
    raw_research: str       # Set by: research_node
    data_score: int         # Set by: validate_node (0-100)
    report: str             # Set by: report_node
    loop_count: int         # Set by: multiple nodes (loop guard)
```

This state dict travels through the entire workflow. When `research_node` returns `{"raw_research": "..."}`, LangGraph merges that into the existing state — the other keys are unchanged.

---

## Designing state well

State should contain **everything a node needs to do its job**. A node should not call external services or read files to get data it needs — that data should already be in state, put there by an earlier node.

**Avoid this:**
```python
def generate_report_node(state: MyState) -> dict:
    # Bad: re-fetching data that should already be in state
    research = fetch_research_again(state["niche"])
    return {"report": format_report(research)}
```

**Do this:**
```python
def generate_report_node(state: MyState) -> dict:
    # Good: uses data that research_node already put in state
    return {"report": format_report(state["raw_research"])}
```

---

## Annotated reducers for lists and messages

By default, when a node returns an update to a key, it **replaces** the current value. For lists (like message histories), you usually want to **append** instead.

LangGraph handles this with `Annotated` type annotations:

```python
from typing import TypedDict, Annotated
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    # Annotated with `add_messages` — new messages are APPENDED, not replaced
    messages: Annotated[list[BaseMessage], add_messages]

    # Regular field — new value REPLACES old value
    summary: str
```

The `add_messages` reducer handles the merging logic. When a node returns `{"messages": [new_message]}`, it appends `new_message` to the existing list rather than replacing the whole list.

---

## A complete state design for NicheXpert

```python
from typing import TypedDict, Annotated, Optional
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages

class NicheResearchState(TypedDict):
    # Input
    niche: str

    # Research data
    messages: Annotated[list[BaseMessage], add_messages]
    raw_research: str
    market_size_usd: Optional[float]
    competitors: list[str]
    growth_rate: Optional[float]

    # Validation
    data_score: int           # 0-100 completeness score
    validation_notes: str     # What's missing, what's reliable

    # Loop control
    loop_count: int

    # Output
    report: str
    status: str               # "in_progress" | "complete" | "rejected"
```

---

## Nodes: Python functions that transform state

A node is a function that:

1. Takes the current state as its only argument
2. Does some work (calls an LLM, runs a tool, transforms data)
3. Returns a dict of only the state keys it wants to update

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, SystemMessage

llm = ChatAnthropic(model="claude-3-5-sonnet-20241022", temperature=0.1)

def research_node(state: NicheResearchState) -> dict:
    """
    Calls Claude to research the niche. Appends the response to messages
    and stores the text in raw_research.
    """
    prompt = f"""Research the following product niche thoroughly:

Niche: {state['niche']}

Provide:
1. Market size estimate (USD, current year)
2. Top 5 competitors with brief description
3. Annual growth rate (CAGR)
4. Primary customer segments
5. Key barriers to entry

Be specific. Cite approximate numbers. If you don't have reliable data for a field, say so explicitly."""

    response = llm.invoke([
        SystemMessage(content="You are a market research analyst."),
        HumanMessage(content=prompt)
    ])

    # Return ONLY the keys that changed
    return {
        "messages": [HumanMessage(content=prompt), response],
        "raw_research": response.content,
        "loop_count": state.get("loop_count", 0) + 1,
        "status": "in_progress"
    }
```

---

## Node best practices

**Return only what changed**

A node returns a dict with only the keys it updated. LangGraph merges this into the existing state. You do not need to return the full state.

```python
# Correct — only return what changed
def my_node(state: MyState) -> dict:
    return {"field_a": new_value}

# Wrong — do not return the full state
def my_node(state: MyState) -> dict:
    return {**state, "field_a": new_value}  # Don't do this
```

**Keep nodes focused**

Each node should do one thing. If a node is making two LLM calls, it probably should be two nodes. This makes the graph easier to debug and reason about.

**Log state changes for debugging**

```python
def research_node(state: NicheResearchState) -> dict:
    print(f"[research_node] loop_count={state.get('loop_count', 0)}, niche={state['niche']}")
    # ... rest of node
```

**Handle errors inside nodes**

```python
def research_node(state: NicheResearchState) -> dict:
    try:
        response = llm.invoke([...])
        return {"raw_research": response.content, "status": "in_progress"}
    except Exception as e:
        return {"status": "rejected", "validation_notes": f"Research failed: {str(e)}"}
```

---

## Adding nodes to the graph

```python
from langgraph.graph import StateGraph, END

workflow = StateGraph(NicheResearchState)

# Add nodes — first arg is the name (used in edges), second is the function
workflow.add_node("research", research_node)
workflow.add_node("validate", validate_node)
workflow.add_node("report", generate_report_node)
workflow.add_node("reject", reject_node)

# Set the entry point
workflow.set_entry_point("research")
```

Node names are strings. Choose them to reflect what the node does, not implementation details. `"research"` is better than `"call_claude_and_format_result"`.

---

## Static edges

A static edge connects two nodes unconditionally:

```python
# Always go from research to validate
workflow.add_edge("research", "validate")

# Always go from report to END
workflow.add_edge("report", END)
```

`END` is LangGraph's built-in terminal node. Import it from `langgraph.graph`.

---

## Compile and run

Once you have added all nodes and edges, compile and invoke:

```python
graph = workflow.compile()

initial_state = {
    "niche": "Zigbee smart home hubs for Apple HomeKit users",
    "loop_count": 0,
    "status": "in_progress",
    "messages": [],
    "competitors": [],
}

result = graph.invoke(initial_state)

print(result["report"])
print(f"Final status: {result['status']}")
print(f"Loops used: {result['loop_count']}")
```

---

## Streaming node outputs

For long-running workflows, stream intermediate results instead of waiting for the full result:

```python
for step in graph.stream(initial_state):
    node_name = list(step.keys())[0]
    node_output = step[node_name]
    print(f"[{node_name}] status={node_output.get('status', '?')}")
```

Each iteration of `graph.stream()` yields a dict with the node name as key and the state updates as value.

---

## Before you continue

- [ ] What is the difference between replacing and appending state updates? When would you use `add_messages`?
- [ ] Why should nodes return only the keys they changed, not the full state?
- [ ] What happens if a node raises an unhandled exception?

## Next

In {doc}`03_conditional_edges` we add the branching logic — routing from `validate` to either `research` (loop), `report`, or `reject` based on the data quality score.
