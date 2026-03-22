# CLAUDE.md

This file provides context to Claude Code when working on this project.

## Project Overview

**Purpose**: This is a course we built together to teach how to build agentic applications. 

**Problem Domain**: ["Visual Language Models with Agents", "Agent Tool Calling", "Building and evaluating Agents", "Building and evaluating Agentic Systems"]

**Key Technologies**:
- PyTorch (ROCm-accelerated via devcontainer)
- [Add your frameworks: [transformers, 	]
- [Add your tools: MLflow, FastAPI, SmolAgents SMolVLM, OpenAI API,Qwen2.5-VL, vLLM]

## Codebase Structure

```
src/
├── data/           # Data loading, preprocessing, augmentation
├── models/         # Model architectures and training logic
└── course/    		# All the actual course material
```

**Key files**:
- `course/README.md` - main material introducing the course material
- `course/curriculum.md` - Outline of the course
- `course/GETTING_STARTED.md` - How to get started with the material

## Development Workflow

This is a bit different than a normal software project. While I will be creating my own examples in the src/ directory, you are mostly helping me to understand the material and review
my work. I am not going to want you to write all the code for me. I may need some assistance but I am looking to you more for back and forth discussion, code review, and help producing summary
documentation/diagrams/cheatsheets that I can use to remember what I learn. 

**Linting/Formatting**:
```bash
ruff check src/        # Lint code
ruff format src/       # Format code
ruff check --fix src/  # Auto-fix linting issues
```

## Architectural Decisions

Document key design choices that Claude should understand:

- We like clear separation of concerns. 
- There are several patterns for using LLMs ov VLMs with agents. Please make sure to include this article in the curriculum https://towardsdatascience.com/the-multi-agent-trap/

## Important Patterns


## Testing Strategy



---

**Note**: This is a ROCm devcontainer project. For ROCm-specific troubleshooting (GPU access, dependency conflicts, Python version issues), see `template_docs/CLAUDE.md`.
