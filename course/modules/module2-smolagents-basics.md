# Module 2: Smolagents Basics

**Duration**: 2-3 hours
**Prerequisites**: Module 1 completed, basic Python OOP
**Example Code**: [`examples/module2_smolagents_basic.py`](../examples/module2_smolagents_basic.py)

---

## Learning Objectives

By the end of this module, you will:

- ✅ Install and configure Smolagents
- ✅ Create custom `Tool` classes
- ✅ Understand `CodeAgent` vs `ToolCallingAgent`
- ✅ Run your first agent with a transformers backend
- ✅ See how Smolagents automates what you did manually in Module 1

---

## Why Smolagents?

After doing manual dispatch in Module 1, you might wonder: *"Why not just keep doing it manually?"*

**Here's why frameworks matter**:

### What You Wrote Manually (Module 1)

```python
# Parse VLM response
tool_call = vlm_response["tool_call"]
tool_name = tool_call["function"]["name"]
arguments = json.loads(tool_call["function"]["arguments"])

# Validate tool exists
if tool_name not in TOOL_REGISTRY:
    raise ValueError(f"Unknown tool: {tool_name}")

# Dispatch
result = TOOL_REGISTRY[tool_name](**arguments)
```

That's **~10 lines** every time you want to call a tool.

### What Smolagents Does

```python
agent = CodeAgent(tools=[MyTool()], model=model)
result = agent.run("Extract from screenshot")
```

That's **2 lines**. Smolagents handles:
- ✅ Tool definition generation (from Tool class)
- ✅ Prompt construction (system prompt + user message)
- ✅ VLM invocation (with retries, error handling)
- ✅ Response parsing (JSON or Python code)
- ✅ Tool dispatch (lookup and execution)
- ✅ Multi-step reasoning (if task needs multiple tools)
- ✅ Error recovery (retry on failures)

**The win**: You focus on **what** to do, not **how** to dispatch.

---

## Smolagents Design Philosophy

Smolagents was designed specifically for **VLM (Vision-Language Model)** use cases:

### 1. VLM-First Design

**Unlike LangChain** (retrofitted for vision):
- ✅ First-class support for images, video, audio
- ✅ Optimized for multi-modal prompts
- ✅ Vision model-specific optimizations

**Example**:
```python
# Smolagents handles images natively
agent.run("Analyze this screenshot", image="/path/to/file.png")

# LangChain requires manual image encoding
# image_data = encode_image(image_path)
# agent.run(f"Analyze this: data:image/png;base64,{image_data}")
```

### 2. Model-Agnostic

Works with:
- ✅ Local models (Qwen, Llama, SmolVLM)
- ✅ OpenAI (GPT-4V, GPT-4o)
- ✅ Anthropic (Claude Sonnet)
- ✅ HuggingFace Inference API
- ✅ Any OpenAI-compatible endpoint (vLLM, TGI)

**Same code, different backend**:
```python
# Local vLLM
from smolagents import OpenAIServerModel
model = OpenAIServerModel(model_id="Qwen2.5-VL", base_url="http://localhost:8001/v1", api_key="EMPTY")

# OpenAI
model = OpenAIServerModel(model_id="gpt-4o", api_key=os.getenv("OPENAI_API_KEY"))

# Same agent code works with both!
agent = CodeAgent(tools=[...], model=model)
```

### 3. CodeAgent Paradigm

**ToolCallingAgent** (JSON-based, like most frameworks):
```json
{
  "tool": "my_tool",
  "arguments": {"x": 1, "y": 2}
}
```

**CodeAgent** (Python-based, unique to Smolagents):
```python
x = 1
y = 2
result = my_tool(x=x, y=y)
print(result)
```

**Why Python > JSON?**
- ✅ Type-checked by Python runtime (catches errors early)
- ✅ Composable (can chain tools: `y = tool1(x); z = tool2(y)`)
- ✅ Debuggable (Python stack traces vs JSON parse errors)
- ✅ Self-correcting (VLM can fix syntax errors)

See diagram: [`diagrams/codeagent-execution.txt`](../diagrams/codeagent-execution.txt)

### 4. Minimal Complexity

**Smolagents**: ~1,000 lines of core code
**LangChain**: ~50,000+ lines
**CrewAI**: ~10,000 lines

**Implication**: You can read and understand the entire Smolagents codebase in an afternoon.

