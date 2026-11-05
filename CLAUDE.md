# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

**Setup:**
```bash
uv venv && source .venv/bin/activate
cp .env.example .env  # Add ANTHROPIC_API_KEY and E2B_API_KEY
```

**Run:**
```bash
uv run app.py    # Web interface at localhost:5000
uv run ce3.py    # CLI interface
```

**Lint & Format:**
```bash
ruff check .          # Lint
ruff check . --fix    # Auto-fix lint issues
black .               # Format
mypy .                # Type check
```

**Tests:**
```bash
pytest                        # Run all tests
pytest tests/test_file.py     # Run a single test file
pytest -k "test_name"         # Run a specific test
```

## Architecture

### Entry Points
- `ce3.py` — CLI interface; contains the `Assistant` class that drives the core loop
- `app.py` — Flask web server; wraps the same `Assistant` backend for a browser UI

### Core Flow
1. User input arrives via CLI (`ce3.py`) or HTTP POST `/chat` (`app.py`)
2. `Assistant.chat()` appends to conversation history and calls `_get_completion()`
3. `_get_completion()` calls the Anthropic API with dynamically loaded tools
4. If `stop_reason == "tool_use"`, `_execute_tool()` runs the requested tool and the result is appended to history; `_get_completion()` is called recursively (tool chaining)
5. Final text response is returned to the user; token usage is accumulated and displayed

### Tool System (`tools/`)
Tools inherit from `BaseTool` (abstract) and are discovered at runtime via `pkgutil`. The `Assistant._load_tools()` method imports all submodules in `tools/` and registers any `BaseTool` subclass it finds.

Key built-in tools:
- `toolcreator` — asks Claude to write new tool code, enabling the self-improving design
- `e2bcodetool` — executes Python in an E2B sandbox
- `lintingtool` — runs Ruff on generated code
- `uvpackagemanager` — installs Python packages at runtime
- File tools: `filecreatortool`, `fileedittool`, `filecontentreadertool`, `createfolderstool`, `diffeditortool`
- Web tools: `duckduckgotool`, `webscrapertool`, `browsertool`, `screenshottool`

### Configuration (`config.py`)
Central constants: `MODEL`, `MAX_TOKENS` (8000), `MAX_CONVERSATION_TOKENS` (200000), `ENABLE_THINKING`, `SHOW_TOOL_USAGE`.

### System Prompts (`prompts/system_prompts.py`)
`DEFAULT` describes the assistant's capabilities; `TOOL_USAGE` provides tool-use guidelines. Both are injected as the system message on every API call.

### Web Interface (`app.py` + `templates/` + `static/`)
Flask routes: `/` (home), `/chat` (POST), `/upload` (image upload), `/reset` (reset session state). The frontend renders Markdown with syntax highlighting and displays real-time token usage. Image uploads are passed to Claude Vision.

### Special CLI Commands
Inside `ce3.py`, the `chat()` method intercepts: `refresh` (reload tools), `reset` (clear history), `quit`/`exit`.
