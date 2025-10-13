# Environments

Axium's environment system provides flexible, YAML-based context management for your terminal sessions.

---

## Overview

Environments allow you to define arbitrary properties and switch between different contexts seamlessly. Each environment can have any properties you need — prefix, AWS profile, region, color, etc.

---

## Configuration

Environments are defined in `~/.config/axium/envs.yaml`:

```yaml
envs:
  root:
    prefix: enva-root
    aws_profile: root
    color: teal
    region: eu-west-1

  builder:
    prefix: enva-builder
    aws_profile: builder
    color: cyan
    region: eu-west-2

  prod:
    prefix: enva-prod
    aws_profile: production
    color: red
    region: us-east-1
```

---

## Template Variables

Access environment properties using `${env.<key>}` syntax in configuration files:

```yaml
# prefixes.yaml
prefixes:
  - command: aws
    prefix: ${env.prefix}

  - command: terraform
    prefix: ${env.prefix}
```

When you run `aws s3 ls`, Axium automatically expands to:
```bash
enva-root aws s3 ls  # if active env is 'root'
```

---

## CLI Commands

### List Environments

```bash
axium env list
```

Output:
```
Available environments:
  * root (prefix, aws_profile, color, region)
    builder (prefix, aws_profile, color, region)
    prod (prefix, aws_profile, color, region)
```

The `*` indicates the active environment.

### Show Environment

Show properties for a specific environment:

```bash
axium env show builder
```

Output:
```
Environment: builder
  prefix: enva-builder
  aws_profile: builder
  color: cyan
  region: eu-west-2
```

Show active environment (no argument):

```bash
axium env show
```

### Set Environment

Switch to a different environment:

```bash
axium env set prod
```

This updates the active environment and triggers the `env_change` event for Spokes to react.

### Get Property

Get a specific property from the active environment:

```bash
axium env get region
```

Output:
```
eu-west-1
```

---

## Python API

### In Spokes

Import the `env` module directly:

```python
from axium.core import env

def my_command():
    # Get active environment name
    env_name = env.get_active_env_name()
    print(f"Active environment: {env_name}")

    # Get specific property
    region = env.get_env_value("region")
    print(f"Region: {region}")

    # Get all environment properties
    env_data = env.get_active_env()
    print(f"AWS Profile: {env_data.get('aws_profile')}")
```

### Core Functions

```python
from axium.core import env

# Load all environments from envs.yaml
envs = env.load_envs()

# Get active environment name
active_name = env.get_active_env_name()

# Get all properties for active environment
active_env = env.get_active_env()

# Get specific property from active environment
value = env.get_env_value("prefix")
```

---

## Event System

When the environment changes, Axium emits an `env_change` event:

```python
def register(app, events):
    def on_env_change(new_env, old_env):
        print(f"Environment changed: {old_env} → {new_env}")

    events.on("env_change", on_env_change)
```

This allows Spokes to react to environment changes (e.g., refresh AWS credentials, update UI).

---

## Best Practices

1. **Arbitrary Properties**: Define any properties your workflow needs — Axium doesn't enforce a schema
2. **Consistent Naming**: Use consistent property names across environments for predictable template expansion
3. **Color Coding**: Use the `color` property for visual distinction in HUD/palette
4. **Prefix Convention**: Use prefixes to namespace commands (e.g., `enva-prod`, `enva-dev`)

---

## Example Use Cases

### AWS Multi-Account Management

```yaml
envs:
  dev:
    prefix: enva-dev
    aws_profile: dev-account
    aws_account_id: "123456789012"
    region: eu-west-1

  prod:
    prefix: enva-prod
    aws_profile: prod-account
    aws_account_id: "987654321098"
    region: us-east-1
```

### Regional Deployments

```yaml
envs:
  eu:
    prefix: deploy-eu
    region: eu-west-1
    cluster: eu-prod-cluster

  us:
    prefix: deploy-us
    region: us-east-1
    cluster: us-prod-cluster
```

### Team Environments

```yaml
envs:
  alice:
    prefix: dev-alice
    namespace: alice-dev

  bob:
    prefix: dev-bob
    namespace: bob-dev
```

---

## See Also

- [Command Prefixes](prefixes.md) - Using `${env.<key>}` in prefix configuration
- [Writing Spokes](../guides/writing-spokes.md) - Reacting to environment changes
- [Configuration Reference](../reference/configuration.md) - Complete envs.yaml schema
