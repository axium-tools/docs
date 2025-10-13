# Command Registry

The Command Registry is Axium's dynamic command discovery system that powers the interactive palette and enables automatic command cataloging.

## Overview

Every command registered with Axium's Typer CLI—whether from core or from Spokes—is automatically captured and cataloged in a global registry. This enables:

- **Dynamic command discovery** in the palette
- **Source attribution** (core vs spoke commands)
- **Automatic command listing** without manual maintenance
- **Metadata preservation** (help text, groups, callbacks)

## How It Works

### Automatic Registration

Commands are registered automatically during Axium initialization:

1. **Core commands** are introspected after `load_spokes()` completes
2. **Spoke commands** are tagged as each Spoke's `register()` function runs
3. **Command metadata** is extracted from Typer's internal structures

```python
# In cli.py _autoload() callback
load_spokes(app)
registry.introspect_typer_app(app, source="core")
```

### Command Metadata

Each registered command includes:

```python
{
    "name": "daemon start",      # Full command path
    "help": "Start the daemon",  # From docstring
    "source": "core",            # "core" or spoke name
    "group": "daemon"            # Parent group if applicable
}
```

## Using the Registry

### In the Palette

The palette dynamically builds its menu from the registry:

```bash
$ axium palette
╔══ Axium Palette ══╗
> [core] daemon start
  [core] daemon status
  [core] env set
  [core] env get
  [aws] whoami
  quit
```

Commands are tagged with `[source]` to show their origin.

### Reloading Commands

Use `--reload` to refresh the command list after installing new Spokes:

```bash
$ axium palette --reload
```

This clears the registry and re-introspects all commands, picking up any newly installed Spokes.

### Programmatic Access

The registry can be queried programmatically:

```python
from axium.core import registry

# Get all commands (sorted alphabetically)
commands = registry.get_all_commands()

# Get commands from specific source
core_commands = registry.get_commands_by_source("core")
aws_commands = registry.get_commands_by_source("aws")

# Get commands with grouped sorting (core first)
commands = registry.get_all_commands(sort_by="grouped")
```

## Spoke Integration

When Spokes register commands, they're automatically tagged:

```python
# In a Spoke's register function
def register(app, events):
    @app.command("whoami")
    def aws_whoami():
        """Get AWS caller identity."""
        import boto3
        print(boto3.client("sts").get_caller_identity())
```

After loading, this command appears in the registry as:

```python
{
    "name": "whoami",
    "help": "Get AWS caller identity.",
    "source": "aws",
    "group": None
}
```

## Registry API

### Functions

**`register_command(name, help, source, group=None, callback=None)`**
- Manually register a command (rarely needed)
- Warns if overwriting existing command

**`get_all_commands(sort_by="alpha")`**
- Get all registered commands
- Sort options: `"alpha"`, `"grouped"`, `"usage"` (future)

**`get_commands_by_source(source)`**
- Filter commands by source

**`clear_registry()`**
- Clear all registered commands
- Used for testing and `--reload`

**`introspect_typer_app(app, source, prefix="")`**
- Walk Typer app structure and register commands
- Handles nested subgroups recursively
- Called automatically during initialization

## Implementation Details

### When Commands Are Registered

```
CLI Startup
    ↓
_autoload() callback
    ↓
load_spokes(app)
    ├─ For each spoke:
    │   ├─ spoke.register(app, events)
    │   └─ introspect_typer_app(app, source=spoke_name)
    │
    └─ introspect_typer_app(app, source="core")

Result: All commands now in registry
```

### Introspection Process

The `introspect_typer_app()` function:

1. Iterates `app.registered_commands` (direct commands)
2. Iterates `app.registered_groups` (subcommands)
3. Recursively processes nested groups
4. Extracts metadata from Typer's CommandInfo objects
5. Registers each command with appropriate source tag

### Command Naming

- **Root commands**: Use function name or explicit `@app.command("name")`
- **Subcommands**: Full path with group prefix (`"daemon start"`)
- **Nested groups**: Multiple prefixes (`"l1 l2 deep"`)

## Best Practices

### For Core Development

- No manual updates needed—commands auto-register
- Use clear docstrings (first line becomes help text)
- Group related commands with `typer.Typer()` subapps

### For Spoke Development

- Commands automatically tagged with spoke name
- No special registration needed
- Help text extracted from docstrings
- Multiple commands per spoke fully supported

### For Testing

- Use `clean_registry` fixture to reset between tests
- Test with mock Typer apps for isolation
- Verify source tagging with `get_commands_by_source()`

## Future Enhancements

Planned registry features:

- **Usage tracking**: Sort by frequency in palette
- **Command history**: Persistent favorites
- **Search/filter**: Fuzzy find in palette
- **Command aliases**: User-defined shortcuts
- **Help browser**: TUI for exploring command documentation

## See Also

- [Palette Guide](../guides/palette.md) - Using the interactive palette
- [Spokes Documentation](spokes.md) - Plugin system overview
- [CLI Reference](../reference/api/cli.md) - Core commands
