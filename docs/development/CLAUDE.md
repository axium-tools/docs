# CLAUDE.md — Axium Developer Guide

## Overview
Axium is a context-aware DevOps assistant: daemon + CLI + HUD + palette + Spokes plugin system. Built with Python asyncio and Typer, it maintains stateful context (environment, AWS profile, tmux pane) in a background daemon accessible via Unix socket IPC.

## Design Philosophy: The Silent Assistant

**Axium is quiet but powerful.** It's a tool that stays out of your way, never interrupts, and fails gracefully.

### Core Principles

1. **Silent Operation**: Only output what's explicitly requested. No startup banners, no success confirmations, no status messages unless the user asks.

2. **Graceful Degradation**: If something fails (daemon down, config missing), fall back silently to basic functionality. Never break the user's terminal workflow.

3. **No Decoration**: Absolutely no emojis, colors, icons, or visual fluff in core functionality. Plain text only. Users or Spokes can add decoration if they want it.

4. **Invisible Until Needed**: The best interaction with Axium is when you don't notice it's there. Commands just work, context switches happen seamlessly, no ceremony.

5. **Power Through Simplicity**: Complex workflows should feel simple. One command to set context, all subsequent commands automatically adapt.

### The Silent Assistant Pattern

**Good examples:**
```bash
$ ax env root
$ aws s3 ls                    # Just works, no Axium output
$ terraform plan               # Automatically uses root context
$ ax env builder               # Switches context silently
$ aws ec2 describe-instances   # Now uses builder context
```

**Bad examples (anti-patterns):**
```bash
$ ax env root
✓ Environment set to root 🔴  # NO: unnecessary confirmation, emoji
$ aws s3 ls
[Axium] Running with root environment...  # NO: noisy wrapper message
Calling enva-root...                       # NO: implementation details exposed
$ terraform plan
⚠️  Using root context                     # NO: unnecessary warning/decoration
```

**Error handling - Good:**
```bash
$ ax env invalidname
axium: environment 'invalidname' not found in envs.yaml
$ ax env root  # Daemon not running - falls back silently
$ aws s3 ls    # Runs unwrapped, no error, just works
```

**Error handling - Bad:**
```bash
$ ax env root
ERROR: Failed to connect to daemon at /tmp/axiumd.sock!  # NO: breaks workflow
Please run: axium daemon start                           # NO: forces user action
$ aws s3 ls
[Axium Error] No environment set!  # NO: noisy even when falling back
```

---

## Architecture

### Core Components
- **axium/core/daemon.py**: Background process managing state, Unix socket server
- **axium/core/cli.py**: Main Typer CLI entrypoint, auto-loads Spokes
- **axium/core/ipc.py**: IPC client utilities for daemon communication
- **axium/core/spokes.py**: Plugin loader system for extending functionality
- **axium/core/env.py**: Environment abstraction layer for envs.yaml access
- **axium/core/prefix.py**: Command prefix system with template expansion
- **axium/core/hud.py**: Terminal status display (heads-up display)
- **axium/core/palette.py**: Curses-based interactive command palette

### State Management
Daemon maintains:
- `active_env`: Current environment name (references envs.yaml)
- `tmux_pane`: Tmux pane identifier
- `started`: Daemon start timestamp

Environment configuration in `~/.config/axium/envs.yaml`:
```yaml
envs:
  root:
    prefix: enva-root
    aws_profile: root
    color: teal
    region: eu-west-1
```

Access environment data:
```python
from axium.core import env
env.get_active_env_name()  # → "root"
env.get_env_value("prefix")  # → "enva-root"
env.get_env_value("region")  # → "eu-west-1"
```

### IPC Protocol
JSON messages over Unix socket (`/tmp/axiumd.sock`):
- `{"cmd": "ping"}` → health check
- `{"cmd": "get_state"}` → retrieve daemon state
- `{"cmd": "set_env", "value": "root"}` → set active environment
- `{"cmd": "apply_prefixes", "command": "aws", "args": ["s3", "ls"]}` → apply prefix rules
- `{"cmd": "list_prefixed_commands"}` → get commands with prefix rules
- `{"cmd": "stop"}` → graceful daemon shutdown

### Template System
Prefix rules support `${env.<key>}` template variables:
```yaml
# prefixes.yaml
prefixes:
  - command: aws
    prefix: ${env.prefix}  # Expands to "enva-root" when root is active
```

Template expansion resolves environment properties from envs.yaml dynamically.

## Development Guidelines (Claude Focus)

### Primary Objectives
1. **Silent operation**: Only output what's requested, no confirmations or status messages
2. **Graceful degradation**: Fail silently with fallback behavior, never break user workflow
3. **UX/docs polish**: Improve clarity, error messages, help text
4. **Calm technical tone**: Professional, concise, absolutely no emojis/colors/decoration
5. **No new dependencies**: Work within typer, pyyaml, asyncio
6. **Non-breaking changes**: Preserve existing CLI interface and behavior

