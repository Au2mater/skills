---
name: uv-private-cli
description: Make private Python CLI repositories that use uv and pyproject.toml installable and runnable globally without publishing to PyPI. Use when Codex needs to inspect or modify a private Python CLI repo, add packaging metadata, add console entry points, validate uv installs, or help install an already-installable local CLI with uv tool install.
---

# UV Private CLI

## Goal

Make a private uv-managed Python CLI repo installable from a local path or private checkout, then runnable as a global command via `uv tool install`, without requiring publication to PyPI.

Prefer the smallest change that makes the existing tool packageable. Preserve the repo's current project style, build backend, import layout, dependency groups, lockfile policy, and CLI framework.

## Workflow

1. Inspect the repo before editing:
   - Confirm `pyproject.toml` exists at the repo root.
   - Read `[project]`, `[build-system]`, `[project.scripts]`, `[project.gui-scripts]`, `[project.entry-points]`, and `[tool.uv]`.
   - Identify the package/module layout: `src/<package>/`, `<package>/`, single `main.py`, or a scripts-only repo.
   - Find the actual callable CLI entry: commonly `main()`, `cli()`, `app()`, or a Typer/Click/argparse command.
   - Check whether tests or smoke commands already exist.

2. Decide whether the repo is already installable:
   - It is installable if it has valid project metadata, a build backend or explicit uv package configuration, importable package code, and at least one console entry point for the intended command.
   - If already installable, avoid code edits. Provide and, when appropriate, run the install commands in "Install an Existing CLI".
   - If not installable, identify the missing pieces in "Make the CLI Installable".

3. Present a minimal change proposal before editing:
   - List only the changes needed to make the CLI globally installable.
   - Include the expected command name and entry point target.
   - Explain any file moves or wrapper modules, if needed.
   - Ask for user confirmation before modifying files, unless the user has already explicitly approved implementation in the current request.

4. Implement only after confirmation:
   - Keep edits limited to the approved packaging, entry point, and minimal importability changes.
   - Do not publish to PyPI or configure publishing unless the user separately asks.
   - Do not rewrite CLI behavior, dependency management, or project layout beyond what is necessary for installation.

5. Validate locally before suggesting global installation:
   - Run `uv sync` when the project has a lockfile or normal uv project workflow.
   - Run the command through the project environment: `uv run <command> --help` or the repo's nearest harmless smoke command.
   - Run tests if present and reasonably scoped.
   - For packaging changes, run `uv build` when available and useful.

6. Install globally only after the project smoke test works, unless the user explicitly asks for only instructions.

## Make the CLI Installable

### Project metadata

Ensure `[project]` has at least:

```toml
[project]
name = "tool-name"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = []
```

Keep existing values when present. Choose a command/package name that is valid for Python packaging; use hyphens for the distribution name and underscores for import packages when needed.

### Build backend

If the project already has `[build-system]`, keep it unless it is broken. If it has no build backend, add one that matches the repo's packaging style. For new or simple repos, prefer Hatchling:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

For `src` layout, add explicit package selection when Hatchling cannot infer it:

```toml
[tool.hatch.build.targets.wheel]
packages = ["src/package_name"]
```

For flat package layout:

```toml
[tool.hatch.build.targets.wheel]
packages = ["package_name"]
```

Do not add setuptools, Hatchling, or package-selection config redundantly if the current backend already handles the package correctly.

### Package layout

Prefer an importable package over a root-level script as the global CLI implementation:

```text
src/package_name/__init__.py
src/package_name/cli.py
```

Move or wrap root-level `main.py` carefully:
   - Preserve behavior and imports.
   - Put CLI logic in an importable function, usually `main()`.
   - Leave a tiny compatibility wrapper only if existing workflows depend on `python main.py`.

Use this shape for a simple entry module:

```python
def main() -> None:
    ...


if __name__ == "__main__":
    main()
```

### Console entry point

Add the command under `[project.scripts]`:

```toml
[project.scripts]
tool-name = "package_name.cli:main"
```

The right-hand side must be importable and callable with no required Python arguments. For Click or Typer apps, point to the callable that the framework expects. If the repo exposes multiple commands, add multiple entries only when they are real user-facing commands.

### uv packaging flags

If `[tool.uv] package = false` exists, remove it or change it to `true` only when the user wants the project itself installed as a package. Do not add `package = true` when a normal build backend is enough.

## Install an Existing CLI

Use `uv tool install` for global commands from a private local checkout:

```bash
uv tool install --editable /absolute/path/to/repo
```

Use editable mode while developing so source changes are reflected without reinstalling. For a snapshot-style install, omit `--editable`:

```bash
uv tool install /absolute/path/to/repo
```

If the executable directory is not on `PATH`, run:

```bash
uv tool update-shell
```

Then restart the shell or source the updated shell profile. Verify:

```bash
uv tool list
<command> --help
```

To reinstall after metadata or entry point changes:

```bash
uv tool install --force --editable /absolute/path/to/repo
```

If the command name changed or the install is stale:

```bash
uv tool uninstall <project-name>
uv tool install --editable /absolute/path/to/repo
```

## Validation Checklist

Before finishing, report:

- The minimal change proposal that was approved, or that no edits were needed.
- The command name added or detected.
- The entry point target, such as `package_name.cli:main`.
- Whether the install should be editable or snapshot-style.
- The exact install command using an absolute path.
- The smoke tests run and their result.

Prefer these commands when applicable:

```bash
uv sync
uv run <command> --help
uv build
uv tool install --editable /absolute/path/to/repo
uv tool list
<command> --help
```

## Common Failures

- `No executables are provided by the package`: add or fix `[project.scripts]`.
- `ModuleNotFoundError`: entry point package path is wrong, package was not included in the wheel, or code still relies on root-relative imports.
- Build cannot find files: add the correct backend package-selection config.
- Command works with `uv run python main.py` but not after install: move CLI code into an importable package and point `[project.scripts]` at it.
- Installed command still runs old code: reinstall with `uv tool install --force --editable ...`, confirm `uv tool dir --bin`, and check shell command resolution with `which` or `where`.
