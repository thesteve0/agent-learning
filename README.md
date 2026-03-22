# agent-learning: VLM Agent/Tool-Calling Course

> A self-contained curriculum for building production-ready VLM agent systems using Smolagents, vLLM, FastAPI, and MLflow. The concrete example is Stardew Vision — an accessibility tool that reads Stardew Valley UI screenshots — but the patterns transfer to any domain.

## Problem Statement

Most developers jumping into agentic AI skip straight to multi-agent frameworks before understanding the fundamentals. This course teaches agent/tool-calling patterns from the ground up: starting with raw manual dispatch, then adding frameworks only where complexity justifies them. A key theme throughout is compound reliability — unstructured multi-agent systems can amplify errors up to 17x, so you learn to measure before you add complexity.

## What This Course Builds

The Stardew Vision application: a VLM-powered agent that accepts Stardew Valley screenshots and extracts structured data (item names, prices, quantities) from UI panels. Built progressively across 7 modules from raw Python to a production FastAPI service.

## Course Structure

```
agent-learning/
├── course/
│   ├── curriculum.md                  # Full 7-module learning path
│   ├── GETTING_STARTED.md             # Session-by-session guide
│   ├── modules/                       # Detailed guides with diagrams
│   │   ├── module1-manual-dispatch.md
│   │   ├── module2-smolagents-basics.md
│   │   ├── module3-vllm-integration.md
│   │   ├── module4-production-wrapper.md
│   │   ├── module5-evaluation.md
│   │   ├── module6-fastapi-integration.md
│   │   └── module7-conference-demos.md
│   ├── examples/                      # Runnable code for each module
│   ├── exercises/                     # Hands-on challenges
│   ├── diagrams/                      # ASCII architecture diagrams
│   ├── docs/                          # Quickstart, best practices, framework decisions
│   └── resources/                     # Cheatsheets, sources, The Multi-Agent Trap PDF
└── src/                               # Your working code goes here
```

## Modules

| # | Topic | Key Skill | Time |
|---|-------|-----------|------|
| 1 | Manual Tool Dispatch | OpenAI function-calling format, no framework | 1-2 hr |
| 2 | Smolagents Basics | `Tool` class, `CodeAgent` vs `ToolCallingAgent` | 2-3 hr |
| 3 | Smolagents + vLLM | `LiteLLMModel`, local GPU serving | 2-3 hr |
| 4 | Production Wrapper | `VLMOrchestrator`, error handling, MLflow, unit tests | 2-3 hr |
| 5 | Evaluation | Compound reliability, step/field/e2e accuracy, MLflow baselines | 1.5-2 hr |
| 6 | FastAPI Integration | Web endpoints, file uploads, async patterns | 1-2 hr |
| 7 | Advanced & Alternatives | LangGraph/CrewAI decision framework, conference demos | 1-2 hr |

**Total**: 8-12 hours across 3-4 sessions

## Key Technologies

- **Smolagents** (HuggingFace) — primary agent framework
- **Qwen2.5-VL-7B-Instruct** — vision language model
- **vLLM** — local GPU serving with OpenAI-compatible API
- **FastAPI** — web API layer
- **MLflow** — experiment tracking and observability
- **PyTorch** with ROCm acceleration (devcontainer)

## Setup

This project runs in a ROCm devcontainer. Prerequisites are already configured if you're inside the container.

**Install dependencies**:
```bash
uv sync
uv add 'smolagents[toolkit,litellm]'
```

**Start vLLM server** (required for modules 3+):
```bash
vllm serve models/base/Qwen2.5-VL-7B-Instruct \
  --port 8001 --dtype float16 --enable-tool-calling
```

**Verify**:
```bash
curl http://localhost:8001/v1/models
python -c "import smolagents; print(smolagents.__version__)"
```

## Where to Start

Read `1-START-HERE-IN-THE-MORNING.md`, then follow `course/GETTING_STARTED.md`.

Work through modules **sequentially** — Module 1 exists so the later abstractions make sense.

## Core Principle

> "Start simple, measure everything, add complexity only when the measurement justifies it."

See `course/resources/TheMulti-Agent Trap _ TowardsDataScience.pdf` for the research behind this.

## Linting

```bash
ruff check src/
ruff format src/
ruff check --fix src/
```

---

This project was created from [datascience-template-ROCm](https://github.com/thesteve0/datascience-template-ROCm). For ROCm setup and troubleshooting, see `template_docs/`.
