# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TrendRadar (热点新闻聚合与分析工具) is a hot-news aggregation and analysis tool. It crawls trending lists from Chinese/global platforms (via the NewsNow API) and RSS feeds, filters them by keywords or AI, optionally runs AI analysis/translation, generates HTML reports, and pushes results to 9 notification channels. The codebase comments and most user-facing strings are in Simplified Chinese — match that convention when editing.

The repo ships **two distinct deliverables** from one source tree:
- `trendradar/` — the crawler + analysis + notification pipeline (the main app).
- `mcp_server/` — a FastMCP 2.0 server exposing the stored data as MCP tools for AI clients (Cherry Studio, Claude Desktop, etc.). These are versioned independently (`version` vs `version_mcp`).

## Commands

Dependencies are managed with **uv** (`uv.lock` is authoritative; `requirements.txt` mirrors it). Requires Python >= 3.12.

```bash
uv sync                       # install deps into .venv
uv sync --frozen --no-dev     # CI/reproducible install (used by GitHub Actions)

# Run the main pipeline (crawl → analyze → report → notify)
uv run python -m trendradar

# Diagnostics / utility subcommands
uv run python -m trendradar --doctor             # environment & config health check (writes output/meta/doctor_report.json)
uv run python -m trendradar --show-schedule      # print resolved schedule for the current time
uv run python -m trendradar --test-notification  # send a connectivity test to all configured channels

# Run the MCP server
uv run python -m mcp_server.server --transport stdio          # for local MCP clients
uv run python -m mcp_server.server --transport http --port 3333   # for remote access (also: ./start-http.sh)
```

Console entry points (defined in `pyproject.toml`): `trendradar` → `trendradar.__main__:main`, `trendradar-mcp` → `mcp_server.server:run_server`.

**There is no automated test suite, linter, or build step configured.** `--doctor` is the closest thing to a self-check; run it after config-affecting changes.

## Configuration Model

All runtime behavior is driven by files in `config/`, loaded by `trendradar/core/loader.py`:
- `config.yaml` — master config (platforms, RSS, schedule, AI, storage, notification, display).
- `frequency_words.txt` — keyword groups + global filters, custom plaintext format (`[GLOBAL_FILTER]` / `[WORD_GROUPS]` sections, blank-line-separated groups, `/a|b/` regex-style alternation).
- `timeline.yaml` — schedule periods, day plans, week map (see Scheduling below).
- `ai_interests.txt` + `config/ai_filter/*.txt` + `ai_analysis_prompt.txt` + `ai_translation_prompt.txt` — AI prompt/interest inputs.
- `config/custom/` — user overrides (`ai/`, `keyword/`) referenced by name from the scheduler.

**Environment variables override `config.yaml`** for nearly every setting (see `_get_env_*` helpers and the mapping in `loader.py`). This is how GitHub Actions and Docker inject secrets (webhook URLs, `AI_API_KEY`, `S3_*`, `GITHUB_ACTIONS`, etc.) without committing them. When adding a config option, wire it through `loader.py` with an env override and surface it via `AppContext` rather than reading `config` dicts ad hoc.

Config files carry a `Version: X.Y.Z` header line; `version_configs` tracks expected versions and `__main__.py` compares local vs remote on startup. Bump these when changing config schema.

## Architecture

The pipeline is orchestrated by `NewsAnalyzer` in `trendradar/__main__.py` (~2300 lines — the single biggest file and the place to start reading). `main()` loads config, runs version checks, constructs `NewsAnalyzer`, and calls `.run()`. The flow:

