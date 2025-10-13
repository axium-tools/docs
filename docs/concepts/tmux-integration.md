# tmux Integration

Axium provides deep integration with tmux, enabling per-pane environment contexts and rich status line displays.

## Overview

When running inside tmux, Axium automatically detects pane IDs and maintains independent environment contexts for each pane. This allows you to work with different environments simultaneously in different panes.

## Features

### Per-Pane Environments

Each tmux pane can have its own active environment:

```bash
# In pane %1
$ axium env set prod
axium: pane %1 env set → prod

# In pane %2 (different environment)
$ axium env set dev
axium: pane %2 env set → dev
```

The daemon maintains a mapping of pane IDs to environments:

```json
{
  "panes": {
    "%1": "prod",
    "%2": "dev",
    "%3": "staging"
  }
}
```

### HUD Status Line

Axium integrates with tmux's status line via the `axium hud` command:

```bash
$ axium hud --pane %1
[axium] pane:%1  env:prod  aws:production  uptime:2h15m
```

The HUD displays:
- Current pane ID
- Active environment for the pane
- AWS profile (if set)
- Daemon uptime
- Custom segments from Spokes (extensible)

### Palette Keybinding

Press `Ctrl+G` in any tmux pane to launch the Axium palette:

```
╔══ Axium Palette ══╗  pane:%1 env:prod
  [core] daemon start
  [core] daemon status
  [core] env set
> [core] env get
  [aws] whoami
  quit
```

The palette header shows your current pane and environment context.

## Setup

### 1. Source Axium Shell Integration

Add to your `~/.bashrc` or `~/.zshrc`:

```bash
source ~/.config/axium/bash/init.sh
```

This automatically loads tmux integration when you're in a tmux session.

### 2. tmux Configuration

The `~/.config/axium/tmux/init.sh` file is created automatically by Axium on first run. It provides modular helper functions that you can enable selectively.

#### Default Mode (Automatic)

By default, Axium enables:
- **Status Line**: Sets `status-right` to display Axium HUD
- **Keybinding**: Binds `Ctrl+G` to launch palette
- **Pane Restore**: Restores pane environment on shell init (always enabled)

#### Minimal Mode (Full Control)

Set `AXIUM_TMUX_MINIMAL=1` to disable automatic configuration:

```bash
# In your ~/.bashrc or ~/.zshrc (before sourcing bash/init.sh)
export AXIUM_TMUX_MINIMAL=1
source ~/.config/axium/bash/init.sh
```

Then selectively enable features by calling helper functions:

```bash
# Enable only HUD (no palette keybinding)
axium_tmux_enable_hud

# Enable only palette keybinding (no HUD)
axium_tmux_enable_palette

# Enable both (equivalent to default mode)
axium_tmux_enable_hud
axium_tmux_enable_palette

# Add custom segments (for Spokes or user scripts)
axium_tmux_enable_segment "kubectl config current-context" "cyan"
```

#### Manual Setup (Alternative)

If you prefer managing tmux configuration in `~/.tmux.conf`:

```bash
# In your ~/.tmux.conf
set -g status-right '#(axium hud --pane #D)'
set -g status-right-length 100
set -g status-interval 5
bind-key -n C-g run-shell "tmux send-keys 'axium palette' Enter"
```

### 3. Start Axium Daemon

```bash
axium daemon start
```

## Usage

### Setting Pane Environment

In tmux, `axium env set` automatically detects the current pane:

```bash
$ axium env set prod
axium: pane %1 env set → prod
```

Outside tmux, it sets the global environment:

```bash
$ axium env set prod
axium: env set → prod
```

### Querying Pane Environment

Get the current pane's environment:

```bash
$ axium env get
prod
```

Query a specific pane:

```bash
$ axium env get --pane %2
dev
```

### Listing Pane Mappings

Show all pane-to-environment mappings:

```bash
$ axium env list --panes
Available environments:
  * prod (prefix, aws_profile, region, color)
    dev (prefix, aws_profile, region, color)

Pane Mappings:
  %1 → prod
  %2 → dev
  %3 → staging
```

### Running Commands with Pane Context

When you run commands via `axium run`, the daemon uses the pane's environment:

```bash
# In pane %1 (env: prod)
$ aws s3 ls
# Actually runs: enva-prod aws s3 ls

# In pane %2 (env: dev)
$ aws s3 ls
# Actually runs: enva-dev aws s3 ls
```

## Spoke Integration

Spokes can extend Axium's tmux integration in two ways:

### 1. HUD Segments (Python-side)

Register HUD segment providers that run in the daemon:

```python
# In your spoke's register() function
from axium.core.hud import register_hud_provider

def my_hud_segment():
    """Provide custom HUD segment."""
    # Query current k8s context, docker status, etc.
    return "k8s:prod"

register_hud_provider(my_hud_segment)
```

The HUD will display:

```
[axium] pane:%1  env:prod  aws:production  uptime:2h15m  k8s:prod
```

**Provider Guidelines:**
- Return a string segment (e.g., `"key:value"`)
- Return empty string `""` to skip
- Keep segments short (status line space is limited)
- Handle exceptions gracefully (providers are silently skipped on error)
- Avoid expensive operations (HUD refreshes every 5 seconds)

### 2. tmux Configuration (Shell-side)

Spokes can extend tmux configuration using helper functions in their shell integration files:

