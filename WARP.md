# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This is an Isaac Sim MCP (Model Context Protocol) Extension and Server that enables natural language control of NVIDIA Isaac Sim through AI assistants like Claude. It consists of two main components:

1. **Isaac Sim Extension** - Runs inside Isaac Sim and listens for commands via socket (localhost:8766)
2. **MCP Server** - Acts as a bridge between AI assistants (via MCP protocol) and the Isaac Sim extension

## Architecture

### Two-Process Architecture
- **MCP Server** (`isaac_mcp/server.py`): FastMCP server that provides tools for AI assistants
- **Isaac Extension** (`isaac.sim.mcp_extension/isaac_sim_mcp_extension/extension.py`): Socket server running inside Isaac Sim
- Communication: TCP socket on localhost:8766 with JSON command/response protocol

### Key Components
- **Socket Communication**: Bidirectional JSON commands with large response handling (300s timeout)
- **Async Execution**: MCP server uses asyncio; Isaac extension uses `omni.kit.async_engine.run_coroutine`
- **Command Handlers**: Extension has handlers for physics scene creation, robot spawning, script execution, 3D generation
- **3D Generation**: Beaver3D integration for text/image-to-3D (`gen3d.py`) with async task monitoring
- **USD Search**: Search and load USD assets from libraries (`usd.py`)

## Prerequisites

### Required Environment Variables
```bash
# For 3D generation (optional but recommended)
export ARK_API_KEY=<Your_Beaver3D_API_Key>
export BEAVER3D_MODEL=<your_beaver3d_model_name>
export NVIDIA_API_KEY=<Your_NVIDIA_API_Key>

# Optional
export USD_WORKING_DIR=/tmp/usd  # Default working directory for USD files
```

### System Requirements
- NVIDIA Isaac Sim 4.2.0 or higher
- Python 3.9+
- uv/uvx package manager
- mcp[cli] in base environment

## Development Commands

### Starting Isaac Sim with Extension
```bash
# Navigate to Isaac Sim installation
cd ~/.local/share/ov/pkg/isaac-sim-4.2.0

# Start with extension enabled
./isaac-sim.sh --ext-folder ~/Documents/isaac-sim-mcp/ --enable isaac.sim.mcp_extension

# On Windows
isaac-sim.bat --ext-folder C:\Users\<username>\Documents\isaac-sim-mcp --enable isaac.sim.mcp_extension
```

### Testing MCP Server Standalone
```bash
# Test server can start
uv run ~/Documents/isaac-sim-mcp/isaac_mcp/server.py

# On Windows
uv run C:\Users\<username>\Documents\isaac-sim-mcp\isaac_mcp\server.py
```

### Development Mode with MCP Inspector
```bash
# Debug MCP server with web interface at http://localhost:5173
uv run mcp dev ~/Documents/isaac-sim-mcp/isaac_mcp/server.py
```

### Testing Individual Components
```bash
# Test Beaver3D integration
cd isaac.sim.mcp_extension/isaac_sim_mcp_extension
python gen3d.py

# Test USD loader
python usd.py
```

## Working with the Codebase

### Adding New MCP Tools

1. Add tool definition in `isaac_mcp/server.py`:
```python
@mcp.tool("my_new_tool")
def my_new_tool(param1: str, param2: int) -> str:
    """Tool description for AI assistant"""
    isaac = get_isaac_connection()
    result = isaac.send_command("my_command", {"param1": param1, "param2": param2})
    return result
```

2. Add handler in `isaac.sim.mcp_extension/isaac_sim_mcp_extension/extension.py`:
```python
# In _execute_command_internal, add to handlers dict:
handlers = {
    # ... existing handlers
    "my_command": self.my_command_handler,
}

# Add implementation:
def my_command_handler(self, param1, param2):
    """Implementation inside Isaac Sim"""
    # Your code here
    return {"status": "success", "message": "Done"}
```

### Robot Creation Pattern

Standard robots are available via `create_robot()` tool with types: "franka", "jetbot", "carter", "g1", "go1"

Robot asset paths follow pattern:
```python
assets_root_path + "/Isaac/Robots/{Brand}/{Model}/{model}.usd"
```

