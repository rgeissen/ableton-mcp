# Ableton MCP — Codebase Guide

## Architecture

Two independent components communicate over a local TCP socket:

```
Claude / MCP Client
      │
      ▼
MCP_Server/server.py          ← FastMCP server (runs outside Ableton)
      │  TCP socket :5005
      ▼
AbletonMCP_Remote_Script/     ← Ableton MIDI Remote Script (runs inside Ableton's Python)
__init__.py
      │
      ▼
Live Python API (Live.*)      ← Ableton's internal API, only accessible from inside Live
```

The remote script is installed in Ableton's MIDI preferences as a Control Surface. It opens a socket server on port 5005 and accepts JSON commands from the MCP server.

---

## Key Files

| File | Role |
|---|---|
| `MCP_Server/server.py` | All MCP tools. Also contains SQLite helpers for tag-based search. |
| `AbletonMCP_Remote_Script/__init__.py` | Command dispatcher + all Live API calls. |
| `pyproject.toml` | Single dependency: `mcp[cli]>=1.3.0`. No other packages needed. |

---

## Communication Protocol

Commands are JSON sent over TCP. The MCP server calls `ableton.send_command(type, params)` which serialises to:

```json
{ "type": "get_session_info", "params": {} }
```

The remote script responds with:

```json
{ "status": "success", "result": { ... } }
```

State-mutating commands (create track, load device, etc.) are dispatched to Ableton's main thread via `schedule_message(0, fn)` with a `queue.Queue` for the response. Read-only commands run directly on the socket thread.

---

## Tag-Based Browser Search

`get_browser_tags` and `search_by_tags` bypass the socket entirely and read Ableton's local SQLite database directly:

```
~/Library/Application Support/Ableton/Live Database/Live-files-<version>.db
```

**Key schema facts:**
- `files` table — every browser item. `file_type` is a FourCC integer (`wav-`, `adg-`, `adv-`, `keyw`, etc.).
- `file_type = 1801812343` (`keyw`) — tag nodes. One row per tag name.
- `keywords` table — many-to-many join between files and their tags (`file_id → keyw_id`).
- `ancestors` table — ordered parent chain for each file, used to reconstruct filesystem paths.
- `places` table — maps a file's `place_id` to its pack/library name (e.g. "User Library", "Build and Drop").

**Why not the Live API?** `Live.Browser.BrowserItem` does not expose a `tags` property (confirmed against `ableton-live-stubs 12.3.6`). Tags only exist in the SQLite DB.

**URI caveat:** `BrowserItem.uri` is generated at runtime by Ableton and is not stored in the DB. Tag search returns a `browser_path` (reconstructed from ancestors) which can be passed to the existing `get_browser_items_at_path` tool to obtain a live URI for loading.

---

## Adding a New Tool

1. Add the handler method to `AbletonMCP_Remote_Script/__init__.py`:
   - Read-only → add an `elif command_type == "..."` branch directly in the dispatcher (around line 359).
   - State-mutating → add to the `command_type in [...]` list (around line 229) and add a branch inside `main_thread_task`.

2. Add the `@mcp.tool()` function to `MCP_Server/server.py`. Call `ableton.send_command("...", params)`.

3. No package installs needed — stdlib only beyond `mcp[cli]`.

---

## Ableton Live API Constraints

- The `Live` module is only importable inside Ableton's Python runtime — never from the MCP server.
- `BrowserItem` properties: `name`, `uri`, `is_folder`, `is_device`, `is_loadable`, `is_selected`, `children`, `source`. No tags, no metadata.
- `Browser` root properties: `instruments`, `sounds`, `drums`, `audio_effects`, `midi_effects`, `samples`, `clips`, `packs`, `user_library`, `current_project`, `max_for_live`, `plugins`.
- State mutations must run on Ableton's main thread — use `schedule_message(0, fn)`.
