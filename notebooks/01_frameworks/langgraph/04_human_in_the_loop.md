# Human-in-the-Loop

---

## Why human-in-the-loop matters

Fully autonomous agents make mistakes. For production workflows where errors are expensive — publishing wrong market data, triggering irreversible API calls, generating content that goes directly to customers — you need a mechanism for a human to review and approve before the workflow continues.

LangGraph builds this in via **checkpointers** and **interrupts**. No custom state management required.

---

## The two mechanisms

**Checkpointer** — persists the entire state graph to storage (memory, SQLite, or Postgres) after each node completes. This makes it possible to:
- Pause the graph mid-run
- Resume from exactly where it stopped
- Inspect the state at any point
- Roll back to a previous state

**Interrupt** — tells LangGraph to pause the graph before (or after) a specific node and wait for external input before continuing.

---

## Step 1: Add a checkpointer

```python
from langgraph.checkpoint.memory import MemorySaver  # In-memory (dev/testing)
# from langgraph.checkpoint.sqlite import SqliteSaver  # SQLite (production)

checkpointer = MemorySaver()

graph = workflow.compile(checkpointer=checkpointer)
```

Without a checkpointer, interrupts will not work. The checkpointer is what saves the state between the interrupt and the resume.

For production, use SQLite (single-process) or Postgres (multi-process):

```python
# SQLite — persists to disk, survives process restarts
from langgraph.checkpoint.sqlite import SqliteSaver
checkpointer = SqliteSaver.from_conn_string("./checkpoints.db")

# Postgres — for multi-process / multi-server setups
# from langgraph.checkpoint.postgres import PostgresSaver
# checkpointer = PostgresSaver.from_conn_string(os.getenv("DATABASE_URL"))
```

---

## Step 2: Add an interrupt

Compile the graph with `interrupt_before` to pause before a specific node:

```python
graph = workflow.compile(
    checkpointer=checkpointer,
    interrupt_before=["report"]  # Pause before the report node runs
)
```

Now when the graph reaches the `validate` → `report` edge, it will stop and return control to you before running `report`.

---

## Step 3: Run with a thread_id

Every run that uses a checkpointer needs a `thread_id`. This is how LangGraph tracks which saved state belongs to which run.

```python
from langchain_core.messages import HumanMessage

thread_config = {"configurable": {"thread_id": "research-run-001"}}

# Run the graph — it will stop before "report"
result = graph.invoke(
    {
        "niche": "Zigbee smart home hubs for Apple HomeKit users",
        "loop_count": 0,
        "status": "in_progress",
        "messages": [],
        "data_score": 0,
        "validation_notes": "",
        "raw_research": "",
        "report": "",
    },
    config=thread_config
)

# result is the state at the point of interruption
print(f"Paused. Data score: {result['data_score']}/100")
print(f"Validation notes: {result['validation_notes']}")
print()
print("Review the research data above. Approve to generate report, or reject.")
```

---

## Step 4: Inspect the paused state

```python
# Get the full current state
current_state = graph.get_state(config=thread_config)

print("Current node:", current_state.next)          # ('report',) — next to run
print("Data score:", current_state.values["data_score"])
print("Research:", current_state.values["raw_research"][:500])
```

`current_state.next` tells you which node will run when you resume.

---

## Step 5: Resume or redirect

**Resume — continue as planned:**
```python
# Pass None to resume with the existing state unchanged
graph.invoke(None, config=thread_config)
```

**Update state before resuming:**
```python
# Human reviewer adds a note to the state
graph.update_state(
    config=thread_config,
    values={
        "validation_notes": "Reviewed and approved. Market size data confirmed via Statista.",
    }
)

# Then resume
graph.invoke(None, config=thread_config)
```

**Override routing — redirect to a different node:**
```python
# Set state to route to reject instead of report
graph.update_state(
    config=thread_config,
    values={"status": "rejected"},
    as_node="validate"  # Pretend the update came from the validate node
)

# Resume — routing function now sees status="rejected" and goes to reject
graph.invoke(None, config=thread_config)
```

---

## Complete HITL example: NicheXpert approval gate