When something breaks, you can debug it. When you need custom behavior, you can fork/extend it.

---

## Installation

```bash
# Install with all backends
uv add 'smolagents[toolkit]'

# Verify installation
python -c "import smolagents; print(smolagents.__version__)"
```

**Extras explained**:
- `toolkit`: Includes default tools (web search, Python REPL, etc.)

**If you get import errors**:
```bash
# Reinstall with correct extras
uv remove smolagents
uv add 'smolagents[toolkit]'
```

**Note on LiteLLM**: Due to a recent supply chain attack (March 2026), we've removed LiteLLM from this course. Module 3 now uses `OpenAIServerModel` instead, which provides the same functionality for connecting to vLLM and other OpenAI-compatible endpoints without the compromised dependency.

---

## Core Concepts

### Concept 1: Tool Classes

In Module 1, you had two separate things:
- Tool definition (JSON)
- Tool implementation (function)

In Smolagents, they're **unified** in a `Tool` class:

```python
from smolagents import Tool

class MyTool(Tool):
    # METADATA (replaces JSON definition)
    name = "my_tool"
    description = "What this tool does"
    inputs = {
        "param1": {"type": "string", "description": "..."},
        "param2": {"type": "integer", "description": "..."}
    }
    output_type = "object"  # str, dict, Image, etc.

    # IMPLEMENTATION (replaces function in registry)
    def forward(self, param1: str, param2: int):
        """This is called when tool is invoked."""
        result = do_something(param1, param2)
        return result
```

**Benefits**:
- ✅ Single source of truth (metadata + implementation together)
- ✅ Auto-generated tool definitions (Smolagents converts to JSON)
- ✅ Type hints in `forward()` method (IDE autocomplete!)
- ✅ Reusable (import and share across projects)

**Comparison to Module 1**:

| Module 1 (Manual)                     | Module 2 (Smolagents)           |
|---------------------------------------|---------------------------------|
| JSON definition + function in registry| Single Tool class               |
| Manually keep in sync                 | Auto-sync (name from class attr)|
| No IDE support                        | Type hints in forward()         |

### Concept 2: Models

Smolagents supports multiple model backends via a unified interface:

```python
# InferenceClientModel: HuggingFace transformers or Inference API
from smolagents import InferenceClientModel
model = InferenceClientModel(model_id="Qwen/Qwen2.5-VL-7B-Instruct")

# OpenAIServerModel: OpenAI-compatible endpoints (vLLM, TGI, OpenAI, etc.)
from smolagents import OpenAIServerModel
model = OpenAIServerModel(
    model_id="Qwen2.5-VL-7B-Instruct",
    base_url="http://localhost:8001/v1",
    api_key="EMPTY"
)

# Or for actual OpenAI API
model = OpenAIServerModel(model_id="gpt-4o", api_key="sk-...")
```

**When to use which**:
- `InferenceClientModel`: Quick testing, no vLLM setup needed (slower)
- `OpenAIServerModel`: Production use with vLLM, OpenShift AI, or OpenAI API (fast, recommended)

**For this tutorial**: We start with `InferenceClientModel` (Module 2), then switch to `OpenAIServerModel` with vLLM (Module 3).

**Security Note**: We previously used `LiteLLMModel`, but removed it due to a March 2026 supply chain attack. `OpenAIServerModel` provides the same functionality without the compromised dependency.

### Concept 3: CodeAgent vs ToolCallingAgent

**Two agent types** in Smolagents:

#### ToolCallingAgent (JSON-based)

```python
from smolagents import ToolCallingAgent

agent = ToolCallingAgent(tools=[...], model=model)
result = agent.run("Do something")
```

**How it works**:
1. VLM returns JSON: `{"tool": "my_tool", "arguments": {...}}`
2. Agent parses JSON
3. Agent calls tool

**Pros**: Simple, widely compatible
**Cons**: Less composable, weaker type safety

#### CodeAgent (Python-based) ⭐ Recommended

```python
from smolagents import CodeAgent

agent = CodeAgent(tools=[...], model=model)
result = agent.run("Do something")
```

**How it works**:
1. VLM returns Python code: `result = my_tool(x=1, y=2)`
2. Agent executes code in sandbox
3. Agent returns result

**Pros**: Composable, type-safe, self-correcting
**Cons**: Requires code execution sandbox

