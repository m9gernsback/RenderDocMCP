# RenderDoc MCP Server

MCP server that operates as a RenderDoc UI extension. Enables AI assistants to access RenderDoc capture data and assist with DirectX 11/12 and Vulkan graphics debugging.

## Architecture

**Hybrid process isolation**:

```
Claude/AI Client (stdio)
        │
        ▼
MCP Server Process (Standard Python + FastMCP 2.0)
        │ File-based IPC (%TEMP%/renderdoc_mcp/)
        ▼
RenderDoc Process (Extension + File Polling)
```

## Project Structure

```
RenderDocMCP/
├── mcp_server/                        # MCP server
│   ├── server.py                      # FastMCP entry point
│   ├── config.py                      # Configuration
│   └── bridge/
│       └── client.py                  # File-based IPC client
│
├── renderdoc_extension/               # RenderDoc extension
│   ├── __init__.py                    # register()/unregister()
│   ├── extension.json                 # Manifest
│   ├── socket_server.py               # File-based IPC server
│   ├── request_handler.py             # Request handling
│   └── renderdoc_facade.py            # RenderDoc API wrapper
│
└── scripts/
    └── install_extension.py           # Extension installer
```

## MCP Tools

| Tool | Description |
|------|-------------|
| `list_captures` | List .rdc files in a specified directory |
| `open_capture` | Open a capture file (auto-closes existing capture) |
| `get_capture_status` | Check capture loading status |
| `get_draw_calls` | Draw call list (hierarchical, with filtering) |
| `get_frame_summary` | Frame statistics (draw call count, marker list, etc.) |
| `find_draws_by_shader` | Reverse lookup draw calls by shader name |
| `find_draws_by_texture` | Reverse lookup draw calls by texture name |
| `find_draws_by_resource` | Reverse lookup draw calls by resource ID |
| `get_draw_call_details` | Details of a specific draw call |
| `get_action_timings` | GPU execution time for actions |
| `get_shader_info` | Shader source / constant buffers |
| `get_shader_bytecode` | Write shader binary to file (SPIR-V/DXBC/DXIL) |
| `get_buffer_contents` | Buffer data (with offset/length) |
| `get_texture_info` | Texture metadata |
| `get_texture_data` | Texture pixel data (mip/slice/3D slice support) |
| `get_pipeline_state` | Full pipeline state |

### get_draw_calls Filtering Options

```python
get_draw_calls(
    include_children=True,      # Include child actions
    marker_filter="Camera.Render",  # Only actions under this marker
    exclude_markers=["GUI.Repaint", "UIR.DrawChain"],  # Markers to exclude
    event_id_min=7372,          # event_id range start
    event_id_max=7600,          # event_id range end
    only_actions=True,          # Exclude markers (draw calls only)
    flags_filter=["Drawcall", "Dispatch"],  # Only specific flags
)
```

### Capture Management Tools

```python
# List capture files in a directory
list_captures(directory="D:\\captures")
# → {"count": 3, "captures": [{"filename": "game.rdc", "path": "...", "size_bytes": 12345, "modified_time": "..."}, ...]}

# Open a capture file (auto-closes any existing capture)
open_capture(capture_path="D:\\captures\\game.rdc")
# → {"success": true, "filename": "game.rdc", "api": "D3D11"}
```

### Reverse Lookup Tools

```python
# Search by shader name (partial match)
find_draws_by_shader(shader_name="Toon", stage="pixel")

# Search by texture name (partial match)
find_draws_by_texture(texture_name="CharacterSkin")

# Search by resource ID (exact match)
find_draws_by_resource(resource_id="ResourceId::12345")
```

### GPU Timing

```python
# Get timings for all actions
get_action_timings()
# → {"available": true, "unit": "CounterUnit.Seconds", "timings": [...], "total_duration_ms": 12.5, "count": 150}

# Get timings for specific event IDs
get_action_timings(event_ids=[100, 200, 300])

# Filter by marker
get_action_timings(marker_filter="Camera.Render", exclude_markers=["GUI.Repaint"])
```

**Note**: GPU timing counters may not be available on all hardware/drivers.
If `available: false` is returned, timing information cannot be retrieved for that capture.

### Shader Bytecode

```python
# Write shader binary to file
get_shader_bytecode(event_id=2607, stage="pixel", output_path="D:\\temp\\EID2607.fs.spv")
# → {"resource_id": "ResourceId::100344", "entry_point": "main_0001bc38_fa74d7fe", "stage": "pixel", "data_length": 113720, "output_path": "D:\\temp\\EID2607.fs.spv"}

# Vertex shader
get_shader_bytecode(event_id=2607, stage="vertex", output_path="D:\\temp\\EID2607.vs.spv")

# Compute shader
get_shader_bytecode(event_id=500, stage="compute", output_path="D:\\temp\\EID500.cs.spv")
```

**Output format** depends on capture API: Vulkan → SPIR-V, D3D11 → DXBC, D3D12 → DXIL.
Writes directly to file (no base64 encoding), keeping the response lightweight and context-friendly.
The `entry_point` field can be passed directly to external tools like AOC.

## Communication Protocol

File-based IPC:
- IPC directory: `%TEMP%/renderdoc_mcp/`
- `request.json`: Request (MCP server → RenderDoc)
- `response.json`: Response (RenderDoc → MCP server)
- `lock`: Write-in-progress lock file
- Polling interval: 100ms (RenderDoc side)

## Development Notes

- File-based IPC is used because RenderDoc's built-in Python lacks socket/QtNetwork modules
- RenderDoc extension uses only Python 3.6 standard library
- ReplayController access is performed via `BlockInvoke`

## WSL Setup (Claude Code on WSL + RenderDoc on Windows)

When running Claude Code on WSL with RenderDoc on Windows, the MCP server must be launched using **Windows Python** so that both the MCP server and RenderDoc extension share the same `%TEMP%` path for file-based IPC.

```
Claude Code (WSL) ──stdio──→ python.exe (Windows) ──file IPC──→ RenderDoc (Windows)
```

### Prerequisites

1. **Conda environment** with dependencies:
   ```powershell
   conda activate RenderDocMCPPy
   cd E:\GitHub\RenderDocMCP
   pip install -e .
   ```

2. **RenderDoc extension installed**:
   ```powershell
   python scripts/install_extension.py
   ```

### MCP Configuration (`.claude/mcp.json`)

```json
{
  "mcpServers": {
    "renderdoc": {
      "command": "/mnt/c/Users/<username>/anaconda3/envs/RenderDocMCPPy/python.exe",
      "args": ["-m", "mcp_server.server"],
      "cwd": "E:\\GitHub\\RenderDocMCP"
    }
  }
}
```

**Key point**: Using `python.exe` (Windows Python) ensures `tempfile.gettempdir()` resolves to the Windows `%TEMP%` path, matching the RenderDoc extension's IPC directory. No path translation between WSL and Windows is needed.

### Launch Steps

1. Open RenderDoc and enable the extension in **Tools > Manage Extensions > RenderDoc MCP Bridge > Load** (check "Always Load" for auto-load on startup)
2. Run `/mcp` in Claude Code or restart the session

## References

- [FastMCP](https://github.com/jlowin/fastmcp)
- [RenderDoc Python API](https://renderdoc.org/docs/python_api/index.html)
- [RenderDoc Extension Registration](https://renderdoc.org/docs/how/how_python_extension.html)
