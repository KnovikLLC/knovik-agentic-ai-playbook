# Agents and Roles

---

## Designing agents that actually work

The most common CrewAI mistake is writing generic agents. Generic agents produce generic output. The goal is to write agents that behave like the best possible version of the specialist you are trying to simulate.

This chapter covers how to design the three components of an agent — role, goal, backstory — and how tool assignment affects agent behavior.

---

## The Role field

The role is a **job title with context**. It tells the LLM what kind of specialist this agent is.

Weak:
```python
role="Researcher"
```

Strong:
```python
role="Senior Market Research Analyst specializing in consumer electronics and IoT devices"
```

The stronger role narrows the LLM's behavior toward domain-specific expertise. It also affects how the agent interprets ambiguous task instructions.

---

## The Goal field

The goal is a **statement of what the agent is optimizing for**. It shapes every decision the agent makes.

Weak:
```python
goal="Research things."
```

Strong:
```python
goal="""Find accurate, quantified market data for product niches.
Prioritize primary sources (industry reports, company filings) over secondary sources (blog posts).
Always include market size in USD, growth rate as a percentage, and at least 3 named competitors."""
```

The goal field is read before every task. Think of it as the agent's standing instructions.

---

## The Backstory field

The backstory is the most powerful and most underused field. It establishes:

- **Expertise level** — how the agent approaches problems
- **Communication style** — how it writes outputs
- **Values and priorities** — what it optimizes for when there are trade-offs

```{admonition} Backstory as system prompt extension
:class: note
Under the hood, CrewAI injects the role, goal, and backstory into the system prompt before every task.
A well-crafted backstory is effectively a detailed system prompt that persists across all tasks this agent handles.
```

**Example — SEO Specialist agent backstory:**
```python
backstory="""You are an SEO specialist with 8 years of experience in technical and on-page SEO.
You have helped SaaS products and e-commerce stores grow organic traffic by 300-500%.
You think in terms of search intent first — you understand that ranking means matching
what real users are actually searching for, not keyword stuffing.
You are familiar with Google's helpful content system and always optimize for E-E-A-T.
You give concrete, actionable recommendations, not generic advice."""
```

---

## Tool assignment

Agents only use tools you explicitly assign to them. This is by design — it prevents agents from taking actions outside their role.

```python
from crewai_tools import SerperDevTool, FileReadTool, FileWriteTool

search_tool = SerperDevTool()
file_read = FileReadTool()
file_write = FileWriteTool()

# Researcher can search and read files
researcher = Agent(
    role="Market Research Analyst",
    goal="...",
    backstory="...",
    tools=[search_tool, file_read],
    llm=llm
)

# Writer can read files (research output) and write files (drafts)
writer = Agent(
    role="Content Writer",
    goal="...",
    backstory="...",
    tools=[file_read, file_write],
    llm=llm
)

# Editor has no tools — only uses its LLM reasoning
editor = Agent(
    role="Senior Editor",
    goal="...",
    backstory="...",
    tools=[],
    llm=llm
)
```

---

## Agent configuration options

Beyond role/goal/backstory, these settings matter in production:

```python
agent = Agent(
    role="...",
    goal="...",
    backstory="...",
    llm=llm,
    tools=[...],

    # Control verbosity
    verbose=True,           # Prints thought process - useful in development

    # Limit loops to control cost
    max_iter=5,             # Maximum ReAct iterations per task

    # Allow the agent to delegate to others
    allow_delegation=False, # Default True in hierarchical process

    # Memory (covered in detail in chapter 3)
    memory=True,            # Enables short-term memory across tasks
)
```

```{admonition} max_iter in production
:class: warning
Always set `max_iter` explicitly. The default is 15 — an agent stuck on a hard task will happily use all 15 iterations, costing 3-5× what you budgeted.
For most Knovik tasks, 3-5 iterations is sufficient.
```

---

## A complete agent design example

Here is the Market Research Analyst agent used throughout this section, fully designed:

```python
from crewai import Agent
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(model="claude-3-5-sonnet-20241022", temperature=0.1)

market_researcher = Agent(
    role="Senior Market Research Analyst — Consumer Electronics & IoT",
    goal="""Deliver accurate, data-driven market research reports.
Every report must include: market size (USD), CAGR, top 3-5 competitors with market share,
key customer segments, and a clear opportunity statement.
Cite sources. Do not speculate — if data is unavailable, say so.""",
    backstory="""You are a senior analyst who has covered the smart home and consumer electronics
space for 12 years. You have produced market research reports for hardware startups,
e-commerce businesses, and venture capital firms.
You understand that bad data leads to bad product decisions — so you are rigorous about sources.
You write clearly and concisely. Executives and engineers both read your reports.""",
    tools=[search_tool],
    llm=llm,
    verbose=True,
    max_iter=4,
    allow_delegation=False,
    memory=True
)
```

---

## Next

In {doc}`03_tasks_and_crews` we cover how to define tasks that get the most out of these agents, and how the Crew ties everything together.