1. **Crawl** (`trendradar/crawler/`) — `DataFetcher` hits the NewsNow API (`DEFAULT_API_URL`, overridable per-platform), with optional `expected_domain` HTTPS/domain safety validation. `crawler/rss/` fetches RSS via `feedparser` with freshness filtering.
2. **Store** (`trendradar/storage/`) — results are normalized to `NewsData`/`RSSData` and persisted. Backend is `local` (SQLite + TXT/HTML in `output/`), `remote` (S3-compatible: R2/OSS/COS/S3 via boto3), or `auto` (picks remote under GitHub Actions, else local — see `manager.py::_resolve_backend_type`). SQLite schemas live in `storage/*.sql`.
3. **Analyze** (`trendradar/core/`) — `count_word_frequency` / `count_rss_frequency` match titles against keyword groups and rank them. Alternatively, `AIFilter` (`trendradar/ai/filter.py`, driven from `AppContext.run_ai_filter`) classifies items against AI-extracted interest tags stored in SQLite, with incremental re-classification when interests change.
4. **AI** (`trendradar/ai/`) — `AIAnalyzer`, `AITranslator`, `AIFilter` all go through `AIClient`, a thin wrapper over **LiteLLM** (`completion`) supporting 100+ providers; model id is `provider/model_name`. Prompts are loaded via `prompt_loader.py`.
5. **Report** (`trendradar/report/`) — generates HTML reports into `output/html/`, including a `latest/<mode>.html`.
6. **Notify** (`trendradar/notification/`) — `NotificationDispatcher` fans out to feishu, dingtalk, wework, telegram, email, ntfy, bark, slack, and a generic webhook. Each channel has format-specific rendering/splitting (`formatters.py`, `splitter.py`, `senders.py`). Multi-account per channel is supported (`parse_multi_account_config`).

### AppContext is the seam
`trendradar/context.py::AppContext` wraps the config dict and exposes every config-dependent operation (time, storage manager, frequency loading, stats, report/notification rendering, scheduler, AI filter). It exists to eliminate global state and is lazily-singleton for storage and scheduler. **Prefer adding methods/properties here over threading raw config through call sites.**

### Three report modes
`incremental` (only newly-appeared items), `current` (current snapshot of the list), `daily` (full-day cumulative). Defined in `NewsAnalyzer.MODE_STRATEGIES` and threaded through every stage — RSS processing, AI filter conversion, and HTML/notification all branch on mode. Two display modes exist: `keyword` (group by matched word) and `platform`.

### Scheduling
`trendradar/core/scheduler.py::Scheduler` resolves, for the current time, a `ResolvedSchedule` deciding `collect` / `analyze` / `push`, the `report_mode`, `ai_mode`, and once-per-period guards. It's built from `config.yaml`'s `schedule` section (a `preset`) plus `timeline.yaml` (periods + day_plans + week_map). The scheduler records executions in storage so "once per period" push/analyze works across cron invocations.

## Deployment Targets

- **GitHub Actions** (`.github/workflows/crawler.yml`) — cron-driven crawl. Note the **7-day trial mechanism**: the workflow auto-disables after 7 days from its first run unless "Check In" is run. Secrets are passed as env vars. Long-term use is steered toward Docker.
- **Docker** (`docker/`) — `Dockerfile` (app) and `Dockerfile.mcp` (MCP server). `entrypoint.sh` runs `python -m trendradar` either once (`RUN_MODE=once`) or on a cron via **supercronic** (`CRON_SCHEDULE`), and starts a web server via `manage.py start_webserver`. `docker/manage.py` is the operator CLI (status/config/logs/restart).
- **GitHub Pages** — `docs/` and root `index.html` host a visual config editor.

## Conventions

- New AI integrations route through `AIClient`/LiteLLM, not provider SDKs directly. Model strings are `provider/model_name`.
- Notification channel additions touch `dispatcher.py`, `formatters.py`, `senders.py`, and `splitter.py` together, plus the config/env plumbing in `loader.py` and the doctor check in `__main__.py::_run_doctor`.
- Output is written under `output/` (gitignored data); never hardcode dates — use `AppContext.format_date()`/`format_time()` and the configured timezone.
- When adding an MCP tool, register it with `@mcp.tool` in `mcp_server/server.py` and implement logic in the matching `mcp_server/tools/*.py` service class (instances are lazily created in `_get_tools`).
