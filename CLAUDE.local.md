# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Open WebUI is a self-hosted AI platform that provides a web interface for interacting with LLMs. It supports Ollama, OpenAI-compatible APIs, and includes built-in RAG capabilities with multiple vector database backends.

## Development Commands

### Frontend (SvelteKit + Svelte 5)

```bash
npm install                    # Install dependencies
npm run dev                    # Start dev server on :5173
npm run dev:5050              # Start dev server on :5050
npm run build                  # Build for production
npm run check                  # Type check with svelte-check
npm run lint:frontend          # ESLint
npm run format                 # Prettier formatting
npm run test:frontend          # Run Vitest tests
npm run cy:open               # Open Cypress for e2e tests
npm run i18n:parse            # Extract i18n strings
```

### Backend (FastAPI + Python 3.11/3.12)

```bash
cd backend
./dev.sh                       # Start backend with hot reload on :8080

# Or manually:
uvicorn open_webui.main:app --port 8080 --host 0.0.0.0 --reload
```

### Backend Testing

```bash
# Tests use pytest with Docker for Postgres integration tests
cd backend
pytest open_webui/test/        # Run all tests
pytest open_webui/test/apps/webui/routers/test_auths.py  # Single test file
```

### Linting/Formatting

```bash
npm run lint                   # All linting (frontend + types + backend)
npm run format:backend         # Format Python with black
pylint backend/                # Python linting
```

### Docker

```bash
make install                   # docker-compose up -d
make start                     # Start containers
make stop                      # Stop containers
make startAndBuild            # Rebuild and start
```

## Architecture

### Backend Structure (`backend/open_webui/`)

