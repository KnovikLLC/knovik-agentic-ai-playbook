# What is Agentic AI?

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/knivok/ai-playbook/blob/main/notebooks/00_foundations/01_what_is_agentic_ai.ipynb)

---

## The difference that matters

A standard LLM call is a **one-shot transformation**: you give it input, it gives you output. Done.

An **agentic system** is different. It:

1. Receives a high-level goal
2. Breaks the goal into sub-tasks
3. Uses tools to act on the world (search, write files, call APIs)
4. Observes the results
5. Replans and continues until the goal is complete

```{admonition} Simple analogy
:class: tip
A standard LLM is like asking someone a question. An agent is like hiring someone to complete a project - they figure out the steps, use whatever tools they need, and report back when done.
```

---

## The spectrum of autonomy

Not all agents are equal. There is a spectrum:

```
Prompt → Chain → ReAct Agent → Multi-Agent → Fully Autonomous
  ↑                                                      ↑
Simple                                              Complex
Low risk                                           High risk
Fast                                               Slow
Cheap                                           Expensive
```

| Level | What it is | Example |
|-------|-----------|---------|
| **Prompt** | Single LLM call | Summarize this text |
| **Chain** | Sequential LLM calls | Translate then summarize |
| **ReAct Agent** | LLM + tools, single agent | Search Google, then answer |
| **Multi-Agent** | Multiple agents collaborating | Researcher + Writer + Editor crew |
| **Autonomous** | Long-running, self-directed | AutoGPT-style goal completion |

```{admonition} Knovik default position
:class: note
Most Knovik product features sit at ReAct Agent or Multi-Agent level. Full autonomy is rarely the right choice in production - the cost and unpredictability are too high. Design for the minimum autonomy that solves the problem.
```

---

## Why now?

Three things converged in 2024-2025 that made production-grade agentic systems practical:

**1. Models got reliable enough.** GPT-4o, Claude 3.5+, Gemini 1.5 have sufficiently low hallucination rates that agents can complete multi-step tasks without going off the rails on every run.

**2. Tool use became standardized.** Function calling / tool use APIs across all major models made it practical to give agents reliable access to external systems.

**3. Frameworks matured.** LangGraph, CrewAI, AutoGen and others solved the hard infrastructure problems (state management, error recovery, observability) so we don't build from scratch.

---

## What makes an agent an agent?

Every agent has four core components:

```{list-table}
:header-rows: 1
:widths: 20 40 40

* - Component
  - What it does
  - Example
* - **LLM Brain**
  - Reasons, plans, decides what to do next
  - Claude 3.5 Sonnet, GPT-4o
* - **Tools**
  - Lets the agent act on the world
  - web_search, read_file, call_api
* - **Memory**
  - Maintains context and history
  - Conversation buffer, vector DB
* - **Orchestrator**
  - Manages the agent loop, routes outputs
  - LangGraph, CrewAI, AutoGen
```

In the next chapter, we dig into exactly how these four pieces work together in the agent loop.

---

## Before you continue

Make sure you can answer these:

- [ ] What is the difference between a chain and an agent?
- [ ] Name the four core components of an agent
- [ ] When would you NOT use an agent? (hint: when a single LLM call is enough)
