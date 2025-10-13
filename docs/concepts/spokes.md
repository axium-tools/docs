# Spokes

Spokes are Axium's plugin system — modular extensions that add commands, react to events, and integrate with external tools without modifying the core.

---

## Overview

Spokes allow you to:
- Add custom CLI commands
- React to environment changes
- Integrate with AWS, Kubernetes, Docker, etc.
- Extend the HUD and palette
- Share functionality across teams

---

## Architecture

Each Spoke lives in `~/.config/axium/spokes/<spoke-name>/`:

```
~/.config/axium/spokes/
  aws/
    spoke.yaml       # Manifest
    aws_spoke.py     # Implementation
  k8s/
    spoke.yaml
    k8s_spoke.py
```

---

## Spoke Manifest

Every Spoke has a `spoke.yaml` manifest:

```yaml
name: aws
version: 1.0.0
description: AWS operations and utilities
entrypoint: aws_spoke:register
permissions:
  exec: false
  notify: true
  fs_read:
    - ~/.aws/credentials
    - ~/.aws/config
  fs_write: []
```

- `name`: Unique identifier for the Spoke
- `version`: Semantic version string
- `description`: Short description for spoke list
- `entrypoint`: Python module and function to call (format: `module:function`)
- `permissions`: Permissions required by the spoke (see [Permissions](permissions.md))

---

## Basic Spoke

Here's a minimal Spoke that adds a command:

```python
# ~/.config/axium/spokes/aws/aws_spoke.py

def register(app, events):
    """
    Register Spoke commands and event handlers.

    Args:
        app: Typer application instance
        events: EventBus instance
    """
    @app.command("aws-whoami")
    def aws_whoami():
        """Show current AWS identity."""
        import boto3
        sts = boto3.client("sts")
        identity = sts.get_caller_identity()
        print(f"Account: {identity['Account']}")
        print(f"User: {identity['Arn']}")
```

Now you can run:
```bash
axium aws-whoami
```

---

## Event System

Spokes can react to events using the EventBus:

```python
def register(app, events):
    # React to environment changes
    def on_env_change(new_env, old_env):
        print(f"Environment changed: {old_env} → {new_env}")
        # Refresh credentials, update state, etc.

    events.on("env_change", on_env_change)

    # React to Spoke loading
    def on_spoke_loaded(spoke_name):
        if spoke_name == "aws":
            print("AWS Spoke loaded!")

    events.on("spoke_loaded", on_spoke_loaded)
```

### Available Events

- `env_change(new_env, old_env)` - Environment switched
- `spoke_loaded(spoke_name)` - Spoke finished loading

---

## Accessing Environment

Import the `env` module to access environment properties:

```python
from axium.core import env

def register(app, events):
    @app.command("show-region")
    def show_region():
        """Show current region from environment."""
        region = env.get_env_value("region")
        env_name = env.get_active_env_name()
        print(f"Environment: {env_name}")
        print(f"Region: {region}")
```

---

## Example Spokes

### AWS Profile Manager

```python
# aws_spoke.py
from axium.core import env
import os

def register(app, events):
    def on_env_change(new_env, old_env):
        """Update AWS_PROFILE when environment changes."""
        envs = env.load_envs()
        env_data = envs.get(new_env, {})
        aws_profile = env_data.get("aws_profile")

        if aws_profile:
            os.environ["AWS_PROFILE"] = aws_profile
            print(f"AWS_PROFILE → {aws_profile}")

    events.on("env_change", on_env_change)

    @app.command("aws-whoami")
    def aws_whoami():
        """Show current AWS identity."""
        import boto3
        sts = boto3.client("sts")
        identity = sts.get_caller_identity()
        print(f"Account: {identity['Account']}")
        print(f"ARN: {identity['Arn']}")
```

### Kubernetes Context Switcher

```python
# k8s_spoke.py
from axium.core import env
import subprocess

def register(app, events):
    def on_env_change(new_env, old_env):
        """Switch kubectl context when environment changes."""
        context = env.get_env_value("k8s_context")
        if context:
            subprocess.run(["kubectl", "config", "use-context", context])
            print(f"kubectl context → {context}")

    events.on("env_change", on_env_change)

    @app.command("k8s-pods")
    def k8s_pods():
        """List pods in current context."""
        result = subprocess.run(
            ["kubectl", "get", "pods"],
            capture_output=True,
            text=True
        )
        print(result.stdout)
```

