# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Python client SDK plus demo scripts for Moon Dev's hosted Hyperliquid data layer (`https://api.moondev.com`). There is no server code here. The repo holds the client (`api.py`), terminal dashboard examples, an endpoint health monitor, and an LLM "swarm" agent that calls the client.

## Setup & commands

```bash
pip install -r requirements.txt          # requests, rich, python-dotenv, pandas, openai, termcolor
cp .env.example .env                     # set MOONDEV_API_KEY (required); OPENROUTER_API_KEY for ai_agents; TELEGRAM_* for api_monitor
```

There is no build step, linter config, or unit test suite. Every check calls the live API, so a valid `MOONDEV_API_KEY` is needed.

```bash
python api.py                            # test_all(): runs most client methods against the live API and prints ✅/⚠️
python api_monitor.py                    # loops every 15 min, validates every endpoint, sends failures to Telegram
python examples/01_liquidations.py       # any example runs standalone; some take CLI args (e.g. 02_positions.py BTC)
python ai_agents/run.py                  # interactive Director + multi-model swarm chat (needs OPENROUTER_API_KEY)
```

To check a single endpoint, call its method directly, e.g. `python -c "from api import MoonDevAPI; print(MoonDevAPI().get_price('BTC'))"`.

## Architecture

- **`api.py` → `MoonDevAPI`**: one class. Each endpoint is a thin method over `_get(endpoint, auth_required, params, timeout)`, which sends `X-API-Key`, calls `raise_for_status()`, and returns the `Response`. Methods return `.json()` (or parsed text for `.txt` endpoints). The long module docstring at the top is the canonical endpoint catalog: routes, auth, rate limits, and error codes (401 key errors, 429, 503 `fills_scanner_busy`, 410 retired endpoints). Keep it in sync.
- **`examples/NN_*.py`**: standalone `rich` dashboards. Each one does `sys.path.insert(0, <repo root>)` and then `from api import MoonDevAPI`, so it runs from any cwd. A few (`23`, `25`, `27`, `33`) call HTTP endpoints directly without the client. Output files go to `examples/data/`, which is gitignored.
- **`api_monitor.py`**: `get_all_tests(api)` returns `(name, callable, validator)` tuples covering the client surface.
- **`ai_agents/`**: `DirectorAgent` (director_agent.py) holds a hand-written `API_KNOWLEDGE` prompt that lists the client methods. It asks an LLM for a plan, pulls method calls out of that plan text, and runs them with `getattr(self.api, name)`. The argument parser passes **at most one string argument**. `SwarmAgent` sends the resulting data to several OpenRouter models in parallel (`SWARM_MODELS`).

## Adding or changing an endpoint

Git history shows endpoint changes touching several files together. Update all of them:
1. Add the method to `MoonDevAPI` and add the route to the `api.py` module docstring (and to `test_all()` if relevant).
2. Add a check to `get_all_tests()` in `api_monitor.py`.
3. Add the method to `API_KNOWLEDGE` in `ai_agents/director_agent.py` if the agent should be able to use it.
4. Add or update an `examples/NN_name.py` script with the next free number.
5. Update the tables in `README.md` and `examples/README.md`. `examples/README.md` also has a dated changelog.

Retired endpoints (return 410) get removed from the client, monitor, and agent knowledge, and listed under "RETIRED" in the `api.py` docstring.

## Conventions

- Match the existing code style: emoji-heavy docstrings and print output, "Moon Dev" attribution in comments and messages, and `# ==================== SECTION ====================` dividers in `api.py`.
- Fills polling: pass a time window (`minutes`/`since_ms` on `get_user_fills`, `start_time`/`minutes` on `get_fills`). Windowed calls take about 50ms. Un-windowed scans can take about 30s, and those callers raise `timeout`. Treat 503 `fills_scanner_busy` as "retry", not as an empty result. An HTTP 200 with an empty array means the wallet has no fills.
- Tick `sz`/`side` fields and bar volume `v` may be null or 0 for historical data collected before the size rollout. Code should handle missing values.
