# 🧭 Axium Roadmap

_Axium: structure for your terminal._

This document outlines the phases of Axium's development — from a stable local assistant to a fully modular, extensible terminal ecosystem.

## 📊 Current Status

- **Phase 1:** ✅ Complete (Core Foundation)
- **Phase 2:** ✅ Complete (Dynamic Environment Layer)
- **Phase 3:** 🚧 In Progress (Spokes & Plugin System - core infrastructure complete)
- **Test Coverage:** 48 tests passing (Phase 1: 11, Phase 2: +29, Phase 3: +8)
- **Latest Commit:** `2150f73` - Self-bootstrapping initialization system (Jan 2025)

---

## ✅ Phase 1 — Core Foundation (complete)

**Goal:** make Axium a reliable daily driver.

### ✅ Delivered Features

- Async daemon (axiumd) with UNIX socket IPC
- Typer-based CLI with structured sub-commands
- HUD (status line for tmux/shell)
- Palette (curses command launcher)
- Config directory: ~/.config/axium/
- Bash init integration with prefix aliases
- Unit tests & GitHub Actions CI (starting with 11 tests)
- **Persistent state in ~/.config/axium/state.json**
- **Reload IPC command (`axium daemon reload`)**
- **`axium daemon logs` command for easy log tailing (-f, -n options)**
- **Structured logging (INFO/DEBUG/WARN) with AXIUM_DEBUG support**
- **Type hints, black/isort formatting, clean linting**
- **Consistent tagline: "Structure for your terminal"**

**Completion:** Commit `dbfa703` (Jan 2025)

---

## ✅ Phase 2 — Dynamic Environment Layer (complete)

**Goal:** Axium becomes the orchestrator for environment context.

### ✅ Delivered Features

- **Environment abstraction layer** (`axium/core/env.py`)
  - Flexible YAML-based configuration (`envs.yaml`)
  - Arbitrary environment properties (prefix, aws_profile, region, color, etc.)
  - Template system with `${env.<key>}` variable expansion
  - Functions: `load_envs()`, `get_active_env_name()`, `get_env_value()`
- **Command prefix system** (`axium/core/prefix.py`)
  - Dynamic command wrapping via `axium run`
  - Prefix configuration (`prefixes.yaml`) with template support
  - Transparent command prefixing (e.g., `aws s3 ls` → `enva-root aws s3 ls`)
  - Graceful fallback when daemon unavailable
- **Environment management commands**
  - `axium env list` - List all environments with properties
  - `axium env show [name]` - Display detailed environment properties
  - `axium env set/get` - Manage active environment
- **Shell integration** (bash/init.sh)
  - Upgraded to full environment bootstrapper
  - Auto-start daemon (opt-out via `AXIUM_NO_AUTOSTART`)
  - Dynamic wrapper function generation from prefixes.yaml
  - Tmux pane context tracking via `TMUX_PANE`
- **Self-bootstrapping initialization**
  - Automatic config directory creation (`~/.config/axium/`)
  - Default `envs.yaml`, `prefixes.yaml`, `state.json` on first run
  - Never overwrites existing user configuration
  - Graceful permission error handling
- **Test coverage:** +29 tests (env: 14, prefix: 11, bootstrap: 8, updates: -4)

**Completion:** Commits `e636450`, `23224ef`, `2150f73` (Jan 2025)


---

## 🚧 Phase 3 — Spokes & Plugin System (current)

**Goal:** make Axium extensible.

### ✅ Delivered Core Infrastructure

- **Spoke architecture** (`axium/core/spokes.py`)
  - EventBus for Spoke coordination
  - `register(app, events)` Spoke signature
  - Spoke loader system with automatic discovery
  - `~/.config/axium/spokes/` directory structure
  - Spoke manifest format (spoke.yaml with entrypoint)
- **Event system**
  - `env_change` event (emitted when environment switches)
  - `spoke_loaded` event (emitted after Spoke registration)
  - Graceful error handling for failing event listeners
- **Commands**
  - `axium spoke-list` - List installed Spokes
- **Documentation**
  - SPOKES.md - Complete Spoke architecture and contribution guide
  - STYLE-GUIDE.md - Official branding and design guidelines
  - Updated CLAUDE.md with Spoke development patterns
- **Test coverage:** +8 tests (spokes: 4, events: 4)

**Completion:** Commits `f144801`, `29f9c5c` (Jan 2025)

### Spoke Model

Each Spoke lives under `~/.config/axium/spokes/<name>/` and registers commands dynamically.

Example spoke.yaml:
```yaml
name: aws
entrypoint: aws_spoke:register
```

Example register function:
```python
def register(app, events):
    @app.command("aws-whoami")
    def aws_whoami():
        # Command implementation
        pass

    events.on("env_change", lambda new, old: ...)
```

### 📋 Remaining Work

**Example Spokes to build:**
- axium-spoke-aws: quick EC2/SSM/STS commands
- axium-spoke-ansible: runbook & inventory helpers
- axium-spoke-db: common Postgres/Redis tooling
- axium-spoke-system: local system metrics & shortcuts

**Core enhancements:**
- Hot-reload command (`axium spoke reload`)
- Palette actions auto-loaded from Spokes
- Registry sync (registry.axium.tools/spokes.yaml)

---

## 🖥️ Phase 4 — UI & Interaction Enhancements

**Goal:** richer, interactive Axium without leaving tmux.

### Ideas

- Split-pane HUD widgets (real-time logs, system summary)
- Interactive dashboards: minimal curses/TUI windows
- Command Palette: fuzzy search across all Spokes + history
- Notifications: lightweight toast banners in tmux/status line
- Color theming: consistent Axium teal (#00B7C7) + cyan highlights

---

## 🧠 Phase 5 — Intelligence & Automation

**Goal:** let Axium proactively assist.

### Planned

- Fuzzy command correction / “Did you mean…”
- Command history & replay (axium replay `<session>`)
- Optional background task runner (lightweight cron)
- Daemon metrics endpoint (axium daemon stats)
- Optional Cloud sync of state/config (private)

---

## 🌐 Phase 6 — Ecosystem & Distribution

**Goal:** Axium becomes a small but complete platform.

### Deliverables

- Public Spoke registry (https://registry.axium.tools)
- Install script (https://install.axium.tools)
- Docs site (https://docs.axium.tools)
- Versioned releases & changelogs
- Signed binary builds (axiumd, ax)

---

## 💡 Future Ideas (long-term / experimental)

| Idea                 | Description                                        |
| -------------------- | -------------------------------------------------- |
| SSH Integration      | Wrap ssh to auto-load Axium env context remotely   |
| Remote daemon bridge | Forward IPC over SSH to interact with remote Axium |
| Web dashboard        | Optional GUI viewer for Spokes/HUD metrics         |
| Plugin sandboxing    | Restricted execution environment for Spokes        |
| Community Spoke API  | Register external Spokes via registry.axium.tools  |

---

## 🪩 Development Principles

- Minimal dependencies — prefer stdlib + Typer + YAML
- Async first — all long-running ops via asyncio
- Testable — every command and IPC endpoint unit-tested
- Composable — Spokes, HUD, and Palette never tightly coupled
- Predictable UX — consistent command structure (axium `<noun>` `<verb>`)
- Invisible until needed — Axium should stay out of your way

---

_“Structure for your terminal.”_