```python
import os
from dotenv import load_dotenv
from typing import TypedDict, Annotated
from langchain_core.messages import BaseMessage, HumanMessage
from langchain_anthropic import ChatAnthropic
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
import json

load_dotenv()

# ── State (same as previous chapters) ────────────────────────────────────────

class NicheResearchState(TypedDict):
    niche: str
    messages: Annotated[list[BaseMessage], add_messages]
    raw_research: str
    data_score: int
    validation_notes: str
    loop_count: int
    report: str
    status: str

# ── Nodes (same as previous chapter) ─────────────────────────────────────────
# ... (research_node, validate_node, generate_report_node, reject_node)

# ── Routing function (same as previous chapter) ───────────────────────────────
# ... (route_after_validate)

# ── Graph with checkpointer and interrupt ─────────────────────────────────────

workflow = StateGraph(NicheResearchState)
workflow.add_node("research", research_node)
workflow.add_node("validate", validate_node)
workflow.add_node("report", generate_report_node)
workflow.add_node("reject", reject_node)
workflow.set_entry_point("research")
workflow.add_edge("research", "validate")
workflow.add_edge("report", END)
workflow.add_edge("reject", END)
workflow.add_conditional_edges(
    "validate",
    route_after_validate,
    {"report": "report", "research": "research", "reject": "reject"}
)

# Add checkpointer and interrupt_before
checkpointer = MemorySaver()
graph = workflow.compile(
    checkpointer=checkpointer,
    interrupt_before=["report"]  # Pause before generating the report
)

# ── Run 1: Execute up to the interrupt ───────────────────────────────────────

thread = {"configurable": {"thread_id": "niche-001"}}

state = graph.invoke(
    {
        "niche": "Zigbee smart home hubs for Apple HomeKit",
        "loop_count": 0,
        "status": "in_progress",
        "messages": [],
        "data_score": 0,
        "validation_notes": "",
        "raw_research": "",
        "report": "",
    },
    config=thread
)

# ── Human review ─────────────────────────────────────────────────────────────

print(f"\n{'='*60}")
print("APPROVAL REQUIRED")
print(f"{'='*60}")
print(f"Niche: {state['niche']}")
print(f"Data quality score: {state['data_score']}/100")
print(f"Validation notes: {state['validation_notes']}")
print(f"Research loops used: {state['loop_count']}")
print(f"\nNext step: {graph.get_state(config=thread).next}")
print(f"{'='*60}\n")

approval = input("Approve report generation? (y/n): ").strip().lower()

# ── Run 2: Resume or reject based on human input ─────────────────────────────

if approval == "y":
    final = graph.invoke(None, config=thread)
    print(f"\n✓ Report generated ({len(final['report'])} chars)")
    print(final["report"])
else:
    graph.update_state(
        config=thread,
        values={"status": "rejected", "validation_notes": "Rejected by human reviewer"},
        as_node="validate"
    )
    final = graph.invoke(None, config=thread)
    print(f"\n✗ Research rejected: {final['report']}")
```

---

## Thread IDs in production

In a real application, `thread_id` maps to a user session or job ID:

```python
import uuid

# For a new research job
thread_id = f"research-{uuid.uuid4()}"

# For an ongoing conversation
thread_id = f"user-{user_id}-session-{session_id}"

# For a scheduled job
thread_id = f"niche-{niche_slug}-{date_today}"
```

Store thread IDs in your database alongside the job status. When a human approves via a web UI, look up the thread_id and call `graph.invoke(None, config={"configurable": {"thread_id": thread_id}})`.

---

## Resuming after a server restart

With `MemorySaver`, state is lost when the process exits. For production, use `SqliteSaver` or `PostgresSaver`:

```python
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = SqliteSaver.from_conn_string("./checkpoints.db")
graph = workflow.compile(checkpointer=checkpointer, interrupt_before=["report"])

# After a restart, resume with the same thread_id — state is loaded from disk
graph.invoke(None, config={"configurable": {"thread_id": "niche-001"}})
```

The graph resumes from exactly where it paused, with all state intact.

---

## Time-travel: rolling back to a previous checkpoint

LangGraph stores every state as it passes through each node. You can roll back to any point:

```python
# List all saved states for a thread
history = list(graph.get_state_history(config=thread))
for checkpoint in history:
    print(f"Step: {checkpoint.next}, score={checkpoint.values.get('data_score', '?')}")

# Roll back to a specific checkpoint
target_checkpoint = history[2]  # 3rd checkpoint
graph.update_state(
    config={"configurable": {
        "thread_id": "niche-001",
        "checkpoint_id": target_checkpoint.config["configurable"]["checkpoint_id"]
    }},
    values={}
)
```

---

## Before you continue

- [ ] What is the difference between `MemorySaver` and `SqliteSaver`? When would you use each?
- [ ] How do you update state before resuming a paused graph?
- [ ] If a graph is interrupted before `"report"` and you want to redirect it to `"reject"` instead, what two calls do you make?

## Next

In {doc}`05_demo_niche_researcher` we wire everything from chapters 1–4 into a complete, runnable notebook: a NicheXpert-style research agent with loops, conditional routing, and a human approval gate.
