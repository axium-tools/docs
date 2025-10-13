# Permission System

Axium provides a declarative permission system for Spokes with daemon-enforced security and user-configurable overrides.

## Overview

Spokes declare required permissions in their `spoke.yaml` manifest. All privileged operations (exec, notify, filesystem access) are enforced by the daemon via IPC. Users can extend or restrict permissions without modifying spoke code.

### Key Features

- **Declarative Permissions** - Spokes declare needs in `spoke.yaml`
- **Daemon Enforcement** - All privileged actions go through daemon IPC
- **User Overrides** - Users can extend/restrict via `~/.config/axium/permissions.yaml`
- **Glob Matching** - Filesystem permissions support wildcards
- **Security Logging** - All permission checks logged with `[SECURITY]` prefix
- **Strict Mode** - `AXIUM_PERMS_STRICT=1` disables all overrides

## Permission Types

### Boolean Permissions

- **`exec`** - Allow background command execution via `daemon_exec`
- **`notify`** - Allow user notifications
- **`net`** - (Reserved) Future HTTP request capability

### List Permissions

- **`fs_read`** - List of allowed readable paths (glob patterns)
- **`fs_write`** - List of allowed writable paths (glob patterns)

## Declaring Permissions

### In spoke.yaml

```yaml
name: creds
version: 0.1.0
description: Credential validity checker
entrypoint: main:register
permissions:
  exec: true                # Allow background commands
  notify: true              # Allow notifications
  net: false                # No network access
  fs_read:
    - ~/.aws/credentials    # Can read AWS creds
    - ~/.aws/config
    - /opt/company/creds/*  # Wildcard pattern
  fs_write:
    - /tmp/creds-cache/*    # Can write to cache
```

### Default Permissions

If no `permissions` section exists, all permissions default to `false` and empty lists (most restrictive).

## User Overrides

Users can customize spoke permissions via `~/.config/axium/permissions.yaml`:

```yaml
# User permission overrides
creds:
  exec: false               # Disable exec even if spoke requests it
  fs_read:
    - ~/.aws/credentials
    - /extra/path/*         # Add additional path

aws-spoke:
  notify: true              # Enable notifications
  fs_write:
    - ~/aws-logs/*          # Add write permission
```

### Merge Rules

- **Booleans** (exec, notify, net): Override replaces base value
- **Lists** (fs_read, fs_write): Union of base + override (deduplicated)

**Example:**

**Base (spoke.yaml):**
```yaml
permissions:
  exec: true
  fs_read:
    - ~/.aws/credentials
```

**Override (permissions.yaml):**
```yaml
creds:
  exec: false
  fs_read:
    - /opt/creds.json
```

**Result:**
```yaml
exec: false                    # Overridden
fs_read:
  - ~/.aws/credentials         # From base
  - /opt/creds.json            # From override
```

## Using Permissions in Spokes

### Sending Notifications

```python
from axium.core.ipc import send_request_sync

def send_notification(title: str, body: str):
    """Send notification via daemon (requires notify permission)."""
    try:
        resp = send_request_sync({
            "cmd": "notify",
            "spoke": "creds",
            "title": title,
            "body": body,
            "level": "info"
        })
        return resp.get("ok", False)
    except Exception:
        return False
```

### Requesting Background Execution

```python
def request_daemon_exec(command: str) -> bool:
    """Request daemon to run command in background (requires exec permission)."""
    try:
        resp = send_request_sync({
            "cmd": "daemon_exec",
            "spoke": "creds",
            "command": command,
            "mode": "background"
        })
        return resp.get("ok", False)
    except Exception:
        return False

# Usage
if request_daemon_exec("aws sso login --profile prod"):
    print("✓ Refresh started in background")
```

### Updating HUD Segments

```python
def update_hud_segment(value: str):
    """Update spoke's HUD segment (no special permission needed)."""
    try:
        send_request_sync({
            "cmd": "hud_segment_value",
            "spoke": "creds",
            "value": value  # e.g., "[creds:Y]" or ""
        })
    except Exception:
        pass  # Non-fatal
```

## Filesystem Permissions

Filesystem permissions use **glob pattern matching** with the `fnmatch` module (shell-style wildcards).

### Pattern Examples

```yaml
fs_read:
  - ~/.aws/credentials              # Exact file
  - ~/.aws/*                        # All files in dir (not recursive)
  - ~/.config/axium/**              # Recursive (all files in tree)
  - /tmp/spoke-*.log                # Pattern matching
  - ~/data/*.{json,yaml}            # Multiple extensions (if shell expanded)
```

### Path Expansion

Paths are automatically expanded:
- Tilde (`~`) → User's home directory
- Environment variables work in override files

### Checking Filesystem Permissions

Daemon automatically checks filesystem permissions when spokes use config loading:

```python
# Config system automatically checks fs_read permissions
config = load_spoke_config("creds", "creds.yaml")
# If creds spoke doesn't have fs_read permission for config file, loading fails
```

## CLI Commands

### View Permissions

```bash
# List all spoke permissions
$ axium perms list
Spoke Permissions:

creds (v0.1.0)
  exec: true (base)
  notify: true (base)
  net: false (base)
  fs_read: 2 paths
  fs_write: 1 path

aws (v1.0.0)
  exec: false (base)
  notify: true (override)
  ...
```

### Show Detailed Permissions

