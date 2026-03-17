# Knovik AI Engineering Playbook

**Built by Knovik engineers, for Knovik engineers — and anyone else who wants to build serious agentic AI systems.**

---

<h1 align="center">Knovik AI Engineering Playbook</h1>
<p>Designed and Developed by Madusanka Premaratne for Knovik Engineering Department</p>

---

## What is this?

This is a living, collaborative engineering playbook covering the full landscape of agentic AI frameworks in 2026. It is not a theoretical survey - every chapter has runnable code you can execute directly in Google Colab.

The consistent demo problem across all framework chapters: **"Research a product niche, validate it, and produce a competitor summary report"** — a real task from NicheXpert. This lets you compare frameworks on identical ground.

## Who is this for?

- Knovik engineers building agentic features into Answee, AIHR, NicheXpert, and the Marketing Platform
- Anyone on the team wanting to understand the "why" behind framework choices
- External contributors and the broader community — PRs are welcome

## How to use this book

**Read linearly** if you are new to agentic AI — Part 0 builds the foundation every other chapter depends on.

**Jump to a framework** if you already know the basics — each Part 1 chapter is self-contained.

**Go straight to recipes** (Part 3) if you are building a specific Knovik product right now.

---

## Frameworks Covered

| Framework | Best For | Knovik Product |
|-----------|----------|----------------|
| **LangGraph** | Stateful workflows, conditional branching | NicheXpert |
| **CrewAI** | Role-based multi-agent teams | Marketing Platform |
| **AutoGen / Agent Framework** | Conversational multi-agent, compliance | AIHR |
| **LlamaIndex** | RAG, data retrieval, knowledge bases | Answee |
| **SmolAgents** | Local/edge inference, GDPR-first | EU Products |
| **Agno** | Fast prototyping, lightweight agents | R&D Experiments |
| **MetaGPT** | Dev workflow automation, code generation | Internal Tooling |

---

## Quick Start

Every chapter has a **"Open in Colab"** badge. Click it, run all cells, and you have a working agent in minutes.

For local setup, see {doc}`notebooks/00_foundations/04_setup_environment`.

---

## Contributing

This book lives at [https://github.com/knivok/ai-playbook](https://knovikllc.github.io/knovik-agentic-ai-playbook/).

- **Add a recipe**: Create a notebook in `notebooks/03_knovik_recipes/` and open a PR
- **Fix an error**: Use the "suggest edit" button on any page
- **Add a framework chapter section**: Follow the structure in any existing framework `index.md`

**Owner:** Deelaka Kariyawasam (R&D Lead) / Iroshana Wickremasinghe (VP Engineering)
**Last updated:** March 2026
