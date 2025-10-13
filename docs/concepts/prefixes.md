# Command Prefixes

Axium's prefix system allows transparent command wrapping — intercepting commands and prepending context-aware prefixes before execution.

---

## Overview

Command prefixes enable workflows like:

```bash
aws s3 ls  # Automatically becomes: enva-root aws s3 ls
```

This is powerful for:
- AWS multi-account management (using `aws-vault` or similar)
- Terraform workspace switching
- Docker context switching
- Any tool that needs context injection

---

## Configuration

Prefixes are defined in `~/.config/axium/prefixes.yaml`:

```yaml
prefixes:
  - command: aws
    prefix: ${env.prefix}

  - command: terraform
    prefix: ${env.prefix}

  - command: kubectl
    prefix: "kubecontext-${env.cluster}"
```

---

## Template Expansion

Use `${env.<key>}` to reference environment properties:

```yaml
prefixes:
  - command: aws
    prefix: ${env.prefix}  # Expands to 'enva-root' if env.prefix = 'enva-root'

  - command: docker
    prefix: "DOCKER_HOST=${env.docker_host}"  # Expands to full env var
```

When you switch environments with `axium env set`, the prefix automatically updates.

---

## Shell Integration

Axium's shell integration (`bash/init.sh`) wraps configured commands automatically:

```bash
# In your shell
aws s3 ls

# Actually executes
enva-root aws s3 ls
```

The wrapping is transparent — you type normal commands, Axium handles the prefix injection.

---

## Example Workflows

### AWS Multi-Account

```yaml
# envs.yaml
envs:
  dev:
    prefix: aws-vault exec dev --
  prod:
    prefix: aws-vault exec prod --

# prefixes.yaml
prefixes:
  - command: aws
    prefix: ${env.prefix}
```

Now:
```bash
axium env set dev
aws s3 ls  # → aws-vault exec dev -- aws s3 ls

axium env set prod
aws s3 ls  # → aws-vault exec prod -- aws s3 ls
```

### Terraform Workspaces

```yaml
# envs.yaml
envs:
  staging:
    prefix: terraform-workspace-staging
  prod:
    prefix: terraform-workspace-prod

# prefixes.yaml
prefixes:
  - command: terraform
    prefix: ${env.prefix}
```

### Docker Contexts

```yaml
# envs.yaml
envs:
  local:
    docker_context: default
  remote:
    docker_context: remote-host

# prefixes.yaml
prefixes:
  - command: docker
    prefix: "docker --context ${env.docker_context}"
```

---

## Python API

### Expand Templates

```python
from axium.core.prefix import expand_env_vars

template = "${env.prefix}"
expanded = expand_env_vars(template)  # → "enva-root"

template = "Using ${env.prefix} in ${env.region}"
expanded = expand_env_vars(template)  # → "Using enva-root in eu-west-1"
```

### Load Prefixes

```python
from axium.core.prefix import load_prefixes

prefixes = load_prefixes()
# Returns: [{"command": "aws", "prefix": "enva-root"}, ...]
```

---

## Advanced Usage

### Conditional Prefixes

You can use different prefixes for different environments:

```yaml
# envs.yaml
envs:
  dev:
    prefix: ""  # No prefix for dev
  prod:
    prefix: aws-vault exec prod --

# prefixes.yaml
prefixes:
  - command: aws
    prefix: ${env.prefix}
```

In dev: `aws s3 ls` runs as-is
In prod: `aws s3 ls` → `aws-vault exec prod -- aws s3 ls`

### Multiple Commands

Wrap multiple commands with the same prefix:

```yaml
prefixes:
  - command: aws
    prefix: ${env.prefix}
  - command: terraform
    prefix: ${env.prefix}
  - command: kubectl
    prefix: ${env.prefix}
```

---

## Best Practices

1. **Use Templates**: Always use `${env.<key>}` instead of hardcoding values
2. **Test First**: Set up prefixes in dev environment before rolling to prod
3. **Keep it Simple**: Prefixes should be single commands or simple wrappers
4. **Document Assumptions**: If your prefix expects specific tools (e.g., `aws-vault`), document in envs.yaml comments

---

## Troubleshooting

### Prefix Not Expanding

Check that:
1. The environment property exists: `axium env show`
2. The template syntax is correct: `${env.key}` not `${env_key}`
3. The daemon is running: `axium daemon status`

### Command Not Being Wrapped

Verify:
1. Shell integration is loaded: `source bash/init.sh`
2. The command is in prefixes.yaml
3. The prefix expansion is valid: `axium env get prefix`

---

## See Also

- [Environments](environments.md) - Defining environment properties
- [Configuration Reference](../reference/configuration.md) - Complete prefixes.yaml schema
- [Examples](../examples/prefixes.yaml) - Real-world prefix configurations