- **main.py** - FastAPI application entry point, mounts all routers
- **env.py** - Environment variable loading and configuration constants
- **config.py** - Runtime configuration (large file, ~120KB)
- **routers/** - API endpoints organized by domain:
  - `auths.py` - Authentication, signup, OAuth
  - `chats.py` - Chat CRUD operations
  - `ollama.py` - Ollama API proxy
  - `openai.py` - OpenAI-compatible API proxy
  - `retrieval.py` - RAG/document processing (~110KB, largest router)
  - `channels.py` - Real-time messaging channels
  - `knowledge.py` - Knowledge base management
  - `files.py` - File uploads and management
  - `functions.py`, `tools.py` - Custom Python functions/tools
- **models/** - SQLAlchemy/Pydantic models (one file per entity)
- **retrieval/** - RAG implementation:
  - `vector/dbs/` - Vector DB adapters (Chroma, PGVector, Qdrant, Milvus, etc.)
  - `loaders/` - Document loaders (YouTube, external docs, etc.)
- **socket/** - Socket.IO for real-time features
- **utils/** - Shared utilities (auth, audit, embeddings, etc.)
- **storage/provider.py** - Storage backends (local, S3, GCS, Azure)
- **migrations/** - Alembic database migrations

### Frontend Structure (`src/`)

- **routes/** - SvelteKit pages:
  - `(app)/` - Main authenticated app routes
  - `(app)/c/[id]` - Chat view
  - `(app)/admin/` - Admin panel
  - `(app)/workspace/` - Functions, knowledge, tools management
  - `auth/` - Login/signup pages
- **lib/components/** - Svelte components organized by feature:
  - `chat/` - Chat UI components
  - `admin/` - Admin panel components
  - `common/` - Shared UI components
  - `workspace/` - Workspace management
- **lib/apis/** - API client modules (mirror backend routers)
- **lib/stores/index.ts** - Svelte stores for global state
- **lib/i18n/locales/** - Translation files

### Key Patterns

- Backend uses Pydantic for request/response validation
- Frontend state managed via Svelte writable stores
- Real-time updates via Socket.IO (socket/main.py + frontend socket store)
- Database: SQLAlchemy with Alembic migrations, supports SQLite and PostgreSQL
- Authentication: JWT tokens, OAuth support, optional LDAP/SCIM

### Database Migrations

```bash
# Alembic migrations in backend/open_webui/migrations/
cd backend
alembic upgrade head          # Apply all migrations
alembic revision --autogenerate -m "description"  # Create new migration
```

## Environment Configuration

Copy `.env.example` to `.env`. Key variables:
- `OLLAMA_BASE_URL` - Ollama server URL
- `OPENAI_API_KEY` / `OPENAI_API_BASE_URL` - OpenAI configuration
- `DATABASE_URL` - PostgreSQL connection (optional, defaults to SQLite)

## Translations

Add new language in `src/lib/i18n/locales/`:
1. Create directory with language code (e.g., `es-ES`)
2. Copy files from `en-US` and translate
3. Add entry to `languages.json`

Keep translation PRs separate from feature PRs.

---

## Custom Deployment: Proxmox LXC Setup

This instance runs in a Proxmox LXC container installed from the official Open WebUI template.

### Installation Path

```
/root/.local/share/uv/tools/open-webui/lib/python3.12/site-packages/open_webui/
```

The installation uses `uv` (Python package manager). The entry point script is at:
```
/root/.local/bin/open-webui
```

### Service Management

```bash
systemctl status open-webui    # Check status
systemctl restart open-webui   # Restart after code changes
journalctl -u open-webui -f    # Follow logs
```

### Patching Production Code

To patch the installed package directly:

```bash
# Find the installation path
OWUI_PATH=$(/root/.local/share/uv/tools/open-webui/bin/python -c "import open_webui; print(open_webui.__path__[0])")

# Backup before editing
cp "$OWUI_PATH/utils/misc.py" "$OWUI_PATH/utils/misc.py.bak"

# Edit and restart
nano "$OWUI_PATH/utils/misc.py"
systemctl restart open-webui
```

**Note:** Files may have hardlinks (uv optimization). When nano warns "file has hardlinks, detach?", say yes.

---

## Bug Investigation: System Prompt Duplication (Native Function Calling)

### Summary

**Bug:** System prompt content gets duplicated during native function calling with MCP tools, causing quadratic token growth.

**Impact:** A 20k token conversation can balloon to 3M+ tokens, causing massive API costs.

**Status:** Fix developed and tested. PR submitted: https://github.com/open-webui/open-webui/pull/20480

### Root Cause

When using native function calling mode with MCP tools:

1. Initial request applies system prompt via `apply_system_prompt_to_body()` with `replace=True`
2. Each tool call iteration calls `generate_chat_completion()` again
3. The router applies model system prompt via `apply_system_prompt_to_body()` with `replace=False`
4. This calls `update_message_content()` with `append=False`, which **prepends** content
5. Same system prompt gets prepended on every iteration:
   - Iteration 1: `"System prompt"`
   - Iteration 2: `"System prompt\nSystem prompt"`
   - Iteration 3: `"System prompt\nSystem prompt\nSystem prompt"`

### Key Code Paths

| Location | File | Line | Description |
|----------|------|------|-------------|
| Chat system prompt | `utils/middleware.py` | ~1170 | Applied with `replace=True` |
| Folder system prompt | `utils/middleware.py` | ~1229 | Applied with `replace=False` |
| Agentic loop | `utils/middleware.py` | ~3050-3074 | Tool call iteration loop |
| Model system prompt | `routers/openai.py` | ~829 | Applied with `replace=False` on EVERY call |
| Model system prompt | `routers/ollama.py` | ~1294, ~1483 | Same issue |
| Content update | `utils/misc.py` | ~167 | `update_message_content()` - prepends without duplicate check |

### The Fix

Add duplicate detection to `update_message_content()` in `utils/misc.py`:

```python
def update_message_content(message: dict, content: str, append: bool = True) -> dict:
    # Get existing text content for duplicate check
    existing_text = ""
    if isinstance(message.get("content"), list):
        for item in message["content"]:
            if item.get("type") == "text":
                existing_text = item.get("text", "")
                break
    else:
        existing_text = message.get("content", "")

    # Skip update if content already present (fixes system prompt duplication
    # during native function calling tool iterations - GitHub #19169)
    if existing_text and content:
        existing_stripped = existing_text.strip()
        content_stripped = content.strip()
        if existing_stripped.startswith(content_stripped):
            return message

    # ... rest of function unchanged
```

### Debugging Tools

Debug patch to trace system message operations:

```python
# Add to utils/misc.py add_or_update_system_message function:
import traceback
sys_count = sum(1 for m in messages if m.get("role") == "system")
msg0_role = messages[0].get("role") if messages else "EMPTY"
print(f"DEBUG_SYSPROMPT sys_msgs={sys_count}, msg[0].role={msg0_role}, append={append}")
```

Check logs: `journalctl -u open-webui -f | grep DEBUG_SYSPROMPT`

### Related Issues

- #19656 - Tool response token duplication (partial fix for non-native mode)
- #19169 - System Prompt Duplication During Agentic Tool Calls (closed)

---

## System Prompt Flow (Native Function Calling)

```
1. POST /chat/completions
   ↓
2. process_chat_payload() [middleware.py:1159]
   - apply_system_prompt_to_body(chat_prompt, replace=True)
   - apply_system_prompt_to_body(folder_prompt, replace=False)
   ↓
3. Native tools added [middleware.py:1550]
   - form_data["tools"] = [tool_specs]
   ↓
4. generate_chat_completion() → routers/openai.py
   - apply_system_prompt_to_body(model_prompt, replace=False) ← PREPENDS
   ↓
5. LLM returns tool call
   ↓
6. Agentic loop [middleware.py:2882-3074]
   - Process tool result
   - Create new_form_data with existing messages
   - Call generate_chat_completion() AGAIN
   ↓
7. routers/openai.py AGAIN
   - apply_system_prompt_to_body(model_prompt) ← PREPENDS AGAIN = DUPLICATION
   ↓
8. Loop continues, duplication compounds
```

---

## Message Utility Functions Reference

**File:** `backend/open_webui/utils/misc.py`

| Function | Purpose |
|----------|---------|
| `get_system_message(messages)` | Find first system message in list |
| `remove_system_message(messages)` | Filter out all system messages |
| `add_or_update_system_message(content, messages, append)` | Add or update system message at position 0 |
| `update_message_content(message, content, append)` | Append or prepend content to a message |
| `replace_system_message_content(content, messages)` | Replace system message content entirely |

**File:** `backend/open_webui/utils/payload.py`

| Function | Purpose |
|----------|---------|
| `apply_system_prompt_to_body(system, form_data, metadata, user, replace)` | Apply system prompt - calls misc.py functions |

**Behavior of `append` parameter:**
- `append=True`: Content added to END of existing content
- `append=False`: Content added to START (prepended) of existing content

---

## Backend File Reference (Explored)

### Core Request Flow

**`backend/open_webui/main.py`**
- FastAPI application entry point
- Mounts all routers
- Sets up middleware pipeline
- Line ~1623-1630: Sets `function_calling` mode in metadata

**`backend/open_webui/utils/middleware.py`** (~3350 lines - LARGE)
- Central request/response processing
- `process_chat_payload()` (line ~1159): Prepares chat request, applies system prompts
- `process_chat_response()` (line ~1640): Handles streaming response
- `stream_body_handler()` (line ~2497): Processes streamed chunks
- `response_handler()` (line ~2081): Main response handler
- Agentic loop (lines ~2882-3074): Tool call iteration
- Native vs non-native branching (lines ~1547-1562)

**`backend/open_webui/utils/chat.py`**
- `generate_chat_completion()` (line ~164-286): Routes to appropriate backend
- Imported by middleware at line 75

### Routers

**`backend/open_webui/routers/openai.py`** (~1100 lines)
- OpenAI-compatible API proxy
- `generate_chat_completion()` (line ~795-1100): Main completion handler
- Line ~823-829: Extracts and applies model system prompt from `model_info.params`
- **BUG LOCATION**: Line ~829 calls `apply_system_prompt_to_body()` without `replace=True`

**`backend/open_webui/routers/ollama.py`** (~1800 lines)
- Ollama API proxy
- `generate_chat_completion()` (line ~1250): Ollama completion handler
- `generate_openai_chat_completion()` (line ~1447): OpenAI-compatible endpoint
- **BUG LOCATIONS**: Lines ~1294 and ~1483 - same issue as openai.py

### Utility Files

**`backend/open_webui/utils/misc.py`** (~650 lines)
- Message manipulation utilities
- `get_system_message()` (line ~152-156): Find system message anywhere in list
- `remove_system_message()` (line ~159-161): Filter out system messages
- `update_message_content()` (line ~167-180): Append/prepend to message content
- `replace_system_message_content()` (line ~183-188): Replace system message entirely
- `add_or_update_system_message()` (line ~191-209): Add or update at position 0
- **FIX LOCATION**: `update_message_content()` - added duplicate detection

**`backend/open_webui/utils/payload.py`** (~60 lines)
- `apply_system_prompt_to_body()` (line ~13-41): Main system prompt application
- Calls either `replace_system_message_content()` or `add_or_update_system_message()`
- `replace` parameter determines which function is called

**`backend/open_webui/functions.py`**
- Custom function/pipe execution
- `generate_function_chat_completion()` (line ~158-352)
- Line ~289: Also applies system prompt (same pattern)

### MCP Integration

**`backend/open_webui/utils/mcp/client.py`**
- MCP (Model Context Protocol) client wrapper
- `connect()` (line ~18-38): Establishes connection to MCP server
- `list_tool_specs()` (line ~40-61): Lists available tools
- `call_tool()` (line ~63-79): Executes tool on MCP server

MCP tool registration happens in middleware.py (lines ~1394-1530):
- Detects MCP tool servers from configuration
- Creates MCPClient instances
- Registers tools into `mcp_tools_dict`
- Merges with local tools into `tools_dict`

---

## Complete Message Flow Architecture

### 1. Request Entry

```
Client POST /api/chat/completions
    ↓
main.py: chat_completion endpoint
    ↓
middleware.py: process_chat_payload()
```

### 2. Payload Processing (`process_chat_payload`)

```
Line 1167: Extract existing system message
Line 1170: apply_system_prompt_to_body(chat_prompt, replace=True)
Line 1229: apply_system_prompt_to_body(folder_prompt, replace=False)
Line 1317: add_or_update_system_message(voice_template) [if voice mode]
Line 548:  add_or_update_system_message(memory_context, append=True) [if memory enabled]
    ↓
Line 1547-1562: Check function_calling mode
    ├── Native: Add tools to form_data["tools"], continue to LLM
    └── Default: Call chat_completion_tools_handler() [different path]
```

### 3. LLM Request (Native Mode)

```
utils/chat.py: generate_chat_completion()
    ↓
Routing based on model type:
    ├── routers/openai.py: generate_chat_completion()
    │   └── Line 829: apply_system_prompt_to_body(model_prompt) ← NO replace=True
    │
    └── routers/ollama.py: generate_chat_completion() or generate_openai_chat_completion()
        └── Lines 1294, 1483: apply_system_prompt_to_body(model_prompt) ← NO replace=True
```

### 4. Response Processing

```
middleware.py: process_chat_response()
    ↓
stream_body_handler() [for streaming]
    ↓
Detect tool_calls in response
    ↓
If tool_calls present → Enter agentic loop
```

### 5. Agentic Tool Loop (Lines 2882-3074)

```
while (tool_calls > 0 and retries < MAX):
    ↓
    Execute tool calls (lines 2911-3030)
        - For MCP tools: await tool["callable"]() → mcp/client.py call_tool()
        - For local tools: execute function
    ↓
    Reconstruct messages (lines 3050-3060):
        new_form_data = {
            **form_data,
            "messages": [
                *form_data["messages"],      ← Original messages
                *convert_content_blocks_to_messages(content_blocks)  ← Tool results
            ]
        }
    ↓
    Call generate_chat_completion() AGAIN (line 3062)
        → Routes back to openai.py/ollama.py
        → apply_system_prompt_to_body() called AGAIN
        → System prompt PREPENDED again = DUPLICATION
    ↓
    Check for more tool_calls → loop continues
```

### 6. Key Functions in misc.py

```python
add_or_update_system_message(content, messages, append=False):
    if messages[0].role == "system":
        update_message_content(messages[0], content, append)  # Update existing
    else:
        messages.insert(0, {role: "system", content})  # Insert new

update_message_content(message, content, append=True):
    if append:
        message.content = f"{existing}\n{content}"  # Add to end
    else:
        message.content = f"{content}\n{existing}"  # Add to start (PREPEND)
```

---

## Investigation Timeline

1. **Initial hypothesis**: `add_or_update_system_message()` checking only `messages[0]`
2. **First fix attempt**: Search for system message anywhere in list - didn't work
3. **Debug patch**: Added logging to trace actual behavior
4. **Discovery**: System message IS found at position 0, but content is being PREPENDED
5. **Root cause**: `update_message_content()` with `append=False` prepends without checking for duplicates
6. **Final fix**: Add duplicate detection before prepending

### Debug Output That Revealed the Bug

```json
{"action": "INSERT", "sys_before": 0, "sys_after": 1, "msg0_role": "user"}
{"action": "UPDATE", "sys_before": 1, "sys_after": 1, "msg0_role": "system"}
{"action": "UPDATE", "sys_before": 1, "sys_after": 1, "msg0_role": "system"}
```

Each "UPDATE" was prepending the same content, causing token growth within a single system message.