**We use CodeAgent** for this project. See full comparison: [`diagrams/codeagent-execution.txt`](../diagrams/codeagent-execution.txt)

---

## How Smolagents Constructs Prompts

Understanding **what the VLM actually sees** is crucial for debugging and writing good tools.

### From Tool Class to System Prompt

When you create a Tool class:
```python
class CropPierresPanelTool(Tool):
    name = "crop_pierres_detail_panel"
    description = "Extract item details from Pierre's General Store detail panel"
    inputs = {
        "image_path": {"type": "string", "description": "Path to screenshot"}
    }
    output_type = "object"

    def forward(self, image_path: str):
        # Implementation...
```

Smolagents **auto-generates** a tool signature for the VLM:
```
crop_pierres_detail_panel(image_path: str) -> object
  Extract item details from Pierre's General Store detail panel

  Parameters:
    - image_path (str): Path to screenshot
```

### The Message Structure

When you call `agent.run("Extract from screenshot.png")`, Smolagents sends a **chat message array** to the VLM:

```python
messages = [
    {
        "role": "system",
        "content": """You are a Python coding assistant.

Available tools:
  - crop_pierres_detail_panel(image_path: str) -> object
      Extract item details from Pierre's General Store detail panel

      Parameters:
        - image_path (str): Path to screenshot

Write Python code to call these tools to complete the user's task.
Output only executable Python code, no explanations."""
    },
    {
        "role": "user",
        "content": "Extract from screenshot.png"
    }
]
```

**Key insights**:
- **System message**: Contains tool definitions + instructions
- **User message**: Your natural language task
- **Separation**: They're distinct messages, not concatenated
- **Standard format**: This is OpenAI chat API format (works with vLLM, OpenAI, Anthropic, etc.)

### What This Means for Tool Design

Since tool metadata becomes the VLM's documentation, **be specific**:

**Bad** (vague):
```python
description = "Processes images"
inputs = {"path": {"type": "string", "description": "File"}}
```

**Good** (specific):
```python
description = "Extract item details from Pierre's General Store detail panel in a Stardew Valley screenshot"
inputs = {
    "image_path": {
        "type": "string",
        "description": "Absolute path to the screenshot file (PNG or JPG)"
    }
}
```

