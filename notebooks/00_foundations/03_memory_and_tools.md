# Memory and Tools

---

## Why memory and tools matter

An LLM has no memory beyond its context window, and no ability to act on the world. Memory and tools are the two mechanisms that fix both problems.

- **Memory** lets an agent maintain context across steps, sessions, and requests
- **Tools** let an agent read from and write to external systems

Together, they are what separates a conversational AI from an agent that can actually complete work.

---

## The four types of memory

### 1. In-context memory (conversation buffer)

The simplest form. Everything the agent has said and received stays in the current context window.

```python
from langchain_core.messages import HumanMessage, AIMessage

messages = [
    HumanMessage(content="What is the market size of the Zigbee hub market?"),
    AIMessage(content="The global Zigbee hub market was valued at approximately $2.1B in 2024..."),
    HumanMessage(content="Who are the top 3 competitors?"),
    # Agent now has context of the previous exchange
]
```

**Limit:** context window size. For Claude 3.5 Sonnet, this is 200K tokens — large enough for most tasks, but not for long-running sessions.

**Use when:** single-session tasks, tasks where the full history fits in context.

---

### 2. External memory (vector store)

Long-term storage retrieved by semantic similarity. The agent stores summaries or facts in a vector database, then retrieves relevant memories when needed.

```python
from langchain_community.vectorstores import Chroma
from langchain_anthropic import AnthropicEmbeddings

# Store a memory
vectorstore = Chroma(embedding_function=AnthropicEmbeddings())
vectorstore.add_texts(
    texts=["NicheXpert research run on 2025-01-15: Zigbee hub market, found $2.1B TAM"],
    metadatas=[{"date": "2025-01-15", "product": "NicheXpert", "niche": "zigbee-hubs"}]
)

# Retrieve relevant memories
results = vectorstore.similarity_search(
    "previous research on smart home hubs",
    k=3
)
```

**Use when:** multi-session applications, knowledge bases, RAG pipelines (Answee, NicheXpert).

---

### 3. Entity memory

Structured storage for facts about specific entities — products, users, companies. Keeps a persistent record that updates as the agent learns more.

```python
# Pseudo-code: entity memory pattern
entity_store = {
    "Zigbee Hub Market": {
        "market_size": "$2.1B (2024)",
        "growth_rate": "12.4% CAGR",
        "top_competitors": ["Philips Hue", "IKEA Tradfri", "Aeotec"],
        "last_updated": "2025-01-15"
    }
}

# Agent updates this as it learns more facts
entity_store["Zigbee Hub Market"]["barriers_to_entry"] = "Z-Wave certification cost ~$5,000"
```

CrewAI has built-in entity memory when you set `memory=True` on a Crew. LangGraph requires you to manage this in state.

---

### 4. Episodic memory (run history)

Records of past agent runs — what was tried, what succeeded, what failed. Used to avoid repeating mistakes.

```python
# LangGraph with SQLite checkpointer — built-in episodic memory
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = SqliteSaver.from_conn_string("agent_memory.db")

# Each run with a thread_id is stored and can be resumed
graph = workflow.compile(checkpointer=checkpointer)
result = graph.invoke(
    {"messages": [HumanMessage(content="Research Zigbee hubs")]},
    config={"configurable": {"thread_id": "run-001"}}
)

# Resume or inspect a previous run
state = graph.get_state(config={"configurable": {"thread_id": "run-001"}})
```

---

## Choosing the right memory type

```{list-table}
:header-rows: 1
:widths: 20 25 25 30

* - Type
  - Storage
  - Retrieval
  - Best for
* - **In-context**
  - LLM context window
  - Always available
  - Single-session tasks
* - **Vector store**
  - Chroma / Pinecone / pgvector
  - Semantic similarity
  - RAG, long-term knowledge
* - **Entity**
  - Dict / database
  - Key lookup
  - Tracking known entities
* - **Episodic**
  - SQLite / Postgres
  - Thread ID
  - Multi-step workflows, HITL
```