### Code Style
- Type hints preferred (`str | None` not `Optional[str]`)
- Async where appropriate (daemon, I/O operations)
- Minimal imports, stdlib first
- **Silent error handling**: Log to file, minimal terminal output
- **Graceful fallback**: Prefer silent degradation over error messages
- **No ceremony**: No startup banners, success messages, or confirmations

### Areas for Contribution
- Documentation improvements (docstrings, README enhancements)
- Silent error handling patterns
- Graceful fallback implementations
- CLI help text refinement (minimal, technical)
- HUD display formatting (plain text only)
- Test coverage expansion
- Edge case handling with silent recovery

### Off-Limits
- Adding new external dependencies
- Breaking existing CLI commands
- Changing core architecture without discussion
- Feature additions to core (use Spokes instead)
- **Decorative output**: emojis, colors (except official palette), banners, success messages
- **Breaking terminal workflow**: errors that stop execution without fallback
- **Noisy operation**: status messages, confirmations, verbose output

### Style & Branding

**IMPORTANT:** Read [STYLE-GUIDE.md](STYLE-GUIDE.md) for complete color palette and branding guidelines.

Axium has an official color palette for consistency across CLI, HUD, web, and documentation:

**Core Colors (for reference, not for core CLI output):**
- Primary Teal: `#00B7C7` - Core Axium brand color
- Accent Cyan: `#14D9E7` - Interactive elements, highlights
- Dark Graphite: `#111111` - Background tone
- Soft White: `#E6E6E6` - Default text

**CLI/Core Usage:**
- **Core CLI output should remain plain text** (silent assistant principle)
- Colors only used in:
  - HUD status display (minimal, functional use)
  - Palette interactive elements (future)
  - Web documentation and branding
  - Spokes (optional, if they want to add color)

