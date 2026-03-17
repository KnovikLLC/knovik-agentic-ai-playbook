# Environment Setup

---

## What you need before running any notebook

This chapter gets your local machine ready to run every notebook in this playbook. By the end you will have:

- Python 3.11 in a virtual environment
- All framework packages installed
- API keys configured in a `.env` file
- A working test that confirms everything is connected

---

## Prerequisites

- Python 3.11 (required — LangGraph and CrewAI have issues with 3.12+ as of mid-2025)
- Git
- A terminal (macOS/Linux) or PowerShell (Windows)

Check your Python version:
```bash
python --version  # should show 3.11.x
```

---

## Step 1 — Clone the playbook and create a virtual environment

```bash
git clone https://github.com/KnovikLLC/knovik-agentic-ai-playbook.git
cd knovik-agentic-ai-playbook

python -m venv .venv

# macOS / Linux
source .venv/bin/activate

# Windows
.venv\Scripts\activate
```

You should see `(.venv)` at the start of your terminal prompt.

---

## Step 2 — Install framework packages

Install all frameworks used in the playbook:

```bash
pip install --upgrade pip

# Core frameworks
pip install langgraph==0.2.35
pip install crewai==0.80.0 crewai-tools==0.14.0
pip install pyautogen==0.3.2
pip install llama-index==0.11.0 llama-index-llms-anthropic llama-index-embeddings-openai

# LLM providers
pip install langchain-anthropic langchain-openai langchain-community

# Utilities
pip install python-dotenv httpx tavily-python

# Jupyter
pip install jupyterlab
```

```{admonition} Pin versions in production
:class: note
The versions above are tested and known to work together.
Running `pip install langgraph` without a pin will give you the latest version,
which may have breaking API changes. Always pin in production.
```

---

## Step 3 — Get your API keys

You need at minimum one LLM provider key. The playbook defaults to **Anthropic Claude**.

### Anthropic (required)

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Create an account → Settings → API Keys → Create Key
3. Copy the key (starts with `sk-ant-...`)

Free tier: Not available. Minimum spend is $5. For running the notebooks in this playbook, $5 is sufficient for all exercises.

### Optional keys (needed for specific notebooks)

| Key | Where to get | Used in |
|-----|-------------|---------|
| `OPENAI_API_KEY` | platform.openai.com | CrewAI memory embedder, LlamaIndex |
| `SERPER_API_KEY` | serper.dev | Web search tool in CrewAI demos |
| `TAVILY_API_KEY` | tavily.com | Web search in LangGraph demos |
| `LANGSMITH_API_KEY` | smith.langchain.com | Observability (Part 4) |

Both Serper and Tavily have free tiers (2,500 searches/month and 1,000 searches/month respectively).

---

## Step 4 — Create your `.env` file

In the project root, create a file named `.env`:

```bash
# macOS/Linux
touch .env

# Windows
type nul > .env
```

Add your keys:

```dotenv
# Required
ANTHROPIC_API_KEY=sk-ant-...

# Optional — needed for specific notebooks
OPENAI_API_KEY=sk-...
SERPER_API_KEY=...
TAVILY_API_KEY=tvly-...
LANGSMITH_API_KEY=lsv2_...

# LangSmith project (optional, for observability)
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=knovik-playbook

# Knovik product APIs (only needed for recipe chapters)
STRAPI_URL=https://your-strapi-instance.com
STRAPI_TOKEN=...
GHOST_API_URL=https://your-ghost-blog.com
GHOST_ADMIN_API_KEY=...
```

```{admonition} Never commit .env
:class: warning
The `.gitignore` in this repo already excludes `.env`.
Double-check with `git status` before every commit.
Never put API keys in notebooks — they end up in git history.
```

---

## Step 5 — Load environment variables

Every notebook in this playbook starts with:

```python
from dotenv import load_dotenv
load_dotenv()
```

This loads your `.env` file into environment variables. All libraries (Anthropic, LangChain, CrewAI) automatically read from environment variables.

Verify it works:

```python
import os
from dotenv import load_dotenv
load_dotenv()

key = os.getenv("ANTHROPIC_API_KEY")
print("Key loaded:", key[:12] + "..." if key else "NOT FOUND")
```

---

## Step 6 — Test your setup

Run this in a Python file or notebook to confirm everything is working:

```python
import os
from dotenv import load_dotenv
from anthropic import Anthropic

load_dotenv()

client = Anthropic()
response = client.messages.create(
    model="claude-3-5-haiku-20241022",  # cheapest model, good for testing
    max_tokens=100,
    messages=[{"role": "user", "content": "Say 'environment configured' and nothing else."}]
)

print(response.content[0].text)
# Expected: environment configured
```

If you see `environment configured`, you are ready to run all notebooks.

---

## Using Google Colab instead

Every chapter has a "Open in Colab" button at the top. To use Colab:

1. Click the button — it opens the notebook in your Google Drive
2. At the top of the notebook, add a cell:
```python
!pip install langgraph crewai langchain-anthropic python-dotenv --quiet
```
3. Set your API key directly in the notebook session (not committed to git):
```python
import os
os.environ["ANTHROPIC_API_KEY"] = "sk-ant-..."  # paste your key here
```

```{admonition} Colab and API keys
:class: warning
Never save a Colab notebook to a shared drive or GitHub with API keys in it.
Use Colab Secrets (the key icon in the left panel) for persistent key storage in Colab.
```

---

## Project structure

```
knovik-agentic-ai-playbook/
├── notebooks/
│   ├── 00_foundations/          ← You are here
│   ├── 01_frameworks/
│   │   ├── langgraph/
│   │   ├── crewai/
│   │   ├── autogen/
│   │   ├── llamaindex/
│   │   ├── smol_agents/
│   │   ├── agno/
│   │   └── metagpt/
│   ├── 02_decision_layer/
│   ├── 03_knovik_recipes/
│   └── 04_production/
├── .env                         ← Your keys (not in git)
├── requirements.txt
└── _config.yml                  ← Jupyter Book config
```

---

## Troubleshooting

**`ModuleNotFoundError: No module named 'langchain_anthropic'`**
→ Your virtual environment is not activated. Run `source .venv/bin/activate` first.

**`AuthenticationError: invalid x-api-key`**
→ Your `.env` file is in the wrong directory, or `load_dotenv()` was not called.
Check: `python -c "from dotenv import load_dotenv; load_dotenv(); import os; print(os.getenv('ANTHROPIC_API_KEY')[:8])"`

**`ImportError` with `langgraph`**
→ Check your Python version. LangGraph requires Python 3.11. Run `python --version`.

**Package conflicts**
→ Create a fresh virtual environment: `deactivate`, `rm -rf .venv`, then redo steps 1-2.

---

## Before you continue

- [ ] Is `(.venv)` shown in your terminal prompt?
- [ ] Does the test in Step 6 print `environment configured`?
- [ ] Is `.env` listed in `.gitignore`?

If all three are yes, move on to Part 1 — Frameworks.