Always position robots using `XFormPrim`:
```python
robot_prim = XFormPrim(prim_path="/World/RobotName")
robot_prim.set_world_pose(position=np.array([x, y, z]))
```

### Physics Scene Setup

Standard pattern for creating physics simulations:
1. Create physics scene with `create_physics_scene()`
2. Add objects with proper physics properties
3. Initialize `World` or `SimulationContext`
4. Use `step_async()` for non-blocking simulation steps

### Socket Communication Details

**Large Response Handling**: Extension sends responses in chunks; server uses `receive_full_response()` to reassemble

**Timeout Management**: 300 second timeout for complex operations (3D generation, long simulations)

**Error Recovery**: Connection automatically recreates on timeout/failure

### Working with Async Code

Isaac Sim requires async operations for long-running tasks:
```python
from omni.kit.async_engine import run_coroutine

# Schedule async task
task = run_coroutine(my_async_function())

# Async functions can use:
await asyncio.sleep(delay)
my_world.step_async()  # Non-blocking simulation step
```

### 3D Generation Workflow

1. Generate task ID from text/image
2. Monitor task status asynchronously (75s+ for high-quality USD)
3. Download and extract ZIP when complete
4. Load USD into scene with USDLoader
5. Apply transformations (position/scale)

Task results are cached by image URL or text prompt to avoid regeneration.

## Important Patterns

### Script Execution Safety
The `execute_script()` tool allows arbitrary Python code execution inside Isaac Sim. Always:
- Check scene connection with `get_scene_info()` first
- Create physics scene before robot operations
- Use try/except blocks for error handling
- Provide formatted code preview before execution

### Robot Simulation Initialization
Follow this sequence to avoid "no active physics scene" errors:
```python
my_world = World(stage_units_in_meters=1.0)
my_world.initialize_physics()
if not my_world.is_playing():
    my_world.play()
# Wait for physics to stabilize
for _ in range(10):
    my_world.step_async()
# Now safe to initialize robots
robot.initialize(my_world.physics_sim_view)
```

### Resource Management
- USD files: Stored in `USD_WORKING_DIR` (default `/tmp/usd`)
- Task caching: `_image_url_cache` and `_text_prompt_cache` in extension
- Connection pooling: Single persistent `IsaacConnection` instance

## Common Issues

### "No active physics scene" Error
- Ensure `World.play()` is called before initializing articulations
- Let physics stabilize with multiple `step_async()` calls
- Initialize physics context before accessing `physics_sim_view`

### Socket Timeout
- Default 300s timeout for long operations
- Simplify requests if timeout occurs
- Check Isaac Sim is responsive (not blocked by UI operations)

### Extension Not Starting
- Verify extension path is correct in `--ext-folder`
- Check logs for `isaac.sim.mcp_extension-0.1.0 startup`
- Ensure port 8766 is not in use

## Example Workflows

### Creating Robot Simulation
```python
# 1. Check connection
get_scene_info()

# 2. Create physics environment
create_physics_scene(floor=True, gravity=[0, -9.81, 0])

# 3. Add robot
create_robot(robot_type="franka", position=[0, 0, 0])

# 4. For complex control, use execute_script with proper async patterns
```

### Generating 3D Assets
```python
# From image
generate_3d_from_text_or_image(
    image_url="https://example.com/image.jpg",
    position=[0, 0, 50],
    scale=[10, 10, 10]
)

# From text
generate_3d_from_text_or_image(
    text_prompt="A red apple",
    position=[0, 0, 50],
    scale=[10, 10, 10]
)
```

### Searching and Loading USD Assets
```python
search_3d_usd_by_text(
    text_prompt="office desk",
    target_path="/World/my_desk",
    position=[0, 5, 0],
    scale=[3, 3, 3]
)
```

## Testing Approach

When developing new features:
1. Test MCP server standalone first
2. Test extension handlers with simple socket client
3. Verify in MCP inspector (http://localhost:5173)
4. Test end-to-end with AI assistant (Cursor, Claude Desktop)

Use the example scripts in `isaac.sim.mcp_extension/examples/` as reference implementations for different robot types.
