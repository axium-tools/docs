# ⚙️ Axium

_Axium: structure for your terminal._

Axium is a context-aware DevOps assistant built to make your shell environments cohesive and intelligent.
It provides a lightweight daemon, intuitive CLI, dynamic HUD, and extensible plugin system ("Spokes") — all designed for engineers who live in tmux and terminals.

---

## 🧭 Features

| Component       | Description                                                                          |
| --------------- | ------------------------------------------------------------------------------------ |
| Daemon (axiumd) | Async background process maintaining session state, env context, and Spokes registry |
| CLI (axium)     | Typer-based command interface with structured sub-commands                           |
| HUD             | Prints a one-line status segment ideal for tmux/shell prompts                        |
| Palette         | Curses-based quick launcher; triggers common commands or Spoke actions interactively |
| Spokes          | Modular plugin system extending Axium without touching the core                      |

---

## 🧱 Architecture Overview

Shell → CLI → Daemon → State

Each layer communicates via JSON over a UNIX socket:

```
Shell (bash/zsh/tmux)
    ↓
Axium CLI (Typer)
    ↓
Axium Daemon (asyncio)
    ↓
State / Config / Spokes
```

---

## 🧩 Directory Structure

```
axium/core/
    cli.py        → Typer CLI entrypoint
    daemon.py     → Async background service
    ipc.py        → JSON IPC utilities
    hud.py        → Status line generator
    palette.py    → Interactive TUI launcher
    spokes.py     → Spoke discovery/loader
bash/init.sh      → Shell alias + integration
tests/            → Pytest suite
```

---

## 🪩 Design Principles

- **Async-first** — everything that can await, does
- **Minimal dependencies** — stdlib + Typer + YAML
- **Predictable UX** — axium `<noun>` `<verb>` pattern
- **Invisible until needed** — helps, never interrupts
- **Composable** — Daemon, HUD, and Spokes work independently

---

## Quick Links

- [Getting Started](getting-started.md) - Installation and quickstart
- [Core Concepts](concepts/environments.md) - Understanding Axium's architecture
- [Writing Spokes](guides/writing-spokes.md) - Extending Axium with plugins
- [Contributing](development/CONTRIBUTING.md) - Join the development
- [Roadmap](development/ROADMAP.md) - Upcoming features

---

_"Structure for your terminal."_
