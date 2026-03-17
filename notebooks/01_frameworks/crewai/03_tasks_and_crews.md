# Tasks and Crews

---

## Tasks: the unit of work

A Task defines **what needs to be done**, **what good output looks like**, and **who does it**.

The most important field is `expected_output`. This is not a description of what the agent will do — it is a description of what you expect to receive. Think of it as the acceptance criteria for the task.

```python
from crewai import Task

research_task = Task(
    description="""Research the {niche} market thoroughly.
    
    Focus on:
    - Total addressable market size (USD, current year)
    - Key competitors and their estimated market share
    - Annual growth rate (CAGR)
    - Primary customer segments
    - Biggest barriers to entry
    - Any recent market disruptions (last 12 months)
    
    The niche to research: {niche}
    """,

    expected_output="""A structured market research report containing:
    1. Market Overview (2-3 sentences)
    2. Market Size: $X billion (source cited)
    3. Growth Rate: X% CAGR
    4. Top 5 Competitors: [name, estimated share, key differentiator]
    5. Customer Segments: [segment, size, key needs]
    6. Barriers to Entry: [barrier, severity]
    7. Recent Disruptions: [event, impact]
    8. Opportunity Summary: 1 paragraph
    """,

    agent=market_researcher,
    output_file="research_output.md"  # Optional: saves to file
)
```

---

## Context: passing data between tasks

By default in sequential process, each task receives the **full output of the previous task** as context. You can also specify explicit context dependencies:

```python
writing_task = Task(
    description="Write a 1,500-word blog post about {niche} based on the research provided.",
    expected_output="A complete blog post with title, intro, 3 main sections, and conclusion.",
    agent=content_writer,
    context=[research_task]  # Explicitly depends on research_task output
)
```

This is particularly useful in hierarchical processes where tasks do not run strictly in sequence.

---

## Template variables

Notice `{niche}` in the task descriptions above. CrewAI supports template variables — you pass them when you call `crew.kickoff()`:

```python
result = crew.kickoff(inputs={"niche": "Zigbee smart home hubs for Apple HomeKit users"})
```

This makes your crews reusable across different inputs without changing any code. For the Marketing Platform, this means one crew definition handles any product category.

---

## Process types

### Sequential (default)

Tasks run one by one. Output of Task N becomes context for Task N+1.

```
Task 1 → Task 2 → Task 3 → Final Output
```

```python
from crewai import Crew, Process

crew = Crew(
    agents=[researcher, writer, seo_specialist, editor],
    tasks=[research_task, writing_task, seo_task, editing_task],
    process=Process.sequential,
    verbose=True
)
```

### Hierarchical

A manager agent receives the goal, delegates tasks to specialist agents, reviews their outputs, and produces the final result. The manager uses its own LLM reasoning to decide delegation — you do not hardcode the order.

```python
from crewai import Crew, Process, Agent

manager = Agent(
    role="Content Director",
    goal="Produce the highest quality content for Knovik's marketing platform.",
    backstory="You are an experienced content director who delegates strategically...",
    llm=llm,
    allow_delegation=True
)

crew = Crew(
    agents=[researcher, writer, seo_specialist, editor],
    tasks=[research_task, writing_task, seo_task, editing_task],
    process=Process.hierarchical,
    manager_agent=manager,
    verbose=True
)
```

```{admonition} When to use hierarchical
:class: note
Hierarchical process is more powerful but significantly more expensive — the manager agent adds extra LLM calls.
Use it when task order is genuinely dynamic (the manager decides what to do next based on intermediate results).
For fixed pipelines like Knovik's Marketing Platform, sequential is more predictable and cheaper.
```

---

## Crew configuration options

```python
crew = Crew(
    agents=[...],
    tasks=[...],
    process=Process.sequential,

    # Logging
    verbose=True,           # Prints full agent reasoning

    # Memory options (see memory chapter)
    memory=True,            # Cross-task memory
    embedder={              # Embedding model for memory
        "provider": "openai",
        "config": {"model": "text-embedding-3-small"}
    },

    # Output
    output_log_file="crew_run.log",

    # Language (useful for non-English content)
    language="en",

    # Planning mode — adds a planning step before execution
    planning=True,          # Manager creates a plan first
)
```

---

## Accessing outputs

```python
result = crew.kickoff(inputs={"niche": "Zigbee hubs"})

# The final task output as a string
print(result.raw)

# All task outputs (useful for pipelines that need intermediate results)
for task_output in result.tasks_output:
    print(f"Task: {task_output.name}")
    print(f"Output: {task_output.raw[:200]}")

# Token usage breakdown
print(result.token_usage)
```

---

## Next

In {doc}`04_tools_integration` we cover how to give your agents real capabilities — web search, file I/O, APIs, and custom tools specific to Knovik's stack.