```bash
# In your spoke's shell integration file (e.g., spokes/my-spoke/init.sh)

# Add custom HUD segment to status-right
axium_tmux_enable_segment "my-spoke status" "green"

# Or configure tmux directly
if [[ -n "$TMUX" ]]; then
  tmux set-hook -g pane-focus-in "run-shell 'my-spoke sync #{pane_id}'"
fi
```

**Available Helpers:**
- `axium_tmux_enable_segment CMD [COLOR]` - Append custom segment to status-right
- `axium_tmux_enable_hud` - Enable Axium HUD (if user disabled it)
- `axium_tmux_enable_palette` - Enable palette keybinding
- `axium_restore_pane_env` - Restore pane environment (usually not needed)

**Example Spoke Integration:**

```bash
# spokes/kubectl/init.sh
if [[ -n "$TMUX" ]]; then
  # Add kubectl context to status line
  axium_tmux_enable_segment "kubectl config current-context" "cyan"
fi
```

This adds the current kubectl context to every tmux pane's status line.

## Architecture

### State Management

The daemon maintains two environment contexts:

1. **Global Environment** (`state["active_env"]`): Used outside tmux
2. **Per-Pane Environments** (`state["panes"]`): Used inside tmux

```python
{
    "active_env": "root",       # Global fallback
    "panes": {
        "%1": "prod",           # Pane-specific overrides
        "%2": "dev",
        "%3": "staging"
    }
}
```

### Pane Detection

Commands detect tmux context via the `TMUX_PANE` environment variable:

```python
import os

pane_id = os.getenv("TMUX_PANE")  # e.g., "%1"
if pane_id:
    # Use pane-specific environment
    pass
else:
    # Use global environment
    pass
```

### Pane Restoration

When you open a new shell in an existing pane, the `axium_restore_pane_env()` function queries the daemon for the pane's saved environment and exports `AXIUM_ENV`:

```bash
# In ~/.config/tmux/init.sh
axium_restore_pane_env() {
  local pane_env=$(axium env get 2>/dev/null)
  if [[ -n "$pane_env" && "$pane_env" != "None" ]]; then
    export AXIUM_ENV="$pane_env"
  fi
}
```

### IPC Commands

The daemon provides three new IPC commands for pane management:

#### `set_pane_env`

Set environment for a specific pane:

```json
{
  "cmd": "set_pane_env",
  "pane": "%1",
  "value": "prod"
}
```

Response:

```json
{
  "ok": true
}
```

#### `get_pane_env`

Get environment for a specific pane:

```json
{
  "cmd": "get_pane_env",
  "pane": "%1"
}
```

Response:

```json
{
  "ok": true,
  "env": "prod"
}
```

#### `clear_pane_env`

Clear environment mapping for a pane:

```json
{
  "cmd": "clear_pane_env",
  "pane": "%1"
}
```

Response:

```json
{
  "ok": true
}
```

## Troubleshooting

### Axium Overwriting My tmux Configuration

If Axium is overwriting your custom `status-right` or keybindings, enable minimal mode:

```bash
# In your ~/.bashrc or ~/.zshrc (before sourcing bash/init.sh)
export AXIUM_TMUX_MINIMAL=1
```

Then selectively enable only the features you want:

```bash
# Example: Enable HUD but not palette keybinding
axium_tmux_enable_hud
```

### HUD Not Updating

Check tmux refresh interval:

```bash
tmux show-option -g status-interval
```

Set to 5 seconds (or lower):

```bash
tmux set-option -g status-interval 5
```

### Ctrl+G Not Working

**Option 1:** Check if minimal mode is enabled and you forgot to enable palette:

```bash
# Enable palette explicitly
axium_tmux_enable_palette
```

**Option 2:** Verify keybinding:

```bash
tmux list-keys | grep C-g
```

Manually bind:

```bash
tmux bind-key -n C-g run-shell "tmux send-keys 'axium palette' Enter"
```

### HUD Shows "[axium] inactive"

This means the daemon is not running or not responding. Check daemon status:

```bash
axium daemon status
```

If not running, start it:

```bash
axium daemon start
```

### Pane Environment Not Persisting

Ensure daemon is running:

```bash
axium daemon status
```

Check state file:

```bash
cat ~/.config/axium/state.json
```

### Wrong Pane ID Detected

Check `TMUX_PANE` variable:

```bash
echo $TMUX_PANE
```

If empty, you're not in tmux. If incorrect, restart your shell.

## Best Practices

1. **Use Per-Pane Environments**: Keep different environments in different panes for safety
2. **Color Code Environments**: Use tmux pane border colors to distinguish environments visually
3. **Monitor HUD**: Keep an eye on the status line to verify your current context
4. **Use Palette**: Press `Ctrl+G` for quick command access
5. **Extend HUD**: Add custom segments for project-specific context

## Example Workflow

```bash
# Create new tmux session
tmux new -s work

# Pane 1: Production monitoring
axium env set prod
aws cloudwatch get-metric-statistics ...

# Split pane (Ctrl+B, %)
# Pane 2: Development work
axium env set dev
terraform plan

# Split pane (Ctrl+B, ")
# Pane 3: Staging deployment
axium env set staging
kubectl get pods

# Press Ctrl+G in any pane to access palette
# HUD shows current context: [axium] pane:%1  env:prod  aws:production  uptime:1h30m
```

## Future Enhancements

- Automatic pane cleanup (remove stale pane IDs)
- Pane grouping (multiple panes share environment)
- Visual indicators (colored borders based on environment)
- Pane templates (save/restore pane layouts with environments)
- Integration with tmux-resurrect for session persistence
