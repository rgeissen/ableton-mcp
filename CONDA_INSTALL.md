# Conda Installation

## 1. Create environment

```bash
conda create -n ableton-mcp python=3.11 -y
```

## 2. Install package

```bash
conda activate ableton-mcp
pip install -e /path/to/ableton-mcp
```

Or directly from PyPI:

```bash
pip install ableton-mcp
```

## 3. Find the binary path

```bash
which ableton-mcp
# e.g. /opt/miniconda3/envs/ableton-mcp/bin/ableton-mcp
```

## 4. Configure Claude Code

Add to `~/.claude/mcp.json`:

```json
{
  "mcpServers": {
    "AbletonMCP": {
      "command": "/opt/miniconda3/envs/ableton-mcp/bin/ableton-mcp"
    }
  }
}
```

## 5. Configure Claude Desktop (optional)

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS):

```json
{
  "mcpServers": {
    "AbletonMCP": {
      "command": "/opt/miniconda3/envs/ableton-mcp/bin/ableton-mcp"
    }
  }
}
```

> Run only one instance at a time (Claude Code **or** Claude Desktop, not both).

## 6. Install the Ableton Remote Script

1. Copy the `AbletonMCP_Remote_Script/__init__.py` file into a new folder called `AbletonMCP` inside Ableton's MIDI Remote Scripts directory:
   - **macOS:** `Applications/Ableton Live.app/Contents/App-Resources/MIDI Remote Scripts/`
   - **Windows:** `C:\ProgramData\Ableton\Live XX\Resources\MIDI Remote Scripts\`
2. In Ableton: **Preferences → Link, Tempo & MIDI → Control Surface → AbletonMCP**
3. Set Input and Output to **None**
4. Restart Claude Code or Claude Desktop
