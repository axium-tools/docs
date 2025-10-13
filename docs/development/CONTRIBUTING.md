# 🤝 Contributing to Axium

Welcome!
Axium is an async, extensible DevOps assistant — and contributions from other builders are always welcome.
This guide explains how to set up a local development environment, follow our standards, and extend Axium via Spokes.

---

## 🧭 Philosophy

Axium follows a few simple rules:

- Keep it calm — clarity and composure over cleverness
- Async-first — all daemon or long-running code uses asyncio
- Predictable UX — axium `<noun>` `<verb>` pattern everywhere
- Extensible — all features should be possible via Spokes
- Minimal dependencies — stdlib + Typer + YAML preferred

---

## ⚙️ Local Development Setup

1️⃣ Clone and install

    git clone https://github.com/axium-tools/axium.git
    cd axium
    pip install -e .[dev]

2️⃣ Enable shell integration

    source bash/init.sh

This adds the ax alias and initializes environment functions.

3️⃣ Run tests

    pytest -q

Tests are run with pytest and must pass before PR submission.

---

## 🧪 Testing Standards

- Unit tests live in the tests/ directory
- Every CLI command and IPC endpoint must have a corresponding test
- Async code should use pytest.mark.asyncio
- Avoid mocking unless network or file IO is required
- Run tests locally before committing

Example commands:

    pytest -v
    pytest tests/core/test_ipc.py
    pytest --maxfail=1 --disable-warnings -q

---

## 🧩 Spoke Development

Spokes are how Axium grows — each Spoke extends Axium’s commands, palette, or HUD.
They live in ~/.config/axium/spokes/`<name>`/.

Example layout:

    ~/.config/axium/spokes/aws/
      ├─ spoke.yaml
      └─ aws_spoke.py

Example spoke.yaml:

    name: aws
    entrypoint: aws_spoke:register

Example aws_spoke.py:

    def register(app):
        @app.command("aws-whoami")
        def aws_whoami():
            import boto3
            print(boto3.client("sts").get_caller_identity())

Test it:

    axium aws-whoami

---

## 🧱 Code Style

- Formatting: use black (line length 100)
- Imports: sorted via isort
- Linting: flake8 must pass cleanly
- Typing: add type hints to all public functions
- Logging: use the logging module, not print statements

Format locally:

    black .
    isort .
    flake8

---

## 🔄 Branching & Workflow

| Step               | Action                                   |
| ------------------ | ---------------------------------------- |
| main               | Stable branch (always green CI)          |
| develop            | Active dev branch (features merged here) |
| feature/`<name>` | Your working branch                      |
| spoke/`<name>`   | Spoke-specific branch for plugin dev     |

Example flow:

    git checkout -b feature/state-persistence
    # ...make changes...
    pytest -q
    git commit -am "Add persistent state.json handling"
    git push origin feature/state-persistence

Then open a Pull Request into develop.

---

## 🧾 Commit Guidelines

Format commits as:

    `<type>`(scope): short summary

Examples:

    feat(daemon): add reload IPC command
    fix(cli): handle missing socket permissions
    test(ipc): increase coverage for state reload
    docs(readme): clarify palette usage

Types: feat, fix, docs, test, refactor, chore.

---

## 🧰 Pull Requests

Before you submit:

- ✅ All tests pass
- ✅ Code formatted (black, isort, flake8)
- ✅ No debug prints
- ✅ ROADMAP.md updated if needed
- ✅ New features documented

CI will verify tests and lint automatically.

---

## 🧠 Good First Issues

- Improve CLI help text
- Expand test coverage for ipc.py
- Add example Spoke (e.g., system, git, or aws)
- Improve error handling for daemon restarts

---

## 🪩 Community

- Discussions and design chat will live under GitHub Discussions
- Spokes can be submitted to the registry at registry.axium.tools once it launches
- For now, feel free to open issues or share ideas directly via PR comments

---

## 🪪 License

By contributing, you agree that your code is licensed under the Apache 2.0 License (LICENSE).

---

_“Structure for your terminal.”_