---

## Tools: giving agents the ability to act

A tool is a Python function that an agent can call. The LLM decides when and how to call it, based on the function name and docstring.

### The anatomy of a tool

```python
from langchain_core.tools import tool

@tool
def search_web(query: str) -> str:
    """
    Search the web for current information about a topic.
    Use this when you need recent data not in your training.

    Args:
        query: The search query to run

    Returns:
        A string containing the top search results
    """
    # Implementation: call Serper, Tavily, or similar API
    return results_as_string
```

The docstring is **not optional**. The LLM reads it to decide whether to call this tool. Write it as if explaining to a junior analyst when and why to use it.

---

### Tool categories in Knovik's stack

**Web and data retrieval**
```python
from langchain_community.tools import TavilySearchResults
from crewai_tools import SerperDevTool

tavily_search = TavilySearchResults(max_results=5)  # Good for research tasks
serper_search = SerperDevTool()                       # Good for SEO data
```

**File I/O**
```python
from langchain_community.tools import ReadFileTool, WriteFileTool

read_file = ReadFileTool()
write_file = WriteFileTool()
```

**Custom business tools**
```python
@tool
def get_product_catalog(category: str) -> str:
    """
    Retrieves product listings from the ATHBridge WooCommerce store.
    Use when you need current product names, prices, and ratings.

    Args:
        category: Product category slug (e.g., 'zigbee-hubs')
    """
    # Call WooCommerce API
    ...
```

---

### Tool safety rules

```{admonition} Never let tools raise exceptions
:class: warning
If a tool raises an unhandled exception, the agent crashes or loops indefinitely.
Always wrap tool implementations in try/except and return an error string.
```

```python
@tool
def get_competitor_data(product_name: str) -> str:
    """Fetches competitor pricing from our internal database."""
    try:
        result = db.query("SELECT * FROM products WHERE name LIKE %s", product_name)
        return json.dumps(result, indent=2)
    except Exception as e:
        return f"Database unavailable: {str(e)}. Use web search as fallback."
```

**Rules for all Knovik tools:**

1. **Return strings only** — serialize dicts and lists with `json.dumps()`
2. **Handle all exceptions** — return error strings, never raise
3. **One job per tool** — narrow scope makes tools easier to debug and for agents to use correctly
4. **Include examples in docstrings** — show the agent what good inputs look like
5. **Set timeouts** — always pass `timeout=10` or similar to external HTTP calls

---

## How agents decide to use tools: function calling

When you give an agent tools, the framework sends the tool definitions alongside your prompt. The LLM chooses to call a tool by returning a structured JSON object instead of text.

```
User: What is the current market size of the Zigbee hub market?

Agent thinks: I need current data. I have `search_web` available.
Agent calls: search_web(query="Zigbee hub market size 2024 2025")
Tool returns: "According to MarketsandMarkets, the global Zigbee hub..."
Agent responds: "Based on current data, the Zigbee hub market..."
```

This loop — **Reason → Act → Observe → Reason** — is the ReAct pattern you saw in chapter 2.

---

## Memory in the frameworks you will use

| Framework | In-context | Vector store | Entity | Episodic |
|-----------|-----------|-------------|--------|---------|
| **LangGraph** | State dict | Manual integration | Manual in state | SQLite/Postgres checkpointer |
| **CrewAI** | Automatic | `memory=True` + embedder | Built-in entity memory | Manual |
| **LlamaIndex** | Chat buffer | Built-in (core capability) | Index metadata | Manual |
| **AutoGen** | Automatic | Via LlamaIndex/Chroma | Manual | ConversableAgent history |

---

## Before you continue

- [ ] What are the four types of memory? When would you use each?
- [ ] Why must tools never raise exceptions?
- [ ] What does the LLM read to decide whether to call a tool?
- [ ] What is the ReAct loop and how do tools fit into it?
