# Hermes Agent — Project Context

## What This Is

Hermes Agent (v0.15.1) by Nous Research — an AI agent framework with CLI, gateway, and ACP integrations.

## Entry Points

| Command | Module | Purpose |
|---------|--------|---------|
| `hermes` | `hermes_cli.main:main` | Interactive CLI chat (default), gateway, setup, dashboard |
| `hermes-agent` | `run_agent:main` | Standalone programmatic agent runner |
| `hermes-acp` | `acp_adapter.entry:main` | ACP JSON-RPC server for editor integration |

## Key Directories

- `hermes_cli/` — CLI commands and config management
- `agent/` — Core agent logic, tool registry, redaction, transport
- `gateway/` — Messaging platform integration (Telegram, Discord, Slack, etc.)
- `acp_adapter/` — Agent Client Protocol for editor integration
- `skills/` — Bundled skill definitions
- `scripts/` — Install scripts (`install.sh`, `install.ps1`)
- `docker/` — Docker entrypoint and s6-overlay config
- `Learning/` — Architecture notes and development guide

## Config Locations

- **Runtime config**: `~/.hermes/config.yaml` (model, security, tools)
- **API keys**: `~/.hermes/.env`
- **Sessions**: `~/.hermes/sessions/`
- **State DB**: `~/.hermes/state.db` (SQLite)

## WSL vs Windows

WSL (`/home/skype/.hermes/`) and Windows (`C:\Users\skype\.hermes/`) have **independent config directories**.
Changes to one do NOT affect the other. See `Learning/Readme.md` for details.

WSL hermes is installed at `~/.hermes/hermes-agent/` with uv-managed venv (Python 3.11.15).
Command `hermes` is available via `~/.local/bin/hermes` wrapper (login shell only).

## Development

```bash
# Dev mode (code changes take effect immediately)
pip install -e .

# Run CLI
python -m hermes_cli.main

# Run tests
pytest tests/
```

## Architecture Reference

See `Learning/Framework.md` for detailed architecture audit (core classes, call chains, plugin system).
See `Learning/Readme.md` for WSL setup and development workflow.
