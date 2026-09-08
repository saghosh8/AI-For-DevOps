# Day 14 — RAG Project: Ask RAG Anything

📦 **Project repo:** [ai-devops-release-assistant](https://github.com/saghosh8/ai-devops-release-assistant)

This day builds directly on top of the Day 7 mini project — same repo, no rewrite. Days 8–13 covered RAG concept by concept (chunking, embeddings, vector databases, retrieval). Today those pieces get wired into the existing assistant so it can answer questions grounded in **real repos**, not a scripted demo.

---

## Problem Statement

A DevOps assistant that only knows what's in its prompt is limited to whatever you paste in by hand. This project pulls real, live content — workflow YAMLs, pull requests, commits, and README docs — from actual GitHub repos, embeds it, indexes it in FAISS, and retrieves the most relevant chunks to ground an answer generated locally by Ollama. Ask it something like *"why would the release workflow fail on the docker build step?"* and it answers using what's actually in those repos, with sources cited — not a guess.

```mermaid
flowchart LR
    A["GitHub repos\n(YAML / PRs / commits / docs)"] --> B["Ingest"]
    B --> C["Chunk + Embed"]
    C --> D["FAISS index"]
    D --> E["Retriever"]
    E --> F["Ollama (local LLM)"]
    F --> G["Answer + sources"]

    classDef blue fill:#DBEAFE,stroke:#3B82F6,color:#1F2937
    classDef orange fill:#FFE8CC,stroke:#F97316,color:#1F2937
    classDef purple fill:#EDE9FE,stroke:#8B5CF6,color:#1F2937
    classDef green fill:#D1FAE5,stroke:#10B981,color:#1F2937

    class A blue
    class B orange
    class C orange
    class D purple
    class E purple
    class F orange
    class G green
```

---

## What Changed From the Sample-Data Version

Early builds of this pipeline ingested from a small folder of fixture data (`sample_repo_data/`) — enough to prove the pipeline worked, but not representative of a real project. That's been replaced:

- `ingest.py` now pulls live from the GitHub REST API — workflow YAMLs, PR titles/bodies, commit messages, and README — instead of reading local JSON/Markdown fixtures
- Ingestion targets **3 real repos**: [`release-automation`](https://github.com/saghosh8/release-automation), `application-one`, and `application-two`
- Retrieval and embedding logic (Days 10–12) didn't need to change — they operate on `Document` objects regardless of where those came from, which is the point of keeping ingestion behind one interface

---

## What You Need

- **Ollama** installed (or let the GitHub Actions workflow install and run it for you — no local setup required)
- A **GitHub token** with read access to the 3 target repos — the built-in Actions token only reads within the same repo, so a personal access token (`GH_PAT` secret, Contents + Pull requests read-only) is needed if those repos are private. Public repos work unauthenticated (lower rate limit)
- That's it — no paid API key, since generation runs on a local model via Ollama

---

## Running It — via GitHub Actions (no local setup)

1. If the 3 target repos are private, create a fine-grained PAT (Contents + Pull requests, read-only) and add it as a repo secret named `GH_PAT`
2. **Actions tab → "Ask RAG Anything" → Run workflow**
3. Type a question — e.g. *"what does the release workflow do on merge to the release branch?"*
4. Optionally restrict the answer to one content type (`yaml`, `pr`, `commit`, `doc`) using the filter input
5. Open the completed run — the answer and its sources are in the **Summary**

The first run builds the FAISS index from scratch (pass `--rebuild-index` is already the default in the workflow); later runs can reuse a cached index if you wire that up yourself.

---

# Input → Output

**Input** (typed into the `question` field when running the workflow):

![Input](../images/day14-input.png)

**Output** (rendered automatically in the GitHub Actions run Summary — answer plus cited sources like `release-automation/.github/workflows/prod_cd.yml` or `application-one PR #12`):

![Output](../images/day14-output.png)

---

## ⭐ Support

If you found this repository useful:

<a href="https://github.com/saghosh8/AI-For-DevOps">
  <img src="https://img.shields.io/github/stars/saghosh8/AI-For-DevOps?style=for-the-badge&logo=github&logoColor=white&label=STAR%20THIS%20REPO" />
</a>
<a href="https://github.com/saghosh8/AI-For-DevOps/fork">
  <img src="https://img.shields.io/github/forks/saghosh8/AI-For-DevOps?style=for-the-badge&logo=github&label=FORK" />
</a>

---

## 💬 Have a Query

Have a question, suggestion, or idea?

<a href="https://github.com/saghosh8/AI-For-DevOps/discussions/3">
  <img src="https://img.shields.io/badge/💬%20JOIN%20DISCUSSION-6366f1?style=for-the-badge&logo=github&logoColor=white" />
</a>

---

| 📘 Next — Day 15: AI Agents | <a href="https://github.com/saghosh8/AI-For-DevOps/blob/main/Week%203%20%E2%80%94%20Agents%2C%20Security%20%26%20Production/Day%2015%20%E2%80%94%20AI%20Agents.md"><img src="https://img.shields.io/badge/NEXT%20DAY-0ea5e9?style=for-the-badge&logo=github&logoColor=white" /></a> |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