**Do NOT add colors to:**
- Core command output
- Error messages (plain text only)
- Success messages (which shouldn't exist anyway)
- Log output

If a Spoke wants to add color for visual feedback, they can reference the official palette from STYLE-GUIDE.md, but core remains monochrome and silent.

### CLI Output Patterns

**When to output:**
- Explicit information requests (`axium env get`, `axium env list`)
- Explicit state changes that need confirmation (`axium env set root` - only if error)
- Error conditions that require user awareness (config missing, validation failed)
- Help text and documentation requests (`axium --help`)

**When NOT to output:**
- Successful operations (`axium env set root` succeeds silently)
- Background operations (daemon start/stop)
- State queries from shell functions (use `--quiet` flags)
- Wrapper function execution (completely transparent)

**Error output format:**
```bash
# Good: concise, actionable
axium: environment 'foo' not found in envs.yaml

# Bad: verbose, decorated
Error: The environment named 'foo' does not exist!
Please check your configuration file at ~/.config/axium/envs.yaml
Available environments: root, builder, opt-au
```

**Success output:**
```bash
# Good: silent (no output on success)
$ ax env root
$

# Bad: noisy
$ ax env root
✓ Successfully set environment to 'root'
$
```

## Spoke Development

**IMPORTANT:** Read [SPOKES.md](SPOKES.md) for complete Spoke architecture and contribution guidelines.

### Core Philosophy: "Config lives in core, logic lives in Spokes"

Spokes are **pure Python runtime extensions** with no persistent config:
- **NO config files** - All configuration lives in core (`envs.yaml`, `prefixes.yaml`, etc.)
- **Stateless** - No hidden state, no user configuration per Spoke
- **Event-driven** - React to daemon events, register hooks
- **IPC communication** - Use daemon IPC for all state queries

### Implementation Pattern

```python
def register(app, events):
    """
    Spokes receive two arguments:
    - app: Typer app to register commands
    - events: Event bus for subscribing to hooks

    Access environment data via axium.core.env module directly.
    """

    # Register CLI commands
    @app.command("my-command")
    def my_command():
        # Import env module for environment access
        from axium.core import env

        current_env = env.get_active_env_name()  # Active environment
        aws_profile = env.get_env_value("aws_profile")  # From envs.yaml
        print(f"Running in {current_env}")

    # Subscribe to events
    def on_env_change(new_env, old_env):
        # React to environment changes
        logger.info(f"Environment changed from {old_env} to {new_env}")

    events.on("env_change", on_env_change)

    # Register prefix rules (if applicable)
    from axium.core.prefix import register_prefix_rule
    register_prefix_rule({
        "command": "aws",
        "prefix": "${env.prefix}"
    })
```

### What Spokes Can Do

- ✅ Register CLI commands (`@app.command`)
- ✅ Subscribe to events (`events.on(...)`)
- ✅ Access environment config (`from axium.core import env`)
- ✅ Register prefix rules dynamically
- ✅ Query daemon state via IPC
- ✅ Read core config (envs.yaml, prefixes.yaml)
- ✅ Add HUD/Palette elements (future)

### What Spokes Should NOT Do

- ❌ Store their own YAML/JSON config
- ❌ Manage state directly (daemon owns state)
- ❌ Modify core files
- ❌ Depend on other Spokes directly (use events instead)
- ❌ Block the main thread (use async)

### Spoke Lifecycle

1. **Load** - Daemon scans `spokes/` and imports
2. **Register** - Calls `register(app, events, env)`
3. **Activate** - Commands and hooks become live
4. **Reload** - `axium daemon reload` without restart
5. **Unload** - On daemon exit

### Available Events

- `env_change(new_env)` - After `axium env set`
- `prefix_apply(command, env)` - Before `axium run` executes
- `command_executed(result)` - After wrapped command
- `spoke_loaded(spoke_name)` - When Spoke registers

### Data Access for Spokes

```python
env.current()              # Returns active env key (e.g., "root")
env.get("aws_profile")     # Returns property from envs.yaml
env.all()                  # Returns full envs.yaml mapping

# Via IPC (if needed)
from axium.core.ipc import send_request_sync
resp = send_request_sync({"cmd": "get_state"})
```

## Testing

### Running Tests
```bash
pip install -e ".[dev]"
pytest                    # All tests
pytest tests/test_cli.py  # Specific file
pytest -v --cov=axium    # With coverage
```

### Test Structure
- `tests/conftest.py`: Shared fixtures
- `tests/test_*.py`: Component tests
- Use pytest-asyncio for async tests
- Mock daemon for CLI tests

## File Reference

### Core Module Structure
```
axium/
├── __init__.py
└── core/
    ├── cli.py         # Main Typer app, command definitions
    ├── daemon.py      # Async daemon, IPC server
    ├── ipc.py         # IPC client utilities
    ├── spokes.py      # Plugin loader
    ├── hud.py         # Terminal HUD display
    └── palette.py     # Curses command palette
```

### Configuration Paths
- Config dir: `~/.config/axium/`
- Daemon PID: `~/.config/axium/axiumd.pid`
- Daemon log: `~/.config/axium/axiumd.log`
- Spokes dir: `~/.config/axium/spokes/`
- Socket: `/tmp/axiumd.sock`

## Common Tasks

### Adding CLI Command (Core)
Edit [axium/core/cli.py](axium/core/cli.py):
```python
@app.command()
def new_command(arg: str):
    """Brief description."""
    print(f"Result: {arg}")
```

### Extending Daemon State
Edit [axium/core/daemon.py](axium/core/daemon.py) `__init__` and add handler in `handle_client`.

### Improving Error Messages
Look for bare exceptions or generic error text:
```python
# Before
except Exception as e:
    print(f"Error: {e}")

# After
except FileNotFoundError:
    print("axium: config file not found at ~/.config/axium/config.yaml")
except Exception as e:
    print(f"axium: unexpected error reading config: {e}")
```

### Documentation Enhancement
- Add/improve docstrings (Google style)
- Update README.md with examples
- Add inline comments for complex logic
- Improve CLI help text in command decorators

## Contribution Workflow

1. Read existing code to understand patterns
2. Propose changes aligned with guidelines (non-breaking, UX-focused)
3. Update tests if behavior changes
4. Run `pytest` and `mypy` before committing
5. Keep commits focused, clear messages
6. For features, consider Spoke architecture first

## Resources

### Core Documentation
- Main README: [README.md](README.md)
- Roadmap: [ROADMAP.md](ROADMAP.md)
- Contributing: [CONTRIBUTING.md](CONTRIBUTING.md)

### Architecture & Design
- **Spoke Architecture**: [SPOKES.md](SPOKES.md) - How Spokes work, what they can/cannot do
- **Style Guide**: [STYLE-GUIDE.md](STYLE-GUIDE.md) - Official color palette and branding

### Development
- AI Guidelines: [AI_GUIDELINES.md](AI_GUIDELINES.md)
- Dependencies: [pyproject.toml](pyproject.toml)
- Test config: [pytest.ini](pytest.ini)
- Example configs: [examples/](examples/)

### Key Principles to Remember
1. **Silent Assistant** - No output unless requested, graceful degradation
2. **Config in Core** - Spokes have no config files, all config lives in core
3. **Plain Text** - No colors/emojis in core output (palette exists for branding only)
4. **IPC Communication** - Spokes use IPC, never direct imports
5. **Event-Driven** - Spokes coordinate via events, not direct dependencies
