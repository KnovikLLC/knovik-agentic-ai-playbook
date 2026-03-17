# Contributing to the Knovik AI Engineering Playbook

This book improves when engineers contribute from real project experience. Here is how.

---

## Ways to contribute

### 1. Fix a typo or error
Use the "suggest edit" button on any page in the online book. It opens a GitHub edit interface directly.

### 2. Add a recipe
Recipes in `notebooks/03_knovik_recipes/` are the most valuable contribution. A good recipe:
- Solves a real Knovik product problem
- Has working, runnable code
- Explains why the framework choice was made
- Includes a "Production notes" section at the end

**Template:**
```
notebooks/03_knovik_recipes/NN_your_recipe_name.ipynb
```

Copy the structure from `01_answee_qa_agent.ipynb`.

### 3. Complete a placeholder chapter
Many framework chapters have placeholder files (e.g., `langgraph/02_state_and_nodes.md`).
Pick one, fill it in, open a PR.

### 4. Update the framework matrix
When a framework releases a significant update, update `notebooks/02_decision_layer/04_framework_matrix.md`.

---

## Pull Request process

1. Fork the repo
2. Create a branch: `git checkout -b add/recipe-aihr-workflow`
3. Add your notebook or markdown file
4. Add the file to `_toc.yml` in the right place
5. Run `jupyter-book build .` locally to check nothing is broken
6. Open a PR with a clear description

**PR title format:**
- `add: recipe for [product] [feature]`
- `fix: [what was wrong] in [chapter]`
- `update: [framework] chapter for v[version]`

---

## Notebook standards

- First cell: Colab badge + chapter description (markdown)
- Second cell: `!pip install -q ...` — all dependencies
- Third cell: imports + API key setup
- Remaining cells: theory → code → output → exercise pattern
- Last cell: "Try it yourself" or "Production notes"

**Do not commit API keys or secrets.** Use `getpass` for all keys in notebooks.

---

## Questions

Open an issue or ping Deelaka on Teams.
