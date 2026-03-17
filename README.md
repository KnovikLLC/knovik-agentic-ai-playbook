# Knovik AI Engineering Playbook

> A collaborative, runnable guide to agentic AI frameworks for Knovik engineers and the broader community.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

** 📖 Read online:** [Knivok Agentic AI Playbook](https://knovikllc.github.io/knovik-agentic-ai-playbook/notebooks/)

---

## What's in here

A practical, code-first playbook covering 7 agentic AI frameworks:

| Framework | Chapter | Knovik Product |
|-----------|---------|----------------|
| LangGraph | Part 1 | NicheXpert |
| CrewAI | Part 1 | Marketing Platform |
| AutoGen / Agent Framework | Part 1 | AIHR |
| LlamaIndex | Part 1 | Answee |
| SmolAgents | Part 1 | EU Products (GDPR) |
| Agno | Part 1 | R&D |
| MetaGPT | Part 1 | Internal Tooling |

Every framework chapter uses the **same benchmark task** (niche research → validation → report) so you can directly compare approaches.

## Quick Start

### Run in Colab (no setup)

Every chapter notebook has an "Open in Colab" badge. Click and run.

### Run locally

```bash
git clone https://github.com/KnovikLLC/knovik-agentic-ai-playbook.git
cd knovik-agentic-ai-playbook
pip install -r requirements.txt
jupyter lab
```

### Build the book

```bash
pip install jupyter-book
jupyter-book build .
# Open _build/html/index.html
```

## Contributing

This book is a living document. Engineers are encouraged to:

- **Add recipes** — create a notebook in `notebooks/03_knovik_recipes/` and open a PR
- **Fix errors** — use the "suggest edit" button on any page in the online book
- **Add framework sections** — each framework has placeholder chapter files ready to fill in

See [CONTRIBUTING.md](CONTRIBUTING.md) for the contribution guide.

## Structure

```
knovik-ai-playbook/
├── _config.yml              # Jupyter Book config
├── _toc.yml                 # Table of contents
├── requirements.txt
├── notebooks/
│   ├── 00_foundations/      # What is agentic AI, the agent loop, setup
│   ├── 01_frameworks/       # One folder per framework
│   │   ├── langgraph/
│   │   ├── crewai/
│   │   ├── autogen/
│   │   ├── llamaindex/
│   │   ├── smol_agents/
│   │   ├── agno/
│   │   └── metagpt/
│   ├── 02_decision_layer/   # How to choose, comparison tables
│   ├── 03_knovik_recipes/   # Product-specific implementation guides
│   └── 04_production/       # Observability, cost, GDPR, deployment
└── .github/
    └── workflows/
        └── deploy.yml       # Auto-deploy to GitHub Pages on push to main
```

## License

MIT — open to the community.

---

**Owner:** Deelaka Kariyawasam, R&D Lead  
**Company:** [Knivok (Private) Limited](https://knovik.com) DBA Knovik
