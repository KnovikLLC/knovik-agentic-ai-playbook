# SmolAgents

**The framework for local, privacy-first, edge-deployed agents.**

---

## Why SmolAgents?

SmolAgents is a HuggingFace library designed around one core idea: **agents should be minimal and run anywhere** — including on your own hardware, without sending data to external APIs.

For Knovik's EU entity and GDPR-regulated products, this is not just a nice-to-have. It is an architectural requirement.

```{admonition} GDPR and agentic AI
:class: important
Under GDPR, processing personal data through third-party LLM APIs (OpenAI, Anthropic) requires appropriate data processing agreements and may still carry risk.
For EU products handling personal data, running inference locally with SmolAgents + Ollama eliminates this concern entirely.
This is Knovik's approved architecture for all EU-facing products.
```

---

## Core Concepts

| Concept | What it is |
|---------|-----------|
| **CodeAgent** | Agent that writes and executes Python code as its action space |
| **ToolCallingAgent** | Standard tool-calling agent (similar to LangGraph/CrewAI style) |
| **LiteLLMModel** | Connect to any model — Ollama, OpenAI, Anthropic, HuggingFace |
| **Tools** | Standard Python functions decorated with `@tool` |
| **ManagedAgent** | Wrap an agent to use it as a tool inside another agent |

---

## The Knovik Local Stack

```
SmolAgents
    ↓
LiteLLMModel (local)
    ↓
Ollama (runtime)
    ↓
Gemma-2B or Llama-3-8B (model)
    ↓
OVH VPS / AlmaLinux 9 (server)
```

No data leaves your infrastructure. Full GDPR compliance by design.

---

## Chapters in this section

```{tableofcontents}
```

---

## Knovik product fit

**Primary:** All EU-facing products (knovik.eu, GDPR-regulated features)  
**Secondary:** R&D experiments where we don't want to burn API credits  
**Benchmark task:** Same niche research task, but running entirely on Ollama + Gemma-2B locally
