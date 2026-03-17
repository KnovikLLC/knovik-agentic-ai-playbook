# Introduction to CrewAI

---

## The mental model

CrewAI is built around a single analogy: **a team of human specialists collaborating on a project**.

When you hire a content team at Knovik, you do not hire one person who does everything. You hire a researcher, a writer, an SEO specialist, and an editor. Each person has a defined role, a goal, and expertise. They hand work off to each other in sequence — or in parallel if the work allows it.

CrewAI models this directly in code.

```
Crew
 ├── Agent: Market Researcher  (role + goal + backstory)
 ├── Agent: Content Writer     (role + goal + backstory)
 ├── Agent: SEO Specialist     (role + goal + backstory)
 └── Agent: Editor             (role + goal + backstory)

Tasks
 ├── Task 1: Research the topic       → assigned to: Market Researcher
 ├── Task 2: Write the draft          → assigned to: Content Writer
 ├── Task 3: Optimize for SEO         → assigned to: SEO Specialist
 └── Task 4: Review and finalize      → assigned to: Editor
```

The Crew runs the tasks in order, passing the output of each task as context to the next.

---

## How CrewAI compares to LangGraph

This is the most important distinction to internalize:

| | LangGraph | CrewAI |
|--|-----------|--------|
| **Mental model** | State machine / graph | Team of specialists |
| **Primary strength** | Conditional routing, loops, state | Role-based delegation, output quality |
| **When to use** | Complex branching logic | Output-quality-driven pipelines |
| **Token cost** | Low | High (3× LangGraph) |
| **Learning curve** | Medium | Low |
| **Best Knovik fit** | NicheXpert validation | Marketing Platform content |

```{admonition} Rule of thumb
:class: tip
If you are asking "what should happen next?" — use LangGraph.
If you are asking "who should do this?" — use CrewAI.
```

---

## The four building blocks

Everything in CrewAI reduces to four things. Understand these and you understand the whole framework.

**1. Agent** — An LLM with a persona
```python
researcher = Agent(
    role="Market Research Analyst",
    goal="Find accurate, current market data for product niches",
    backstory="You are a senior analyst with 10 years of experience...",
    tools=[search_tool],
    llm=llm,
    verbose=True
)
```

**2. Task** — A unit of work with expected output
```python
research_task = Task(
    description="Research the {niche} market. Find market size, competitors, growth rate.",
    expected_output="A structured market research report with data points and sources.",
    agent=researcher
)
```

**3. Crew** — The team + process definition
```python
crew = Crew(
    agents=[researcher, writer, editor],
    tasks=[research_task, writing_task, editing_task],
    process=Process.sequential,
    verbose=True
)
```

**4. Process** — How tasks execute
- `Process.sequential` — tasks run one after another, each receiving the previous output
- `Process.hierarchical` — a manager agent delegates and reviews work

---

## What makes CrewAI produce high-quality outputs

The `backstory` field is not decoration. It is one of the most important levers you have.

A vague backstory:
```python
backstory="You are a writer."
```

A backstory that drives quality:
```python
backstory="""You are a senior content strategist at a technology company.
You have written hundreds of product comparison articles that rank on page 1 of Google.
You write in clear, direct prose — no fluff, no generic statements.
Your readers are technically literate consumers making purchasing decisions.
You always back claims with specific numbers and product names."""
```

The second backstory consistently produces better output because it gives the LLM a precise persona to inhabit, not a generic one.

---

## Next

In {doc}`02_agents_and_roles` we go deeper on agent design — how to write roles, goals, and backstories that produce the outputs you actually want.

