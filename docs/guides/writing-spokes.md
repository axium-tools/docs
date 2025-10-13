# 🧩 Axium Spokes: Architecture & Contribution Guide

_Axium’s Spokes are how we extend the platform — logic, not config._

---

## Overview

Spokes are **lightweight Python modules** that extend Axium’s behavior by registering **hooks, commands, and integrations**.
They never carry their own configuration files.
All persistent data lives in Axium’s core config (like `envs.yaml`, `prefixes.yaml`, `settings.yaml`).

This design keeps Axium declarative, deterministic, and reloadable at runtime.

---

## 🔧 Philosophy: “Config lives in core, logic lives in Spokes”

| Responsibility               | Lives in                  | Example                                          |
| ---------------------------- | ------------------------- | ------------------------------------------------ |
| Persistent configuration     | **Core**            | `~/.config/axium/envs.yaml`, `prefixes.yaml` |
| Runtime logic & integrations | **Spokes**          | Hooks, commands, event listeners                 |
| Declarative state            | **Daemon**          | `state.json` (active env, status)              |
| Execution context            | **Command wrapper** | `axium run <cmd>`                              |

---

## 🧠 What a Spoke *is*

A Spoke is just a Python file (or package) that exports a single `register()` function.

    def register(app, events):
        # Add CLI commands
        @app.command("aws-whoami")
        def aws_whoami():
            import boto3
            print(boto3.client("sts").get_caller_identity())

        # Subscribe to events
        def on_env_change(new_env):
            print(f"[spoke:aws] environment changed → {new_env}")
        events.on("env_change", on_env_change)

        # Access environment data
        from axium.core import env
        current_env = env.get_active_env_name()
        prefix = env.get_env_value("prefix")

That's it.
No YAML, no hidden state, no user configuration.

---

## ⚙️ What Spokes Can Do

| Capability                           | Description                                    | Example                                                |
| ------------------------------------ | ---------------------------------------------- | ------------------------------------------------------ |
| **Register CLI commands**      | Add subcommands to `axium`                   | `axium aws-whoami`                                   |
| **Hook into events**           | React to daemon or CLI lifecycle               | `on_env_change`, `on_prefix_apply`                 |
| **Extend prefix logic**        | Register new prefix rules dynamically          | `register_prefix("aws", prefix="enva-${axium_env}")` |
| **Add HUD / Palette elements** | (Future) Display widgets or menu items         | `register_hud_widget("aws-status")`                  |
| **Run background tasks**       | Schedule or poll data quietly via daemon hooks | Example: refresh EC2 list every 60s                    |

---

## 🪝 Hook System

Axium provides an in-memory event bus for runtime extensibility.

| Hook                              | When it triggers              | Common use                      |
| --------------------------------- | ----------------------------- | ------------------------------- |
| `on_env_change(new_env)`        | After `axium env set`       | Update context, reload displays |
| `on_prefix_apply(command, env)` | Before `axium run` executes | Modify or enrich env data       |
| `on_command_executed(result)`   | After any wrapped command     | Log or report results           |
| `on_spoke_loaded(spoke_name)`   | When a Spoke registers        | Initialization or health checks |

Hooks are registered via `events.on()` and receive relevant arguments.

---

## 🧩 Spoke Lifecycle

1. **Load** – The daemon scans `spokes/` and imports each Spoke.
2. **Register** – Calls each Spoke's `register(app, events)` function.
3. **Activate** – Hooks and commands become live.
4. **Reload** – On demand (`axium daemon reload`) without restart.
5. **Unload** – When daemon exits or Spoke disables itself.

Spokes can be safely added or removed between reloads — no reboot required.

---

## 📦 Directory Layout Example

    axium/
      core/
        prefix.py
        env.py
        spokes.py       # Loader
      spokes/
        aws_spoke.py
        terraform_spoke.py
        command_wrapper.py
      ...

---

## 🧱 Example: Command Wrapper Spoke

This built-in Spoke handles all prefixed command execution.
It defines the `axium run` CLI command and applies prefix rules before running binaries.

    def register(app, events, env):
        @app.command("run")
        def run(cmd: list[str]):
            from axium.core.prefix import apply_prefixes
            full_cmd = apply_prefixes(cmd)
            subprocess.run(full_cmd)

Other Spokes (like AWS or Terraform) simply register prefix metadata — not their own executors.

---

## 🚫 What Spokes Should NOT Do

| Don’t                              | Reason                                                |
| ----------------------------------- | ----------------------------------------------------- |
| Store their own YAML or JSON config | Breaks the declarative model                          |
| Manage state directly               | The daemon owns runtime state                         |
| Modify core files                   | All extension points go through events or the app API |
| Depend on other Spokes directly     | Use event hooks instead                               |

---

## 🧩 Data Access for Spokes

Spokes can safely read contextual data through `env` and `core` helpers.

    env.current() → returns active env key (e.g. "root")
    env.get("aws_profile") → returns property from envs.yaml
    env.all() → returns full envs.yaml mapping

They can also query or update via IPC if needed (`axiumd` exposes small APIs).

---

## ✅ Contribution Guidelines for Spokes

1. Keep it **stateless** — no persistent config or files.
2. Use `register()` with `(app, events, env)` signature.
3. Prefer declarative config in core (`envs.yaml`, etc.).
4. Ensure commands are **namespaced** (`axium aws-*`, `axium tf-*`).
5. Use events for coordination between Spokes.
6. Document any external dependencies (e.g., boto3).
7. Avoid blocking calls — use async when possible.

---

## 🧠 Design Summary

- Axium owns all persistent config (YAML, state).
- Spokes are pure Python runtime extensions.
- Each Spoke registers commands and event hooks.
- The daemon orchestrates, reloads, and isolates them.
- Core provides shared APIs for envs, prefixing, state, and IPC.

**Declarative data. Procedural extensions. One consistent model.**

---

_This doc complements `CONTRIBUTING.md` by describing how to author and structure new Spokes._
