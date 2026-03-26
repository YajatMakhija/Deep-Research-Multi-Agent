# Deep-Research-Multi-Agent
<img width="2776" height="596" alt="image" src="https://github.com/user-attachments/assets/4ed1307a-8403-4798-aaa9-64802fecc8e8" />

A multi-agent AI system that autonomously researches any topic and produces comprehensive, citation-rich reports — built from scratch with LangGraph, LangChain, Tavily, and Claude/GPT models.

<img width="1634" height="1332" alt="image" src="https://github.com/user-attachments/assets/b29297ab-4e53-4642-9a80-f7ddc3d651b9" />

Inspired by the [LangChain Academy](https://academy.langchain.com/) course on building a deep research agent from scratch.

---

## Overview

Given a research topic, the system orchestrates a full end-to-end workflow:

1. **Scopes** the research — clarifies ambiguities with the user and generates a detailed research brief
2. **Delegates** sub-topics to parallel researcher agents via a supervisor
3. **Searches** the web using Tavily, summarizes sources, and reflects on findings
4. **Compresses** each agent's raw notes into structured, citation-backed summaries
5. **Writes** a final long-form report with inline citations and a sources list

The entire pipeline is orchestrated as composable LangGraph state graphs with a supervisor pattern coordinating multiple independent researcher sub-agents.

---

## Architecture

```
User Input
    │
    ▼
┌──────────────────┐
│  Clarify & Scope │  ← Asks clarifying questions if needed
│  (research_agent_ │    Generates a detailed research brief
│   scope.py)       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   Supervisor     │  ← Breaks brief into sub-topics
│  (multi_agent_   │    Delegates via ConductResearch tool
│   supervisor.py) │    Launches parallel researcher agents
└────────┬─────────┘
         │
    ┌────┼────┐
    │    │    │
    ▼    ▼    ▼
┌──────┐┌──────┐┌──────┐
│Agent ││Agent ││Agent │  ← Each runs an independent
│  1   ││  2   ││  N   │    search → think → search loop
└──┬───┘└──┬───┘└──┬───┘    using Tavily + think_tool
   │       │       │
   └───────┼───────┘
           │
           ▼
┌──────────────────┐
│ Compress Research│  ← Deduplicates, cleans, preserves
│                  │    all findings with citations
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Final Report    │  ← Synthesizes everything into a
│  Generation      │    structured markdown report
└──────────────────┘
```

---

## Tech Stack

| Component          | Technology                                      |
|--------------------|--------------------------------------------------|
| Orchestration      | LangGraph (state graphs, conditional edges)      |
| Agent Framework    | LangChain                                        |
| Web Search         | Tavily API                                       |
| Supervisor LLM     | Anthropic Claude Sonnet 4                        |
| Research LLM       | Anthropic Claude Sonnet 4                        |
| Summarization      | OpenAI GPT-4.1-mini                              |
| Report Writer      | OpenAI GPT-4.1                                   |
| MCP Integration    | `langchain-mcp-adapters` (filesystem server)     |
| Language           | Python (async)                                   |

---

## Key Features

- **Supervisor Pattern** — A lead researcher agent plans, delegates, and evaluates progress before spawning sub-agents, preventing redundant or unfocused research.
- **Parallel Execution** — Multiple researcher agents run concurrently via `asyncio.gather`, with configurable concurrency limits.
- **Think Tool** — A deliberate reflection step after each search forces the agent to assess gaps and decide whether to continue or stop, improving research quality.
- **Research Compression** — Raw findings are compressed into clean, citation-backed summaries that preserve all relevant information while removing noise.
- **User Clarification Loop** — Before any research begins, the system evaluates whether the user's request is specific enough, and asks targeted follow-up questions if not.
- **MCP Support** — An alternate research agent (`research_agent_mcp.py`) integrates with MCP filesystem servers for local document research.
- **Structured Outputs** — Pydantic models enforce deterministic routing decisions (clarify vs. proceed, continue vs. stop).

---

## Project Structure

```
deep_research_from_scratch/
│
├── research_agent_full.py          # Full pipeline — entry point
├── research_agent_scope.py         # User clarification + research brief generation
├── multi_agent_supervisor.py       # Supervisor that delegates to parallel sub-agents
├── research_agent.py               # Individual researcher agent (web search)
├── research_agent_mcp.py           # Researcher agent with MCP filesystem access
│
├── state_scope.py                  # State definitions for scoping workflow
├── state_multi_agent_supervisor.py # State + tools for supervisor coordination
├── state_research.py               # State + schemas for researcher agents
│
├── prompts.py                      # All prompt templates
├── utils.py                        # Tavily search, summarization, think tool
└── files/                          # Local documents for MCP research (optional)
```

---

## Getting Started

### Prerequisites

- Python 3.10+
- API keys:
  - [Anthropic](https://console.anthropic.com/) (Claude Sonnet 4)
  - [OpenAI](https://platform.openai.com/) (GPT-4.1 / GPT-4.1-mini)
  - [Tavily](https://tavily.com/) (web search)

### Installation

```bash
git clone https://github.com/<your-username>/deep-research-from-scratch.git
cd deep-research-from-scratch

python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt
```

### Environment Variables

Create a `.env` file:

```env
ANTHROPIC_API_KEY=your_anthropic_key
OPENAI_API_KEY=your_openai_key
TAVILY_API_KEY=your_tavily_key
```

### Usage

```python
from deep_research_from_scratch.research_agent_full import agent

# Run the full pipeline
result = await agent.ainvoke({
    "messages": [{"role": "user", "content": "Research the current state of quantum computing and its impact on cybersecurity"}]
})

print(result["final_report"])
```

---

## Agent Workflow in Detail

### 1. Scoping Phase (`research_agent_scope.py`)
The system uses structured output (`ClarifyWithUser` schema) to decide whether to ask the user a clarifying question or proceed directly. If the request is clear enough, it generates a `ResearchQuestion` brief that guides all downstream agents.

### 2. Supervisor Phase (`multi_agent_supervisor.py`)
The supervisor receives the brief and uses three tools:
- **`think_tool`** — Reflects on strategy before delegating
- **`ConductResearch`** — Spawns a researcher sub-agent for a specific sub-topic
- **`ResearchComplete`** — Signals that all needed research is done

Up to 3 sub-agents run in parallel per iteration, with a maximum of 6 total iterations.

### 3. Research Phase (`research_agent.py`)
Each sub-agent runs an independent loop:
- Searches the web via Tavily (with raw content extraction and summarization)
- Reflects on findings using `think_tool`
- Repeats until it can answer confidently or hits the tool call budget
- Compresses its findings into a structured summary with inline citations

### 4. Report Generation (`research_agent_full.py`)
All compressed findings are aggregated and passed to the writer model, which produces a final markdown report with proper headings, inline citations, and a sources section.

---

## Configuration

Key parameters can be tuned in `multi_agent_supervisor.py`:

| Parameter                    | Default | Description                                      |
|------------------------------|---------|--------------------------------------------------|
| `max_researcher_iterations`  | 6       | Max tool call rounds per supervisor cycle         |
| `max_concurrent_researchers` | 3       | Max parallel sub-agents per delegation round      |

Individual researcher agents have their own budgets defined in `prompts.py` (2–5 search calls depending on query complexity).

---

## Acknowledgments

- [LangChain Academy](https://academy.langchain.com/) — Course material and multi-agent design patterns
- [LangGraph](https://github.com/langchain-ai/langgraph) — State graph orchestration
- [Tavily](https://tavily.com/) — AI-optimized search API
- [Anthropic](https://www.anthropic.com/) & [OpenAI](https://openai.com/) — LLM providers

---

## License

This project is open source under the [MIT License](LICENSE).


