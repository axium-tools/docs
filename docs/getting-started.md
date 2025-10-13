# Getting Started

Get up and running with Axium in minutes.

---

## Installation

Install in editable mode:

```bash
pip install -e .
```

---

## Shell Integration

Load Axium shell integration (adds the `ax` alias):

```bash
source bash/init.sh
```

Add this to your `.bashrc` or `.zshrc` to load automatically:

```bash
# Axium shell integration
[ -f ~/path/to/axium/bash/init.sh ] && source ~/path/to/axium/bash/init.sh
```

---

## First Steps

### 1. Start the Daemon

```bash
axium daemon start
```

The daemon runs in the background and maintains your session state.

### 2. Set an Environment

```bash
axium env set prod
```

Axium uses environments to manage different contexts (dev, staging, prod, etc.).

### 3. View the HUD

```bash
axium hud
```

Output:
```
[axium] env:prod aws:- uptime:5m
```

### 4. Launch the Palette

```bash
axium palette
```

The interactive palette provides quick access to common commands and Spoke actions.

---

## Configuration

Axium automatically creates its configuration directory at `~/.config/axium/` on first run.

Contents:

- `axiumd.log` — daemon log file
- `state.json` — daemon state (env, aws profile, etc.)
- `spokes/` — installed plugin folders
- `envs.yaml` — environment definitions
- `prefixes.yaml` — command prefix mappings

---

## Environment Variables

- `AXIUM_DEBUG=1` — enables verbose debug logs
- `AXIUM_HOME` — override default config directory

---

## Example Workflows

### Switch Environment

```bash
axium env set builder
axium hud
```

Output:
```
[axium] env:builder aws:- uptime:5m
```

### Reload Daemon

Reload daemon configuration without restart:

```bash
axium daemon reload
```

### View Logs

```bash
axium daemon logs -f
```

---

## Development Setup

Requirements:

- Python ≥ 3.9
- Typer
- PyYAML
- pytest

Setup:

```bash
git clone https://github.com/axium-tools/axium.git
cd axium
pip install -e .[dev]
pytest -q
```

---

## Next Steps

- [Understanding Environments](concepts/environments.md)
- [Command Prefixes](concepts/prefixes.md)
- [Writing Your First Spoke](guides/writing-spokes.md)
- [Configuration Reference](reference/configuration.md)
