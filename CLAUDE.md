# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Dev Environment

```bash
source .venv/bin/activate   # or: source venv/bin/activate
```

## Testing

Always use the wrapper — never call `pytest` directly. It enforces CI parity (hermetic env, UTC, C.UTF-8, credential isolation, subprocess-per-test):

```bash
scripts/run_tests.sh                                  # full suite
scripts/run_tests.sh tests/gateway/                   # one directory
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # one test
scripts/run_tests.sh -v --tb=long                     # pass-through pytest flags
scripts/run_tests.sh --no-isolate tests/foo/          # disable subprocess isolation (faster, for debugging)
```

Each test file runs in its own spawned subprocess (no module-level leakage between files). `tests/conftest.py` has an autouse fixture that redirects `HERMES_HOME` to a temp dir and blanks all `*_API_KEY`/`*_TOKEN` env vars.

## Lint

```bash
ruff check .     # Only PLW1514 (unspecified-encoding) is enforced
```

## Architecture

Hermes Agent is a self-improving AI agent by Nous Research. It runs as either a CLI (`hermes`) or a multi-platform messaging gateway (`hermes gateway`).

### Core files (load-bearing entry points)

| File | Role |
|------|------|
| `run_agent.py` | `AIAgent` class — core conversation loop with tool calling (~190k chars) |
| `model_tools.py` | Tool orchestration, schema collection, `handle_function_call()` dispatch |
| `toolsets.py` | `TOOLSETS` dict defining which tools each platform can use; `_HERMES_CORE_TOOLS` is the default bundle |
| `cli.py` | `HermesCLI` class — interactive CLI with prompt_toolkit + Rich (~685k chars) |
| `hermes_state.py` | `SessionDB` — SQLite (WAL mode, FTS5 search) session store |
| `hermes_constants.py` | `get_hermes_home()` / `display_hermes_home()` — profile-aware paths, no deps, import-safe |
| `hermes_logging.py` | Profile-aware logging setup |
| `gateway/run.py` | Gateway runner — manages platform adapters, dispatches messages/commands |
| `agent/` | Provider adapters, memory, compression, context engine, skill loading, LSP integration |
| `tools/` | Auto-discovered tool implementations via `tools/registry.py` |
| `hermes_cli/` | CLI subcommands, config, slash-command registry, skin engine, plugins loader, curses UI |
| `cron/` | `jobs.py` (store) + `scheduler.py` (tick loop) — natural-language scheduled automation |

### Dependency chain

```
tools/registry.py  (no deps)
       ↑
tools/*.py  (each calls registry.register() at import time)
       ↑
model_tools.py  (imports tools/registry, triggers auto-discovery)
       ↑
run_agent.py, cli.py, batch_runner.py
```

### Configuration

- **User config:** `~/.hermes/config.yaml` (settings)
- **Secrets:** `~/.hermes/.env` (API keys, tokens — secrets only)
- **DEFAULT_CONFIG** in `hermes_cli/config.py` — all config keys with defaults
- Never hardcode `~/.hermes` — use `get_hermes_home()` from `hermes_constants`
- Never hardcode `~/.hermes` in user-facing messages — use `display_hermes_home()`

Three config loaders exist and loading a default into one without the others causes drift:
- `load_cli_config()` in `cli.py` (CLI mode) 
- `load_config()` in `hermes_cli/config.py` (subcommands like `hermes tools`, `hermes setup`)
- Direct YAML read in `gateway/run.py`

### Tools

**Adding a core tool** requires 2 changes:
1. Create `tools/your_tool.py` with `registry.register(name=..., toolset=..., schema=..., handler=..., ...)` 
2. Add the tool name to `_HERMES_CORE_TOOLS` (or another toolset) in `toolsets.py`

Auto-discovery imports any `tools/*.py` with a `registry.register()` call — no manual import list. But the tool is only **exposed** to an agent when its name appears in a toolset.

**User/custom tools** should be plugins in `~/.hermes/plugins/`, not edits to core.

### Slash commands

All defined in `COMMAND_REGISTRY` in `hermes_cli/commands.py` as `CommandDef` objects. Adding a command:
1. Add `CommandDef(...)` to `COMMAND_REGISTRY`
2. Add handler in `HermesCLI.process_command()` in `cli.py`
3. If gateway-available, add handler in `gateway/run.py`

Aliases only need adding to the `aliases` tuple on the existing `CommandDef` — everything updates automatically.

## Critical Policies

### Prompt caching
Do NOT change past context, toolsets, or reload memories mid-conversation. Cache-breaking forces dramatically higher costs. Slash commands that mutate system-prompt state must default to deferred invalidation (takes effect next session), with an opt-in `--now` flag.

### Plugins must not modify core files
`run_agent.py`, `cli.py`, `gateway/run.py`, `hermes_cli/main.py` etc. are off-limits. Expand the plugin surface (new hook, new ctx method) if needed.

### Dependency pinning
All deps in `pyproject.toml` are exact-pinned to `==X.Y.Z`. No ranges. This was a direct response to supply-chain attacks (Mini Shai-Hulud worm, May 2026). When updating, bump the pin and run `uv lock`.

### Dependencies are split: core vs lazy
Only universally-needed packages go in `[project.dependencies]`. Provider-specific packages (anthropic, firecrawl, exa, fal, edge-tts, etc.) go in extras or `tools/lazy_deps.py` for first-use resolution. This limits blast radius for supply-chain attacks.

## Known Pitfalls

- **Never hardcode `~/.hermes`** — use `get_hermes_home()` (code paths) or `display_hermes_home()` (user messages). Hardcoding breaks profiles.
- **Tests must not write to `~/.hermes/`** — `tests/conftest.py` redirects to a temp dir automatically.
- **Don't write change-detector tests** — no snapshot tests of model catalogs, config version numbers, or enumeration counts. Test invariants and behavior, not specific data.
- **Don't use `\033[K`** in display code — leaks as literal text under prompt_toolkit. Use space-padding instead.
- **Don't introduce new `simple_term_menu` usage** — use `hermes_cli/curses_ui.py` for interactive menus.
- **Gateway has TWO message guards** — new commands that must reach the runner while an agent is blocked must bypass both the base adapter's queue and the gateway runner's intercept.
- **Squash merges from stale branches** silently revert recent fixes. Ensure branch is up to date with main before squash-merging.
- **Bare `open()`/`read_text()`/`write_text()`** without encoding can corrupt non-ASCII content on Windows (cp1252 default). Ruff enforces this via PLW1514.
- **`os.kill(pid, 0)`** is a silent killer on Windows. Use `psutil.pid_exists()` instead.

For deeper developer documentation, see `AGENTS.md`.
