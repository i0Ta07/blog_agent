# Blog Agent

An AI-powered blog generation system built with **LangGraph** that takes a topic and produces a fully written, research-backed Markdown blog post — complete with optional AI-generated images.

## Architecture

Blog Agent is structured around independent subgraphs, each with a single responsibility and clean I/O boundaries — much like microservices. The two subgraphs (Research, Reducer) can be reasoned about, tested, and extended in isolation. The main graph simply wires them together via shared state.

The agent follows a multi-stage pipeline:

```text
┌──────────────────────────────────────────────────────┐
│                    Blog Agent                        │
│                                                      │
│  topic                                               │
│    │                                                 │
│    ▼                                                 │
│ ┌──────────┐                                         │
│ │  Router  │                                         │
│ └────┬─────┘                                         │
│      │                                               │
│      ├── needs_research ──> ┌──────────────────────┐ │
│      │                      │  Research Subgraph   │ │
│      │                      │  searchInternet      │ │
│      │                      │       ↓              │ │
│      │                      │  scrape_web          │ │
│      │                      │       ↓              │ │
│      │                      │  summarise           │ │
│      │                      └──────────┬───────────┘ │
│      │                                 │             │
│      └─────────────────────────────────┘             │
│                        │                             │
│                        ▼                             │
│               ┌──────────────────┐                   │
│               │   Orchestrator   │                   │
│               └────────┬─────────┘                   │
│                        │ Send(task) × N              │
│              ┌─────────┼─────────┐                   │
│              ▼         ▼         ▼                   │
│           worker    worker    worker  (parallel)     │
│              └─────────┼─────────┘                   │
│                        │                             │
│                        ▼                             │
│            ┌───────────────────────┐                 │
│            │   Reducer Subgraph    │                 │
│            │  merge_content        │                 │
│            │       ↓               │                 │
│            │  gen_placeholders     │                 │
│            │       ↓               │                 │
│            │  generate_image × N   │                 │
│            │  (parallel)           │                 │
│            │       ↓               │                 │
│            │  formatCreateMD       │                 │
│            └───────────┬───────────┘                 │
│                        │                             │
│                        ▼                             │
│                  blog_post.md                        │
└──────────────────────────────────────────────────────┘
```

### 1. Router

Decides whether the topic needs web research before writing. Generates 3–8 targeted search queries when research is needed.

### 2. Research Subgraph *(conditional)*

Runs only when `needs_research = True`. Three sequential steps:

1. **`searchInternet`** — runs all queries concurrently via Tavily, deduplicates results
2. **`scrape_web`** — LLM selects up to 2 high-value URLs and scrapes them (supports PDF and HTML)
3. **`summarise_scraped_pages`** — distils scraped content into dense, implementation-relevant notes

### 3. Orchestrator

The most critical node in the pipeline. Uses structured output prompting to produce a `Plan` — a strict Pydantic schema of `Task` objects — that governs the entire downstream workflow. Section goals, word counts, code/citation flags, and research requirements are all defined here. Workers have no autonomy beyond what the Plan specifies.

### 4. Workers (parallel fan-out)

Each `Task` is dispatched to a `worker` node via LangGraph's `Send`. All workers run concurrently. Each writes one Markdown section following the tone, audience, and citation rules from the plan.

Results are collected via an `Annotated[List[str], add]` reducer on the `sections` state key.

### 5. Reducer Subgraph

1. **`merge_content`** — sorts sections by task ID and assembles the full Markdown document
2. **`generate_placeholders`** — LLM decides if images add value (max 2); inserts `[[IMAGE_N]]` placeholders
3. **`generate_image`** — generates images via `gpt-image-2` (fan-out, runs in parallel)
4. **`formatCreateMD`** — replaces placeholders with image tags, saves the final `.md` to `test_blogs/`

## Prompt Engineering

All prompts live in `prompts.py`, fully separated from graph logic. Each node uses a distinct prompting technique:

| Node                      | Technique                     | Purpose                                                       |
|---------------------------|-------------------------------|---------------------------------------------------------------|
| **Router**                | Classification prompting      | Categorises the topic to determine research mode              |
| **Orchestrator**          | Structured output prompting   | Produces the `Plan` schema that drives all downstream nodes   |
| **Workers**               | Role-based system prompts     | Writer persona with tone, audience, and citation rules        |
| **Scrape selector**       | Selection/ranking prompting   | Picks the 2 most authoritative URLs from candidates           |
| **Summariser**            | Compression prompting         | Condenses scraped content into dense, relevant notes          |
| **Placeholder generator** | Constraint-based prompting    | Decides image placement within a hard budget of max 2         |

## Async & Concurrent Workflow

Sequential blog generation would be O(n) — each section written one after another. Blog Agent eliminates this bottleneck in two places:

- **Worker fan-out** — the Orchestrator dispatches all `Task` objects simultaneously via LangGraph's `Send` API; all sections are written in parallel
- **Image generation fan-out** — multiple images are generated concurrently in the Reducer Subgraph

In practice, a 7-section blog runs in roughly the same time as a single section.

## Project Structure

```text
blog_agent/
├── main.ipynb
├── prompts.py
├── pyproject.toml
├── images/
└── test_blogs/
```

## Tech Stack

| Library               | Role                                                  |
|-----------------------|-------------------------------------------------------|
| `langgraph`           | Agent graph orchestration, fan-out with `Send`        |
| `langchain-openai`    | LLM calls (`gpt-4.1-mini`)                            |
| `openai`              | Image generation (`gpt-image-2`)                      |
| `langchain-tavily`    | Web search                                            |
| `langchain-community` | Web + PDF scraping (`WebBaseLoader`, `PyPDFLoader`)   |
| `pydantic`            | Structured outputs for all LLM responses              |
| `python-dotenv`       | API key management                                    |

## Setup

**Requirements:** Python 3.13+, [uv](https://github.com/astral-sh/uv)

```bash
git clone https://github.com/i0Ta07/blog_agent.git
cd blog_agent
uv sync
```

Create a `.env` file with your API keys:

```dotenv
OPENAI_API_KEY=your_openai_key
TAVILY_API_KEY=your_tavily_key
USER_AGENT="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0 Safari/537.36 BlogAgent/1.0"
```

Then open `main.ipynb` and run all cells. To generate a blog, invoke the graph at the bottom:

```python
response = await graph.ainvoke({"topic": "Your topic here"})
```

The output `.md` file will appear in `test_blogs/`.

## Key Design Decisions

- **Concurrent workers** — sections are written in parallel using LangGraph's `Send` API, reducing latency proportionally to the number of sections.
- **Structured outputs everywhere** — all LLM responses use `with_structured_output(PydanticModel)`, eliminating brittle string parsing.
- **Research is optional** — the router skips the research subgraph entirely for stable topics, saving time and cost.
- **Image budget** — at most 2 images per blog, only inserted when they reduce cognitive load (not for decoration).
- **Source authority filtering** — both the search and scrape steps use prompts that strongly prefer official docs, release notes, and primary sources over SEO content.

## Future Updates

- **Frontend** — UI to input a topic and display the generated blog
- **More prompt variants** — flexible tone, style, and audience controls for more diverse blog outputs