---

## Configuration

Spokes should use the centralized config system instead of implementing their own config loading. See [Configuration](config.md) for details.

### Loading Config

```python
from axium.core.config import load_spoke_config

def register(app, events):
    # Load spoke configuration (env-aware)
    config = load_spoke_config("creds", "creds.yaml", env_aware=True)

    # Access values (already expanded)
    timeout = config["timeout"]
    path = config["path"]  # Tilde and vars already expanded
```

### Config File Structure

Create `<spoke-name>.yaml` in your spoke directory:

```yaml
# ~/.config/axium/spokes/creds/creds.yaml
default:
  timeout: 30
  check:
    type: mtime
    path: ~/.aws/credentials

prod:
  timeout: 60
  check:
    type: command
    command: aws sts get-caller-identity
```

Users can override values without modifying your spoke:

```bash
$ axium config edit creds
# Creates ~/.config/axium/overrides/creds.yaml
```

See [Configuration](config.md) for complete documentation.

---

## Permissions

Spokes must declare required permissions in `spoke.yaml`. All privileged operations are enforced by the daemon. See [Permissions](permissions.md) for details.

### Requesting Background Execution

```python
from axium.core.ipc import send_request_sync

def refresh_credentials():
    """Refresh credentials via daemon (requires exec permission)."""
    resp = send_request_sync({
        "cmd": "daemon_exec",
        "spoke": "creds",
        "command": "aws sso login --profile prod",
        "mode": "background"
    })
    return resp.get("ok", False)
```

### Sending Notifications

```python
def notify_user(title: str, message: str):
    """Send notification (requires notify permission)."""
    try:
        send_request_sync({
            "cmd": "notify",
            "spoke": "creds",
            "title": title,
            "body": message,
            "level": "info"
        })
    except Exception:
        pass  # Non-fatal
```

### Updating HUD

```python
def update_hud(is_valid: bool):
    """Update HUD segment (no special permission required)."""
    value = "[creds:Y]" if is_valid else "[creds:N]"
    try:
        send_request_sync({
            "cmd": "hud_segment_value",
            "spoke": "creds",
            "value": value
        })
    except Exception:
        pass
```

---

## Best Practices

1. **Command Naming**: Prefix commands with spoke name (e.g., `aws-whoami`, `k8s-pods`)
2. **Configuration**: Use `load_spoke_config()` instead of custom config loading
3. **Permissions**: Request minimal permissions needed, fail gracefully when denied
4. **Graceful Degradation**: Handle missing dependencies and daemon unavailability
5. **Event Handlers**: Keep handlers fast — don't block the event loop
6. **Environment Access**: Use `env.get_env_value()` instead of reading files directly
7. **Documentation**: Add docstrings to all commands for `axium --help`

---

## Testing Spokes

Create a test file in your Spoke directory:

```python
# test_aws_spoke.py
import pytest
from aws_spoke import register

def test_register():
    """Test Spoke registration."""
    from typer import Typer
    from axium.core.spokes import EventBus

    app = Typer()
    events = EventBus()

    register(app, events)

    # Verify command registered
    assert any(cmd.name == "aws-whoami" for cmd in app.registered_commands)
```

Run tests:
```bash
pytest ~/.config/axium/spokes/aws/
```

---

## Distributing Spokes

### As Git Repositories

```bash
cd ~/.config/axium/spokes/
git clone https://github.com/your-org/axium-spoke-aws.git aws
axium daemon reload
```

### As Python Packages

```bash
pip install axium-spoke-aws
# Package installs to ~/.config/axium/spokes/aws/
axium daemon reload
```

---

## See Also

- [Configuration](config.md) - Centralized config system
- [Permissions](permissions.md) - Permission system and security
- [Writing Spokes Guide](../guides/writing-spokes.md) - Detailed tutorial
- [API Reference](../reference/api.md) - Complete API documentation
- [Examples](../examples/spokes/) - Example Spoke implementations
