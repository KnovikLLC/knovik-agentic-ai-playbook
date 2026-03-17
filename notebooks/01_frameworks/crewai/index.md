# CrewAI

**The framework for role-based multi-agent teams.**

---

## Why CrewAI?

CrewAI models agents as a team of specialists, each with a defined role, goal, and backstory. Think of it like hiring a crew:

- A **Researcher** who finds information
- A **Writer** who turns findings into content  
- An **Editor** who reviews quality
- A **Publisher** who handles distribution

This mental model maps directly to how the Knovik Marketing Platform should work — and why CrewAI is the right choice for it.

```{admonition} CrewAI performance note
:class: warning
CrewAI's role/goal/backstory injection means it consumes more tokens than LangGraph or Agno for simple tasks.
This is the cost of its expressive role model. For the Marketing Platform where output quality matters more than token efficiency, this trade-off is worth it.
For high-volume pipelines, prefer LangGraph.
```

---

## Core Concepts

| Concept | What it is |
|---------|-----------|
| **Agent** | An LLM with a role, goal, and backstory |
| **Task** | A specific piece of work assigned to an agent |
| **Crew** | A collection of agents and tasks |
| **Process** | How tasks execute: `sequential` or `hierarchical` |
| **Tool** | A function an agent can call |

---

## Chapters in this section

```{tableofcontents}
```

---

## Knovik product fit

**Primary:** Marketing Platform — content research, writing, SEO review, social adaptation  
**Secondary:** NicheXpert content layer (after LangGraph does the research, CrewAI writes the report)  
**Benchmark task:** Research a niche → write a blog post → SEO review → adapt for social media
