# Claude Compact Summary #1

**Date:** 2026-01-08
**Tags:** open-webui, bug-fix, system-prompt-duplication, native-function-calling, mcp-tools
**Branch:** `docs/bug-investigation`

---

## 1. Primary Request and Intent

User reported a critical bug in Open WebUI where **system prompts were being duplicated during native function calling with MCP tools**, causing quadratic token growth and massive API cost overruns. A 20k token conversation could balloon to 3M+ tokens.

**Goals:**
- Investigate and find the root cause of the bug
- Develop and test a fix
- Submit a PR to upstream
- Document everything for future reference

---

## 2. Work Completed

### Investigation
- Launched 7+ parallel investigators to trace message flow
- Identified all system prompt injection points in the codebase
- Created debug patches to trace actual runtime behavior
- Confirmed bug via debug logging on user's LXC production instance

### Root Cause Found
- Bug is in `update_message_content()` in `utils/misc.py`
- When `append=False`, content is PREPENDED without checking for duplicates
- Each tool call iteration prepends the same system prompt again
- Result: `"Prompt\nPrompt\nPrompt..."` growing with each iteration

### Fix Developed & Tested
- Added duplicate detection to `update_message_content()`
- Checks if content already exists at start of message before prepending
- Tested on user's LXC - token count stayed stable at ~12k after 3 tool calls (was growing to 36k+)

### PR Submitted
- PR #20480: https://github.com/open-webui/open-webui/pull/20480
- Targets `dev` branch (not `main` - project convention)
- Includes CLA agreement (required by project)
- Clean single-commit history

### Documentation
- Created comprehensive `CLAUDE.local.md` with full investigation notes
- Saved to separate branch `docs/bug-investigation` in user's fork
- PR branch kept clean with only the fix

---

## 3. Work Remaining

- **PR Review:** Waiting for maintainers to review PR #20480
- **PR may need updates:** Bot flagged need for explicit testing confirmation
- **If PR rejected:** User can re-apply patch manually using documented instructions

---

## 4. Critical Context Information

### LXC Deployment Setup
- Open WebUI installed via Proxmox LXC template
- Installation path: `/root/.local/share/uv/tools/open-webui/lib/python3.12/site-packages/open_webui/`
- Uses `uv` package manager (creates hardlinks - must detach when editing)
- Service: `systemctl restart open-webui`
- Logs: `journalctl -u open-webui -f`

### Key Discovery Process
1. Initial hypothesis (wrong): `add_or_update_system_message()` checking only `messages[0]`
2. First fix attempt (failed): Search for system message anywhere in list
3. Debug patch revealed: System message WAS at position 0, but content was being PREPENDED
4. Actual bug: `update_message_content()` prepends without duplicate check

### GitHub Repository Setup
- Upstream: `open-webui/open-webui`
- User's fork: `jvadura/open-webui`
- PR branch: `fix/system-prompt-duplication`
- Docs branch: `docs/bug-investigation`

### Project Conventions
- PRs target `dev` branch, not `main`
- CLA required in PR description
- Bot auto-closes PRs without CLA

---

## 5. Key Technical Concepts

- **Native Function Calling:** Tools passed directly to LLM via `form_data["tools"]`
- **Agentic Loop:** Iterative tool call processing in `middleware.py:2882-3074`
- **MCP (Model Context Protocol):** Tool server integration
- **System Prompt Flow:** Multiple injection points with different `replace` settings
- **Prepend vs Append:** `append=False` means content added to START of existing

---

## 6. Files and Code Sections

### Modified Files

**`backend/open_webui/utils/misc.py`** - THE FIX
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

    # Skip update if content already present
    if existing_text and content:
        existing_stripped = existing_text.strip()
        content_stripped = content.strip()
        if existing_stripped.startswith(content_stripped):
            return message

    # ... rest unchanged
```

### Key Files Explored

| File | Purpose | Key Lines |
|------|---------|-----------|
| `utils/middleware.py` | Central request processing | ~1170, ~1229, ~2882-3074 |
| `utils/misc.py` | Message utilities | ~167-180, ~191-209 |
| `utils/payload.py` | System prompt application | ~13-41 |
| `routers/openai.py` | OpenAI API proxy | ~829 |
| `routers/ollama.py` | Ollama API proxy | ~1294, ~1483 |
| `utils/mcp/client.py` | MCP tool client | ~63-79 |

---

## 7. Errors and Problem Solving

### Debug Patch Syntax Error
- **Issue:** Regex-based patch inserted code inside `try:` block before `except:`
- **Solution:** Created simpler patch that replaces entire function

### First Fix Didn't Work
- **Issue:** Changed `add_or_update_system_message()` to search anywhere
- **Discovery:** System message WAS at position 0, problem was content prepending
- **Solution:** Fix `update_message_content()` instead

### CLAUDE.md in .gitignore
- **Issue:** Couldn't commit CLAUDE.md
- **Solution:** Renamed to `CLAUDE.local.md` (not ignored)

### Docs Accidentally in PR
- **Issue:** Committed docs to PR branch
- **Solution:** Reset branch, force pushed clean history, created separate docs branch

---

## 8. Current State

- **Branch:** `docs/bug-investigation` (contains CLAUDE.local.md)
- **PR:** #20480 open, targeting `dev`, clean single commit
- **User's LXC:** Patched and working
- **Local repo:** Has uncommitted debug/patch scripts (gitignored)

### Branches in User's Fork
- `fix/system-prompt-duplication` - PR branch with fix only
- `docs/bug-investigation` - Documentation branch

---

## 9. Next Steps

1. **Monitor PR #20480** for maintainer feedback
2. **If requested:** Add testing evidence to PR description
3. **After Open WebUI updates:** Re-apply patch if PR not merged yet
4. **Patch command for LXC:**
   ```bash
   python3 fix_duplicate_content.py /root/.local/share/uv/tools/open-webui/lib/python3.12/site-packages/open_webui
   systemctl restart open-webui
   ```

---

## Related Links

- PR: https://github.com/open-webui/open-webui/pull/20480
- Fork: https://github.com/jvadura/open-webui
- Related Issue: #19656 (partial fix for non-native mode)
