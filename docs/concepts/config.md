# Configuration System

Axium provides a centralized configuration system for Spokes with support for user overrides, environment-specific settings, and automatic value expansion.

## Overview

The config system separates **base configuration** (bundled with spokes) from **user overrides** (in `~/.config/axium/overrides/`), allowing users to customize spoke behavior without modifying the spoke itself.

### Key Features

- **Centralized Loading** - All spokes use `load_spoke_config()` for uniform config handling
- **Deep Merging** - Override values merge recursively with base config
- **Path Expansion** - Automatic expansion of `~`, `$VAR`, and `${env.key}`
- **Environment Awareness** - Config sections can vary by active environment
- **Config Caching** - Efficient caching with automatic invalidation
- **Override Pattern** - Small focused overrides, not full config copies

## File Locations

```
~/.config/axium/
├── spokes/
│   └── <spoke-name>/
│       └── <spoke-name>.yaml    # Base config (bundled with spoke)
└── overrides/
    └── <spoke-name>.yaml        # User overrides (optional)
```

## Config Structure

### Base Config (Bundled with Spoke)

Located in `~/.config/axium/spokes/<spoke>/config.yaml`, this is the default configuration shipped with the spoke.

**Example:** `~/.config/axium/spokes/creds/creds.yaml`
```yaml
# Base configuration for creds spoke
default:
  check:
    type: mtime
    path: ~/.aws/credentials
    max_age: 86400  # 24 hours
  refresh:
    command: aws sso login
  auto_refresh: false

prod:
  check:
    type: command
    command: aws sts get-caller-identity --profile prod
    expect_output: Account
  refresh:
    command: aws sso login --profile prod
```

### Override Config (User Editable)

Located in `~/.config/axium/overrides/<spoke>.yaml`, this contains only the values you want to change.

**Example:** `~/.config/axium/overrides/creds.yaml`
```yaml
# Override only what you need
default:
  auto_refresh: true    # Enable auto-refresh
  check:
    max_age: 43200      # Change to 12 hours

staging:
  check:
    type: mtime
    path: ~/.aws/credentials-staging
```

**Important:** Do NOT copy the entire base config. Only include values you want to override.

## Merge Behavior

The config system uses **deep merge** semantics:

- **Dictionaries**: Merge recursively
- **Scalars** (strings, numbers, booleans): Override replaces base
- **Lists**: Override replaces base (no union)

### Merge Examples

**Base:**
```yaml
default:
  timeout: 30
  retries: 3
  nested:
    a: 1
    b: 2
```

**Override:**
```yaml
default:
  timeout: 60
  nested:
    b: 20
    c: 3
```

**Result:**
```yaml
default:
  timeout: 60      # Overridden
  retries: 3       # From base
  nested:
    a: 1           # From base
    b: 20          # Overridden
    c: 3           # From override
```

## Environment-Aware Config

When `env_aware=True`, the config system merges sections based on the active environment:

1. Load base config
2. Load override config
3. Merge `default` section
4. Merge `<active_env>` section (if exists)
5. Expand variables

**Example with `env=prod`:**
```yaml
# Base
default:
  timeout: 30
prod:
  timeout: 60

# Result when env=prod:
{
  "timeout": 60,    # From prod section
  ...other values from default...
}
```

## Value Expansion

The config system automatically expands:

### Tilde Expansion
```yaml
path: ~/data/file.txt
# Expands to: /home/user/data/file.txt
```

### Environment Variables
```yaml
# Using $VAR
path: /opt/$USER/data

# Using ${VAR}
url: https://${REGION}.example.com
```

### Axium Environment Values
```yaml
# Reference values from active Axium environment
bucket: s3://${env.region}/data
account: ${env.account_id}
```

## CLI Commands

### View Config Paths
```bash
$ axium config path creds
Base config (bundled):
  ~/.config/axium/spokes/creds/creds.yaml ✓

Override config (user editable):
  ~/.config/axium/overrides/creds.yaml ✗ (not found)

To create override: axium config edit creds
```

### Show Merged Config

The `axium config show` command queries the daemon via IPC to display configuration. The daemon is the single source of truth for all config data, ensuring consistency and security.

```bash
# Show full merged config
$ axium config show creds
Merged Configuration (creds):
{
  "check": {
    "type": "mtime",
    "path": "/home/user/.aws/credentials",
    "max_age": 86400
  },
  ...
}

# Show specific key path
$ axium config show creds --key check.path
Merged Configuration (creds, key: check.path):
/home/user/.aws/credentials

# Redact sensitive values (passwords, tokens, keys)
$ axium config show creds --redact
Merged Configuration (creds):
{
  "api_key": "***REDACTED***",
  "check": {
    "type": "mtime",
    "path": "/home/user/.aws/credentials"
  }
}
```