```bash
$ axium perms show creds
Permissions for 'creds' (v0.1.0):

Boolean Permissions:
  exec: true (base)
  notify: true (base)
  net: false (base)

Filesystem Permissions:
  fs_read:
    • ~/.aws/credentials (base)
    • ~/.aws/config (base)
    • /opt/company/creds/* (override)

  fs_write:
    • /tmp/creds-cache/* (base)

Override source: ~/.config/axium/permissions.yaml
```

### Edit Overrides

```bash
$ axium perms edit creds
# Opens ~/.config/axium/permissions.yaml in $EDITOR
# Creates file if it doesn't exist
```

## Notification Management

### Drain Notifications

```bash
$ axium notify drain
Queued Notifications (2):

[2025-10-07 16:30:00] creds: Credentials expired
  Run: aws sso login or 'axium creds refresh'

[2025-10-07 16:31:15] system: Update available
  Run: axium update
```

### Test Notification

```bash
$ axium notify send --title "Test" --body "Hello world"
✓ Notification sent

View with: axium notify drain
```

## Security Logging

All permission checks are logged with `[SECURITY]` prefix for audit trails:

```
[SECURITY] spoke=creds action=exec allowed=true detail=None via_override=false
[SECURITY] spoke=creds action=notify allowed=true detail=None via_override=false
[SECURITY] spoke=aws action=fs_read:/etc/passwd allowed=false detail=/etc/passwd via_override=false
```

### Log Format

```
[SECURITY] spoke=<name> action=<action> allowed=<bool> detail=<info> via_override=<bool>
```

- **spoke**: Spoke name
- **action**: Permission type (exec, notify, fs_read:path, etc.)
- **allowed**: true if permitted, false if denied
- **detail**: Additional context (command, path, etc.)
- **via_override**: true if permission came from user override

## Strict Mode

Strict mode disables all user overrides, using only spoke-declared permissions. Useful for CI/CD, locked-down hosts, or security compliance.

### Enable Strict Mode

```bash
export AXIUM_PERMS_STRICT=1
axium daemon restart
```

In strict mode:
- All `~/.config/axium/permissions.yaml` overrides ignored
- Only permissions in `spoke.yaml` are active
- Logged: `"Strict mode enabled, ignoring overrides for spoke: <name>"`

## Best Practices

### For Spoke Developers

✅ **Request minimal permissions**
```yaml
# Good - only what you need
permissions:
  notify: true
  fs_read:
    - ~/.aws/credentials
```

✅ **Document why you need permissions**
```yaml
# Good - explain in spoke README
permissions:
  exec: true  # Required to refresh credentials via 'aws sso login'
```

✅ **Fail gracefully when denied**
```python
# Good - handle permission denial
if not request_daemon_exec(command):
    logger.warning("Could not refresh credentials (exec permission denied)")
    # Continue with cached data or manual prompt
```

❌ **Don't request blanket permissions**
```yaml
# Bad - too broad
permissions:
  fs_read:
    - ~/**           # Everything in home dir!
    - /etc/**        # All system configs!
```

❌ **Don't hardcode actions without permission checks**
```python
# Bad - bypasses permission system
import subprocess
subprocess.run(["aws", "sso", "login"])  # Should use daemon_exec
```

### For Users

✅ **Use overrides to grant additional access**
```yaml
# Good - allow spoke to access extra credential files
creds:
  fs_read:
    - /opt/company/special-creds.json
```

✅ **Use strict mode in production**
```bash
# Good - no user overrides in CI
export AXIUM_PERMS_STRICT=1
```

❌ **Don't grant excessive permissions**
```yaml
# Bad - defeats security model
aws-spoke:
  exec: true
  fs_write:
    - ~/**  # Can write anywhere!
```

## Architecture

### Permission Flow

1. **Spoke declares permissions** in `spoke.yaml`
2. **Daemon loads permissions** on spoke load
3. **User overrides merged** (unless strict mode)
4. **Spoke requests action** via IPC
5. **Daemon checks permission** before executing
6. **Action allowed/denied** with security log
7. **Response sent** to spoke

### IPC Handlers

- `load_spoke_permissions` - Load/merge permissions on spoke load
- `notify` - Check notify permission, queue notification
- `daemon_exec` - Check exec permission, run background command
- `get_permissions` - Return effective permissions for spoke
- `notify_drain` - Return and clear notification queue

## Example: Complete Workflow

### 1. Spoke Declares Permissions

`spokes/creds/spoke.yaml`:
```yaml
name: creds
permissions:
  exec: true
  notify: true
  fs_read:
    - ~/.aws/credentials
```

### 2. User Adds Override

```bash
$ axium perms edit creds
```

`~/.config/axium/permissions.yaml`:
```yaml
creds:
  exec: false  # Disable auto-refresh
  fs_read:
    - ~/extra-creds.json
```

### 3. Daemon Loads Merged Permissions

```python
# Result:
{
  "exec": false,                  # From override
  "notify": true,                 # From base
  "fs_read": [
    "~/.aws/credentials",         # From base
    "~/extra-creds.json"          # From override
  ]
}
```

### 4. Spoke Requests Action

```python
# Spoke tries to refresh
result = request_daemon_exec("aws sso login")
# Returns: False (exec denied by override)
```

### 5. Security Log

```
[SECURITY] spoke=creds action=exec allowed=false detail=aws sso login via_override=true
```

## See Also

- [Configuration](config.md) - Config system documentation
- [Spokes](spokes.md) - Spoke development guide
- [Security](../development/SECURITY.md) - Security considerations (if exists)