The VLM uses these descriptions to decide:
1. **Which tool to call** (based on description matching user's task)
2. **What arguments to pass** (based on parameter descriptions)

### Multi-Step Conversations

For complex tasks, the message array grows:

**Step 1**:
```python
[
    {"role": "system", "content": "...tool definitions..."},
    {"role": "user", "content": "Extract from screenshot.png"}
]
# VLM generates: result = crop_pierres_detail_panel(...)
```

**Step 2** (if agent needs more reasoning):
```python
[
    {"role": "system", "content": "...tool definitions..."},
    {"role": "user", "content": "Extract from screenshot.png"},
    {"role": "assistant", "content": "result = crop_pierres_detail_panel(...)"},
    {"role": "user", "content": "Execution result: {'name': 'Parsnip', ...}"}
]
# VLM sees previous code + result, decides if task is complete
```

**The loop continues** until:
- VLM calls `final_answer()` function
- `max_steps` reached
- Error occurs

---

## Understanding forward() Methods

The `forward()` method is where your tool's logic lives. **Keep it simple**, but add logic when needed.

### Pattern: Thin Wrapper (Recommended)

```python
class MyTool(Tool):
    name = "my_tool"
    # ... metadata ...

    def forward(self, arg: str):
        # Just call your real implementation
        from my_module import my_actual_function
        return my_actual_function(arg)
```

**Why thin?**
- Separates tool interface (Smolagents) from implementation (your code)
- Easier to test implementation independently
- Can reuse implementation outside of agents

### When to Add Logic in forward()

#### 1. Error Handling (Most Common)
```python
def forward(self, image_path: str):
    try:
        from stardew_vision.tools import crop_pierres_detail_panel
        return crop_pierres_detail_panel(image_path)
    except FileNotFoundError:
        return {"error": "Screenshot file not found"}
    except Exception as e:
        logger.error(f"Tool failed: {e}")
        return {"error": str(e)}
```

**Why**: VLMs handle structured error messages better than raw exceptions.

#### 2. Input Validation
```python
def forward(self, image_path: str):
    if not Path(image_path).exists():
        return {"error": f"File does not exist: {image_path}"}

    if not image_path.endswith(('.png', '.jpg', '.jpeg')):
        return {"error": "Only PNG/JPG images supported"}

    from stardew_vision.tools import crop_pierres_detail_panel
    return crop_pierres_detail_panel(image_path)
```

**Why**: Fail fast, avoid wasting GPU/API calls on invalid input.

#### 3. Logging/Observability
```python
def forward(self, image_path: str):
    import time
    start = time.time()
    logger.info(f"Processing {image_path}")

    from stardew_vision.tools import crop_pierres_detail_panel
    result = crop_pierres_detail_panel(image_path)

    duration = time.time() - start
    logger.info(f"Completed in {duration:.2f}s")
    return result
```

**Why**: Track tool performance, debug issues in production.

#### 4. Caching (Avoid Redundant Work)
```python
def __init__(self):
    super().__init__()
    self.cache = {}

def forward(self, image_path: str):
    if image_path in self.cache:
        logger.info(f"Cache hit: {image_path}")
        return self.cache[image_path]

    from stardew_vision.tools import crop_pierres_detail_panel
    result = crop_pierres_detail_panel(image_path)
    self.cache[image_path] = result
    return result
```

**Why**: OCR/vision operations are expensive—don't reprocess the same image.

### Rule of Thumb

If `forward()` is getting longer than ~20 lines, ask yourself: **"Should this logic be in my actual tool implementation instead?"**

Usually yes. Keep `forward()` focused on:
- ✅ Calling your implementation
- ✅ Handling errors gracefully
- ✅ Validating inputs
- ✅ Logging/monitoring

**Don't put in** `forward()`:
- ❌ Core business logic (put that in your actual implementation)
- ❌ Complex algorithms (separate into dedicated functions)

---

## CodeAgent Requirements

**Important**: CodeAgent requires a **code-capable VLM**.

### What CodeAgent Expects

CodeAgent asks the VLM to generate **Python code**:
```python
# VLM must generate simple code like:
image_path = "/path/to/screenshot.png"
result = crop_pierres_detail_panel(image_path=image_path)
print(result)
```

This is **not complex programming**—it's:
- Variable assignment
- Function calls with keyword arguments
- Print statements

The VLM doesn't write algorithms or classes, just **calls your tools using Python syntax**.

### Which Models Work?

**Good for CodeAgent** (code-trained VLMs):
- ✅ Qwen2.5-VL-7B-Instruct (our choice)
- ✅ GPT-4V / GPT-4o (OpenAI)
- ✅ Claude Sonnet 4 (Anthropic)
- ✅ Any model trained on code corpora

**Challenging for CodeAgent**:
- ❌ Small models (<3B params)
- ❌ Vision-only models without code training
- ❌ Models not fine-tuned for instruction following

### Why It Works Well

1. **Modern VLMs are trained on code**
   - Qwen2.5-VL has massive code corpora in training
   - Has seen millions of Python function call examples

2. **Patterns are predictable**
   - Tool calling follows consistent structure
   - VLM learns: "To use tool X with argument Y, write `X(Y=value)`"

3. **Self-correction via error messages**
   ```python
   # Step 1 - VLM generates buggy code:
   result = crop_pierres_detail_panel(image_path)  # NameError!

   # Step 2 - CodeAgent shows error to VLM:
   # NameError: name 'image_path' is not defined

   # Step 3 - VLM sees error, fixes it:
   image_path = "/path/to/screenshot.png"
   result = crop_pierres_detail_panel(image_path)
   ```

This self-correction loop makes CodeAgent **robust even when VLMs make mistakes**.

### Trade-off: CodeAgent vs ToolCallingAgent

| Aspect | ToolCallingAgent (JSON) | CodeAgent (Python) |
|--------|------------------------|-------------------|
| VLM requirement | Any function-calling model | Code-trained VLM |
| Model compatibility | ✅ Wider support | ⚠️  Needs code training |
| Composability | ❌ Limited | ✅ Full (chain tools in code) |
| Error messages | JSON parsing errors | Python stack traces |
| Self-correction | Limited | Automatic |

**For this course**: Qwen2.5-VL-7B-Instruct is explicitly trained on code, so CodeAgent works excellently.

---

## Code Walkthrough

Now let's walk through `examples/module2_smolagents_basic.py` line-by-line.

### Part 1: Import Smolagents (Lines 7-10)

```python
from smolagents import CodeAgent, Tool, InferenceClientModel
from PIL import Image
import json
```

**New imports**:
- `Tool`: Base class for creating tools
- `CodeAgent`: Agent that generates Python code
- `InferenceClientModel`: HuggingFace transformers backend

### Part 2: Create Tool Class (Lines 12-35)

```python
class CropPierresPanelTool(Tool):
    """
    Smolagents Tool wrapper for crop_pierres_detail_panel.

    Tool classes define:
    - name: Tool identifier (matches function name)
    - description: What the tool does (shown to VLM)
    - inputs: Parameter schema (type, description)
    - output_type: Return type (str, dict, Image, etc.)
    - forward(): Actual implementation
    """
    name = "crop_pierres_detail_panel"
    description = "Extract item details from Pierre's General Store detail panel in screenshot"

    inputs = {
        "image_path": {
            "type": "string",
            "description": "Path to the screenshot file"
        }
    }

    output_type = "object"

    def forward(self, image_path: str):
        """Execute the extraction tool."""
        from stardew_vision.tools import crop_pierres_detail_panel

        print(f"[CropPierresPanelTool] Executing on {image_path}")
        result = crop_pierres_detail_panel(image_path)
        print(f"[CropPierresPanelTool] Extraction successful: {result}")

        return result
```

**Line-by-line**:
- **Line 12**: Class inherits from `Tool` (required)
- **Line 22**: `name` - Must match what VLM will call
- **Line 23**: `description` - Shown to VLM, helps it decide when to use tool
- **Lines 25-30**: `inputs` - JSON schema for parameters (same format as Module 1!)
- **Line 32**: `output_type` - What this tool returns (`str`, `dict`, `Image`, etc.)
- **Line 34**: `forward()` - The actual implementation (replaces function from Module 1)
- **Line 37**: Import real tool (lazy import, only when called)
- **Lines 39-41**: Logging (same as Module 1, but in class method)

**Comparison to Module 1**:

Module 1 (Manual):
```python
# Definition (JSON)
TOOL_DEFINITIONS = [{"type": "function", "function": {"name": "...", ...}}]

# Implementation (function)
def crop_pierres_detail_panel(image_path: str) -> dict:
    ...

# Registry (dict)
TOOL_REGISTRY = {"crop_pierres_detail_panel": crop_pierres_detail_panel}
```

Module 2 (Smolagents):
```python
# Everything in one class!
class CropPierresPanelTool(Tool):
    name = "crop_pierres_detail_panel"
    description = "..."
    inputs = {...}
    output_type = "object"

    def forward(self, image_path: str):
        ...
```

**The win**: Definition and implementation are in sync automatically.

### Part 3: Initialize Model (Lines 45-51)

```python
print("Loading model (InferenceClientModel)...")
model = InferenceClientModel(
    model_id="Qwen/Qwen2.5-VL-7B-Instruct"
)
print("Model loaded!")
```

**What happens**:
- Smolagents connects to HuggingFace Inference API (or downloads model)
- First run may download model weights (~15 GB for Qwen2.5-VL-7B)
- Subsequent runs use cached model

**Model ID format**: `{org}/{repo}` from HuggingFace Hub

**Note**: This is **not** using vLLM yet. That's Module 3. This is for quick testing.

### Part 4: Create Tools List (Line 53)

```python
tools = [CropPierresPanelTool()]
```

**Why a list?**
- Agent can have multiple tools
- VLM sees all tools, picks the right one
- Example: `[CropPierresPanelTool(), GetScreenTypeTool(), ExtractTVDialogTool()]`

**Instance vs class**:
```python
# WRONG
tools = [CropPierresPanelTool]  # Class, not instance

# RIGHT
tools = [CropPierresPanelTool()]  # Instance (note the parentheses)
```

### Part 5: Initialize CodeAgent (Lines 55-61)

```python
print("Initializing CodeAgent...")
agent = CodeAgent(
    tools=tools,
    model=model,
    add_base_tools=False,  # Don't add default DuckDuckGo, etc.
    max_steps=3,           # Limit iterations
    verbosity=2            # Show reasoning
)
print("Agent initialized!")
```

**Parameters explained**:
- `tools`: List of Tool instances
- `model`: Model backend (InferenceClientModel, OpenAIServerModel, etc.)
- `add_base_tools`: If `True`, adds web search, Python REPL, etc. (we don't need these)
- `max_steps`: Max reasoning iterations (prevents infinite loops)
- `verbosity`: 0=silent, 1=minimal, 2=detailed (shows VLM reasoning)

**What CodeAgent does during init**:
1. Converts Tool classes to JSON tool definitions
2. Prepares system prompt with available tools
3. Sets up code execution sandbox
4. Connects to model backend

### Part 6: Run Agent (Lines 65-71)

```python
image_path = "/workspaces/stardew-vision/tests/fixtures/pierre_shop_001.png"

print(f"Running agent on: {image_path}")
print("-" * 80)

result = agent.run(
    f"Extract all item information from the Pierre's General Store screenshot at {image_path}. "
    f"Use the crop_pierres_detail_panel tool."
)
```

**The `agent.run()` method**:
- Takes a natural language prompt
- Returns extracted data

**What happens inside** (see [`diagrams/codeagent-execution.txt`](../diagrams/codeagent-execution.txt)):
1. Agent sends prompt + tool definitions to VLM
2. VLM generates Python code: `result = crop_pierres_detail_panel(image_path="/path/...")`
3. Agent executes code in sandbox
4. Code calls `CropPierresPanelTool.forward()`
5. Tool extracts data, returns dict
6. Agent returns final result

**With `verbosity=2`, you'll see**:
```
[Agent] Step 1/3
[VLM] Generating code...
[Agent] Executing:
  image_path = "/workspaces/stardew-vision/tests/fixtures/pierre_shop_001.png"
  result = crop_pierres_detail_panel(image_path=image_path)
  print(result)
[CropPierresPanelTool] Executing on /workspaces/stardew-vision/tests/fixtures/pierre_shop_001.png
[CropPierresPanelTool] Extraction successful: {"name": "Parsnip", ...}
[Agent] Step 1 complete
```

### Part 7: Display Result (Lines 73-82)

```python
print("-" * 80)
print()
print("Agent Result:")
print(json.dumps(result, indent=2))
print()

print("✅ Module 2 Complete!")
print("You now understand:")
print("  - How to create Smolagents Tool classes")
print("  - How CodeAgent writes Python code to call tools")
print("  - How InferenceClientModel works with HF models")
```

---

## Hands-On Activity

### Checkpoint 1: Run the Example

```bash
# From project root
cd /workspaces/stardew-vision

# Run Module 2 example
python agent-learning/examples/module2_smolagents_basic.py
```

**First run**: May download Qwen2.5-VL-7B (~15 GB), takes 5-10 minutes

**Expected output**:
```
================================================================================
MODULE 2: SMOLAGENTS BASICS
================================================================================

Loading model (InferenceClientModel)...
Model loaded!

Initializing CodeAgent...
Agent initialized!

Running agent on: /workspaces/stardew-vision/tests/fixtures/pierre_shop_001.png
--------------------------------------------------------------------------------
[Agent] Step 1/3
[VLM] Generating code...
[Agent] Executing:
  image_path = "/workspaces/stardew-vision/tests/fixtures/pierre_shop_001.png"
  result = crop_pierres_detail_panel(image_path=image_path)
  print(result)
[CropPierresPanelTool] Executing on /workspaces/stardew-vision/tests/fixtures/pierre_shop_001.png
[CropPierresPanelTool] Extraction successful: {...}
--------------------------------------------------------------------------------

Agent Result:
{
  "name": "Parsnip",
  "description": "A spring vegetable. Still loved by many.",
  "price_per_unit": 20,
  "quantity_selected": 5,
  "total_cost": 100
}

✅ Module 2 Complete!
```

**If it fails**:
- Check internet connection (needs HF Hub access)
- Verify Smolagents installed: `python -c "import smolagents"`
- Check PYTHONPATH includes `/workspaces/stardew-vision/src`

### Checkpoint 2: Experiment with Parameters

Modify the example and see what changes:

**Test 1: Silent mode**
```python
agent = CodeAgent(..., verbosity=0)  # No reasoning output
```

**Test 2: Single-step**
```python
agent = CodeAgent(..., max_steps=1)  # Only one reasoning iteration
```

**Test 3: Add base tools**
```python
agent = CodeAgent(..., add_base_tools=True)  # Web search, Python REPL, etc.
```

What tools are available now?
```python
print([tool.name for tool in agent.tools])
```

**Test 4: Different prompt**
```python
result = agent.run("What's in this screenshot? Extract details.")
# Vaguer prompt - does VLM still pick the right tool?
```

### Checkpoint 3: Create a Second Tool

Add a tool that identifies screen type:

```python
class GetScreenTypeTool(Tool):
    name = "get_screen_type"
    description = "Identify which UI panel is shown in a Stardew Valley screenshot"

    inputs = {
        "image_path": {
            "type": "string",
            "description": "Path to the screenshot file"
        }
    }

    output_type = "object"

    def forward(self, image_path: str):
        """Mock implementation - always returns pierre_shop for now."""
        # In production, this would use a classifier
        return {"screen_type": "pierre_shop"}

# Add to tools list
tools = [CropPierresPanelTool(), GetScreenTypeTool()]

# Run agent with both tools
agent = CodeAgent(tools=tools, model=model, verbosity=2)
result = agent.run("First identify the screen type, then extract details from the screenshot")
```

**What to observe**:
- Does VLM call both tools?
- In what order?
- How does it use the screen_type result?

**Full exercise**: [`exercises/exercise2_agent_config.md`](../exercises/exercise2_agent_config.md)

---

## Key Patterns

### Pattern 1: Tool Class Structure

**Always include**:
```python
class MyTool(Tool):
    name = "unique_name"          # Required
    description = "Clear desc"    # Required (helps VLM)
    inputs = {...}                # Required (JSON schema)
    output_type = "object"          # Required

    def forward(self, **kwargs):  # Required
        # Implementation here
        return result
```

### Pattern 2: Lazy Imports in forward()

**Don't import at top of file**:
```python
# WRONG
from heavy_library import expensive_function

class MyTool(Tool):
    def forward(self):
        return expensive_function()
```

**Do import inside forward()**:
```python
# RIGHT
class MyTool(Tool):
    def forward(self):
        from heavy_library import expensive_function
        return expensive_function()
```

**Why?**
- Tool definitions are sent to VLM (metadata only)
- Heavy imports slow down tool registration
- Import only when tool is actually called

### Pattern 3: Tool Instantiation

```python
# Create instances, not classes
tools = [Tool1(), Tool2(), Tool3()]  # ✅

# NOT
tools = [Tool1, Tool2, Tool3]  # ❌ Missing parentheses
```

---

## Common Pitfalls

### Pitfall 1: Forgetting to Instantiate Tools

**Error**:
```python
tools = [CropPierresPanelTool]  # Class, not instance
agent = CodeAgent(tools=tools, model=model)
# TypeError: 'type' object is not iterable
```

**Fix**:
```python
tools = [CropPierresPanelTool()]  # Add parentheses!
```

### Pitfall 2: Missing Model Backend

**Error**:
```python
from smolagents import CodeAgent, Tool
agent = CodeAgent(tools=[MyTool()])  # No model!
# TypeError: missing required argument: 'model'
```

**Fix**:
```python
from smolagents import CodeAgent, Tool, InferenceClientModel
model = InferenceClientModel(model_id="Qwen/Qwen2.5-VL-7B-Instruct")
agent = CodeAgent(tools=[MyTool()], model=model)
```

### Pitfall 3: Vague Tool Descriptions

VLM uses descriptions to decide which tool to call:

**Bad**:
```python
description = "Does stuff with images"  # Too vague!
```

**Good**:
```python
description = "Extract item details from Pierre's General Store detail panel in a Stardew Valley screenshot"
```

### Pitfall 4: Invalid output_type

**Wrong**:
```python
output_type = "dict"  # NOT a valid Smolagents type!
```

**Valid `output_type` values** (from Smolagents v1.24.0):
- `"string"` - For text/string data
- `"integer"` - For whole numbers
- `"number"` - For floats
- `"boolean"` - For true/false
- `"object"` - For dictionaries/JSON objects ← **Use this for dict returns!**
- `"array"` - For lists
- `"image"` - For PIL.Image
- `"audio"` - For audio files
- `"any"` - For any type
- `"null"` - For None

**Fix**:
```python
output_type = "object"  # Correct for dictionary returns
```

### Pitfall 5: Incorrect Input Schema

**Wrong types**:
```python
inputs = {
    "image_path": {"type": "file"}  # Not a valid JSON schema type!
}
```

**Valid JSON schema types**: `string`, `integer`, `number`, `boolean`, `object`, `array`, `null`

**Fix**:
```python
inputs = {
    "image_path": {"type": "string", "description": "Path to image file"}
}
```

---

## Key Takeaways

### 1. Tool Classes Unify Definition + Implementation

Module 1: Definition (JSON) + Implementation (function) + Registry (dict)
Module 2: One Tool class contains everything

**Bonus**: Metadata auto-generates the system prompt!

### 2. Smolagents Handles the Full Orchestration Loop

Module 1: You only wrote dispatch (~10 lines)
Module 2: Smolagents handles prompt construction, VLM calls, response parsing, dispatch, multi-step loops (~60-110 lines → 2 lines)

### 3. System Prompts are Auto-Generated

Tool metadata → Function signatures → System prompt → VLM sees documentation

**Your job**: Write clear `name`, `description`, `inputs` metadata
**Smolagents' job**: Convert to prompt format

### 4. Messages are Separate (System vs User)

Not concatenated! Chat message array:
```python
[
    {"role": "system", "content": "Tool definitions..."},
    {"role": "user", "content": "Your task..."}
]
```

### 5. CodeAgent Requires Code-Capable VLMs

Works great with: Qwen2.5-VL, GPT-4o, Claude Sonnet
Struggles with: Small models (<3B), non-code-trained models

But the Python code it generates is **simple** (just function calls).

### 6. forward() Methods Should Be Thin Wrappers

Keep it simple: just call your implementation
Add logic for: error handling, validation, logging, caching

Don't put core business logic in `forward()`—keep it in your actual implementation.

---

## What's Next?

**Current state**:
- ✅ Tool classes (cleaner than manual definitions)
- ✅ CodeAgent (automated dispatch)
- ✅ InferenceClientModel (HuggingFace backend)

**Limitations**:
- ❌ No local vLLM (slower, requires internet)
- ❌ No production-grade error handling
- ❌ No observability (logging, metrics)

**Module 3** fixes these:
- Use vLLM for faster local inference
- Use OpenAIServerModel for OpenAI-compatible endpoints
- Same agent code, just swap model backend

**Module 4** adds production features:
- VLMOrchestrator wrapper class
- Error handling and validation
- MLFlow logging
- Unit tests with mocking

---

## Summary

You now understand:

1. ✅ **Why Smolagents** - Automates tedious dispatch logic
2. ✅ **Tool classes** - Single source of truth for definition + implementation
3. ✅ **CodeAgent** - Generates Python code (not JSON)
4. ✅ **Model backends** - InferenceClientModel for quick testing
5. ✅ **Agent.run()** - Natural language → tool execution → result

**The abstraction ladder**:
- Module 1: Raw JSON + manual dispatch
- Module 2: Tool classes + CodeAgent
- Module 3: Add vLLM for production speed
- Module 4: Add error handling + observability

---

## Additional Resources

**Code**:
- Full example: [`examples/module2_smolagents_basic.py`](../examples/module2_smolagents_basic.py)
- Exercise: [`exercises/exercise2_agent_config.md`](../exercises/exercise2_agent_config.md)

**Diagrams**:
- [`diagrams/codeagent-execution.txt`](../diagrams/codeagent-execution.txt) - How CodeAgent works
- [`diagrams/tool-calling-flow.txt`](../diagrams/tool-calling-flow.txt) - General flow

**Official Docs**:
- [Smolagents Guided Tour](https://huggingface.co/docs/smolagents/guided_tour)
- [Tool Conceptual Guide](https://huggingface.co/docs/smolagents/main/en/conceptual_guides/tools)
- [Building Good Agents](https://huggingface.co/docs/smolagents/tutorials/building_good_agents)

**Related**:
- [`docs/smolagents-quickstart.md`](../docs/smolagents-quickstart.md) - Quick reference
- [`docs/framework-decision.md`](../docs/framework-decision.md) - Why Smolagents over alternatives

---

**Ready for Module 3?** → [`module3-vllm-integration.md`](module3-vllm-integration.md)

Learn how to connect Smolagents to vLLM for production-grade serving!