**Key path syntax:** Use dot notation to navigate nested config:
- `check.type` → `config["check"]["type"]`
- `nested.deep.value` → `config["nested"]["deep"]["value"]`

**Redaction patterns:** The `--redact` flag masks values for keys matching:
- password, passwd, pwd
- secret, token, key, api_key, apikey
- credential, cred, auth, authorization

### Edit Override Config
```bash
$ axium config edit creds
# Opens ~/.config/axium/overrides/creds.yaml in $EDITOR
# Creates file and template if it doesn't exist
```

## Using Config in Spokes

### Loading Config

```python
from axium.core.config import load_spoke_config

def register(app, events):
    # Load config (env-aware)
    config = load_spoke_config("creds", "creds.yaml", env_aware=True)

    # Access values
    check_type = config["check"]["type"]
    path = config["check"]["path"]  # Already expanded
```

### Querying Config via IPC (Recommended for Spokes)

Spokes can query config through the daemon instead of loading files directly:

```python
from axium.core.ipc import send_request_sync

# Get full config via daemon
resp = send_request_sync({
    "cmd": "get_config",
    "spoke": "creds"
})

if resp["ok"]:
    config = resp["config"]
    check_type = config["check"]["type"]

# Get specific key path
resp = send_request_sync({
    "cmd": "get_config",
    "spoke": "creds",
    "key": "check.path"
})

if resp["ok"]:
    path = resp["config"]  # Direct value
```

**Benefits of IPC approach:**
- Single source of truth (daemon)
- Automatic caching
- Security auditing
- Consistent with other spoke operations

### Getting Specific Values (Direct Access)

```python
from axium.core.config import get_spoke_config_value

# Get nested value with default (bypasses daemon)
max_age = get_spoke_config_value(
    "creds",
    "check.max_age",
    default=86400
)
```

### Config Paths Helper

```python
from axium.core.config import get_config_paths

paths = get_config_paths("creds", "creds.yaml")
if paths["override"]["exists"]:
    print(f"Override found: {paths['override']['path']}")
```

## Cache Management

Config is cached per `(spoke_name, env_name)` tuple for performance.

### Automatic Invalidation

Cache is automatically invalidated on:
- Spoke reload: `axium spoke reload <spoke>`
- Environment change: `axium env set <env>`
- Daemon restart: `axium daemon restart`

### Manual Invalidation

```python
from axium.core.config import invalidate_cache

# Invalidate specific spoke
invalidate_cache("creds")

# Invalidate all
invalidate_cache()
```

## Best Practices

### DO

✅ **Keep overrides minimal**
```yaml
# Good - only override what you need
default:
  auto_refresh: true
```

✅ **Use environment sections**
```yaml
default:
  timeout: 30
prod:
  timeout: 120
```

✅ **Document your overrides**
```yaml
# Increase timeout for slow network
default:
  timeout: 120
```

### DON'T

❌ **Don't copy entire base config**
```yaml
# Bad - this is a maintenance nightmare
default:
  check:
    type: mtime
    path: ~/.aws/credentials
    max_age: 86400
  refresh:
    command: aws sso login
  auto_refresh: false
  # ... 50 more lines ...
```

❌ **Don't hardcode paths**
```yaml
# Bad - not portable
path: /Users/jdoe/my-data

# Good - use tilde
path: ~/my-data
```

❌ **Don't duplicate logic**
```yaml
# Bad - logic should be in spoke code
default:
  if_condition: true
  then_value: x
  else_value: y
```

## Example: Complete Config Workflow

### 1. Spoke Ships with Base Config

`~/.config/axium/spokes/creds/creds.yaml`:
```yaml
default:
  check:
    type: mtime
    path: ~/.aws/credentials
    max_age: 86400
  auto_refresh: false
```

### 2. User Wants to Customize

```bash
$ axium config edit creds
# Opens editor with template
```

### 3. User Creates Minimal Override

`~/.config/axium/overrides/creds.yaml`:
```yaml
default:
  auto_refresh: true
  check:
    max_age: 43200  # 12 hours instead of 24
```

### 4. Spoke Loads Merged Config

```python
config = load_spoke_config("creds", "creds.yaml", env_aware=True)
# Result:
# {
#   "check": {
#     "type": "mtime",              # From base
#     "path": "~/.aws/credentials", # From base
#     "max_age": 43200              # From override
#   },
#   "auto_refresh": true            # From override
# }
```

### 5. User Inspects Result

```bash
$ axium config show creds
Merged Configuration (creds):
{
  "check": {
    "type": "mtime",
    "path": "/home/user/.aws/credentials",
    "max_age": 43200
  },
  "auto_refresh": true
}
```

## See Also

- [Spokes](spokes.md) - Spoke development guide
- [Environments](environments.md) - Environment management
- [Permissions](permissions.md) - Permission system (if you create this)
