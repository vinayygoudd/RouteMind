# 🧭 RouteMind

### A self-correcting agentic RAG system that decides where to look, judges its own evidence, and fixes itself before it answers.

> **One line:** RouteMind routes every question to the right source (documents, SQL database or the web), grades what it retrieves, rewrites weak queries, checks the final answer against the evidence, and only then replies, all on a laptop CPU with no API keys.



## 📑 Table of Contents

1. [The Problem](#-the-problem)
2. [The Solution at a Glance](#-the-solution-at-a-glance)
3. [Tech Stack](#-tech-stack)
4. [How a Question Flows](#-how-a-question-flows)
5. [The Five Agentic Behaviours](#-the-five-agentic-behaviours)
6. [Three Sources, One Router](#-three-sources-one-router)
7. [Quickstart](#-quickstart)
8. [Bring Your Own Data](#-bring-your-own-data)
9. [Evaluation](#-evaluation)
10. [Design Decisions](#-design-decisions)
11. [Project Structure](#-project-structure)
12. [Roadmap](#-roadmap)
13. [License](#-license)

---

## 🎯 The Problem

A basic RAG system retrieves chunks and generates an answer, whatever it found. It breaks in predictable ways:

- It searches the **wrong source**. Numeric questions ("which product sold most?") need exact SQL, not text chunks.
- It feeds the LLM **irrelevant context** and gets confident nonsense.
- It never notices when its answer is **not supported by the evidence**.
- It can loop forever or fail silently when retrieval comes up empty.

## ✅ The Solution at a Glance

RouteMind replaces the fixed retrieve-then-generate pipeline with an **agentic AI** state machine that makes explicit, inspectable decisions at every step.

| Capability | What it does |
|---|---|
| **Adaptive routing** | Picks documents, SQL or web search per question |
| **Retrieval grading** | Judges every retrieved chunk for relevance and discards the bad ones |
| **Query rewriting** | Reformulates the question when nothing relevant is found |
| **Grounding check** | Verifies the answer is supported by evidence before returning it |
| **Bounded self-correction** | Every loop has a retry budget, so the graph always terminates |
| **Honest failure** | Says "I don't know" instead of filling gaps from memory |

**Runs 100% locally on CPU. No API keys.**

---

## 🧱 Tech Stack

| Layer | Role in this project |
|---|---|
| **Python** | The whole system: agent graph, ingestion, command-line tool, evaluation, tests |
| **SQL** | Exact answers for tabular and numeric questions, with generated queries validated as read-only and self-corrected on failure |
| **LLMs** | A small CPU-friendly local model runs the agent. A larger one is used only as the evaluation judge. |
| **RAG** | Multilingual embeddings and a vector store for retrieval over your documents |
| **Agentic AI** | Routing, grading, rewriting and verification as an explicit stateful graph |
| **GenAI** | Answer generation grounded strictly in retrieved evidence, with sources |
| **Machine Learning** | Embedding models for semantic search, plus metric-driven evaluation of the pipeline |

---

## 🔀 How a Question Flows

```text
                        ┌───────────┐
        question ─────► │   route   │
                        └─────┬─────┘
          ┌───────────────────┼────────────────────┐
          ▼                   ▼                    ▼
   ┌─────────────┐     ┌─────────────┐      ┌─────────────┐
   │  retrieve   │     │ web_search  │      │  query_sql  │
   │ (documents) │     │ (recent)    │      │ (structured)│
   └──────┬──────┘     └──────┬──────┘      └──────┬──────┘
          ▼                   │                    │
   ┌─────────────┐            │                    │
   │ grade_docs  │            │                    │
   └──┬───────┬──┘            │                    │
 relevant│   │nothing relevant│                    │
          │   ├─ retries left ─► rewrite_query ─► (back to retrieve)
          │   └─ budget spent ─► web_search       │
          ▼                   ▼                    ▼
        ┌───────────────────────────────────────────┐
        │                  generate                  │
        └─────────────────────┬─────────────────────┘
                              ▼
                   ┌────────────────────┐
                   │ grounded in evidence? │
                   └───┬────────────┬───┘
                   yes │            │ no
                       ▼            ├─ retries left ─► rewrite_query
              answer + sources      └─ budget spent ─► answer (flagged)
```

---

## 🧠 The Five Agentic Behaviours

1. **Adaptive routing.** A structured LLM decision sends each question to documents, the SQL database or web search.
2. **Retrieval grading.** Every retrieved chunk is judged for relevance. Irrelevant chunks never reach generation.
3. **Query rewriting.** If nothing relevant survives grading, the query is reformulated and retried, then falls back to web search.
4. **Hallucination check.** The answer is verified against the evidence. Ungrounded answers trigger self-correction.
5. **Guaranteed termination.** All loops are capped by a retry budget (default **2**), so there are no runaway LLM calls.

---

## 🗂 Three Sources, One Router

| Question | Route | Why |
|---|---|---|
| "What is retrieval grading and why is it useful?" | 📚 Documents (RAG) | Answer lives in your knowledge base |
| "Which product had the most sales in Spain?" | 🗄️ SQL | Needs exact aggregation, not LLM arithmetic over text |
| "What happened in the news today?" | 🌐 Web search | Needs fresh information |

---

## 🚀 Quickstart

**Requirements:** Python 3.10+ and a local LLM runtime.

```bash
# 1. Pull the small local model (~2 GB, CPU-friendly) into your local LLM runtime

# 2. Install
git clone <your-repo-url> routemind
cd routemind
python -m venv .venv && source .venv/bin/activate
pip install -e ".[ui,eval,dev]"

# 3. Create the sample database, load sample tables, index sample documents
python scripts/create_sample_db.py
agentic-rag-ingest-csv
agentic-rag-ingest

# 4. Ask questions
streamlit run app.py                         # web UI with live agent steps
agentic-rag -v "What is retrieval grading?"  # command line
```

**Docker:** `docker compose up -d`, pull the model once inside the model container, then open `http://localhost:8501`.

---

## 📦 Bring Your Own Data

**Documents** (`.md`, `.txt`, `.pdf`)

1. Drop them into `data/documents/`.
2. Run `agentic-rag-ingest`.
3. Update `KB_DESCRIPTION` in `.env` so the router knows what the knowledge base covers.

**Tables** (`.csv`, `.xlsx`)

1. Drop them into `data/tables/`.
2. Run `agentic-rag-ingest-csv`. Each file becomes a SQL table named after the file.
3. Copy the printed `SQL_DESCRIPTION` line into `.env` so numeric questions route to SQL.

> **Why separate paths?** Text embeddings are the wrong tool for aggregations. Exact SQL is.

---

## 📊 Evaluation

The harness runs the full agent over a question set (`evaluation/dataset.json`) and scores it with four metrics:

```bash
python evaluation/run_evaluation.py
```

| Metric | What it diagnoses | Score |
|---|---|---|
| **Faithfulness** | Is every claim supported by the evidence? (hallucination) | **0.89** |
| **Response relevancy** | Does the answer address the question? | **0.81** |
| **Context precision** | Are the best chunks ranked first? (retriever ranking) | **0.88** |
| **Context recall** | Does retrieval contain everything needed? | **0.77** |

*Measured with a 3B-parameter agent on CPU and a 7B-parameter judge.*

**How to read them together**

- Low recall + high faithfulness: retrieval is the bottleneck.
- High recall + low faithfulness: the generator is ignoring its evidence.

**The judge matters (measured).** A 3B model judging itself scored faithfulness **0.50**. A 7B judge rescored the *same answers* at **0.83**. The agent was not hallucinating; the small judge could not verify claims. The fix is to keep the small model for the agent and use the larger one only for offline evaluation.

> Treat scores as **regression signals** across changes, not absolute truth. Read the per-question results file (`evaluation/results.csv`), not just the averages.

---

## 🧩 Design Decisions

| Decision | Reasoning |
|---|---|
| **Graders reason before judging** | Forcing an instant yes/no from a small LLM made it reject relevant chunks. A short reasoning step followed by a parsed verdict fixed it. |
| **Parse failures default to the safe verdict** | Grading noise can never discard all evidence or trap the graph in a loop. |
| **Every loop is bounded** | The graph always terminates. |
| **CPU latency shaped the design** | Each grader is an extra LLM call. Only the two highest-value checks (relevance and grounding) are kept. |
| **SQL is read-only by construction** | Queries must be a single retrieval statement with no data-changing keywords. |
| **SQL self-corrects and can use a bigger model** | Syntax errors and suspicious empty results are fed back for a bounded number of retries. A larger model can be assigned to SQL alone. |
| **Honest failure over confident nonsense** | If a source returns nothing, the generator says "I don't know". |
| **Testable control flow** | Routing decisions are pure functions, unit-tested with no model running. |

---

## 🗂 Project Structure

```text
routemind/
├── app.py                      Web chat interface with live agent steps
├── src/agentic_rag/
│   ├── config.py               Settings from environment variables
│   ├── ingestion.py            Load → chunk → embed → index
│   ├── tabular.py              CSV/Excel → SQL tables
│   ├── cli.py                  Command-line interface
│   ├── graph/                  State, router/graders/rewriter/generator, nodes, assembly
│   └── tools/                  Web search source, read-only SQL source
├── evaluation/                 Question set and evaluation harness
├── data/documents/             Knowledge base
├── scripts/create_sample_db.py Sample database
├── tests/                      Control-flow and safety tests
├── Dockerfile
└── docker-compose.yml
```

---

## 🛣 Roadmap

- [ ] **Plan-and-execute router:** chain sources for questions like "does our best-selling product have good reviews online?" (SQL, then web)
- [ ] **Graph-based retrieval route:** answer questions whose evidence lives in relationships across many documents
- [ ] **SQL intent grader:** verify that the generated query matches what the user actually asked
- [ ] Hybrid keyword and semantic search with reranking
- [ ] Multi-turn conversation memory
- [ ] Observability and tracing
- [ ] Unanswerable-question tests to verify "I don't know" behaviour
- [ ] Synthetic evaluation-set generation

---

