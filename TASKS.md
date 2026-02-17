# bloompschmapf — MVP Task List

Implementation order follows dependency order. Tasks within the same milestone
can be done in parallel; each milestone should be complete before the next.

---

## Milestone 0 — Project scaffold

### T-01: Create `pyproject.toml`

Set up the Python package using modern `pyproject.toml` (PEP 517/621):
- Package name: `bloompschmapf`, version `0.1.0`
- `src/` layout (`packages = [{include = "bloompschmapf", from = "src"}]`)
- Runtime dependencies: `pydantic>=2.0`, `pyyaml>=6.0`, `httpx>=0.27`,
  `typer>=0.12`, `python-dateutil>=2.9`, `tavily-python>=0.3`,
  `anthropic>=0.40`
- Dev dependencies (optional group `dev`): `pytest>=8.0`, `respx>=0.21`,
  `ruff>=0.4`, `mypy>=1.10`, `types-PyYAML`, `types-python-dateutil`
- CLI entry point: `bloompschmapf = "bloompschmapf.cli:app"`
- `[tool.ruff]` section: `line-length = 100`, select `["E","F","I","UP","B"]`
- `[tool.mypy]` section: `strict = true`, `python_version = "3.11"`
- `[tool.pytest.ini_options]` section: `testpaths = ["tests"]`

### T-02: Create package skeleton (empty modules)

Create all directories and stub `__init__.py` / module files so imports resolve
before any logic is written. Files needed:

```
src/bloompschmapf/__init__.py          (set __version__ = "0.1.0")
src/bloompschmapf/cli.py
src/bloompschmapf/config.py
src/bloompschmapf/models.py
src/bloompschmapf/pipeline.py
src/bloompschmapf/deduper.py
src/bloompschmapf/ranker.py
src/bloompschmapf/llm.py
src/bloompschmapf/renderer.py
src/bloompschmapf/sources/__init__.py
src/bloompschmapf/sources/base.py
src/bloompschmapf/sources/tavily.py
src/bloompschmapf/notifiers/__init__.py
src/bloompschmapf/notifiers/base.py
src/bloompschmapf/notifiers/telegram.py
tests/__init__.py
tests/conftest.py
```

---

## Milestone 1 — Core data types

### T-03: Implement `models.py`

Define three Pydantic v2 models (no external service dependencies):

- `SearchResult`: `title: str`, `url: str`, `snippet: str`,
  `published_at: datetime | None`, `source_name: str`
- `RankedItem`: `result: SearchResult`, `score: float` (0..1),
  `rationale: str` (LLM's 1-sentence reason),
  `summary_bullets: list[str]` (LLM's 1-2 bullets)
- `Report`: `agent_name`, `query`, `window_days`, `run_at: datetime`,
  `items: list[RankedItem]`, `fetched_count: int`, `deduped_count: int`

### T-04: Implement `config.py`

Pydantic v2 models for YAML agent configuration:

- `LLMConfig`: `provider: Literal["anthropic"] = "anthropic"`,
  `model: str = "claude-haiku-4-5-20251001"`,
  `api_key_env: str = "BLOOMPSCHMAPF_ANTHROPIC_API_KEY"`,
  `max_items_to_rank: int = 30`
- `TavilySourceConfig`: `type: Literal["tavily"]`,
  `api_key_env: str = "BLOOMPSCHMAPF_TAVILY_API_KEY"`,
  `max_results: int = 50`
- `SourceConfig = Annotated[TavilySourceConfig, ...]` (discriminated on `type`)
- `TelegramNotifierConfig`: `type: Literal["telegram"]`,
  `bot_token_env: str = "BLOOMPSCHMAPF_TELEGRAM_BOT_TOKEN"`,
  `chat_id_env: str = "BLOOMPSCHMAPF_TELEGRAM_CHAT_ID"`
- `NotifierConfig = Annotated[TelegramNotifierConfig, ...]` (discriminated on `type`)
- `AgentConfig`: `name`, `description`, `query`, `window_days: int = 7`,
  `max_results: int = 10`, `source: SourceConfig`,
  `llm: LLMConfig = LLMConfig()`,
  `notifiers: list[NotifierConfig]`
- `load_config(path: Path) -> AgentConfig` — reads YAML and validates;
  raises `SystemExit` with a human-readable message on validation error.

---

## Milestone 2 — Pipeline components

### T-05: Implement `sources/base.py` and `sources/__init__.py`

- `base.py`: `BaseSource` ABC with abstract method
  `search(query: str, window_days: int, max_results: int) -> list[SearchResult]`
- `__init__.py`: `source_from_config(cfg: SourceConfig) -> BaseSource` factory

### T-06: Implement `sources/tavily.py` — TavilySource

`TavilySource(BaseSource)`:
- Constructor takes `TavilySourceConfig`; reads API key from env var at
  instantiation time; raises `RuntimeError` with clear message if missing.
- `search()` calls `tavily_client.search(query, days=window_days,
  max_results=max_results)`.
- Maps Tavily response dicts to `SearchResult` objects:
  - `published_date` → parse with `dateutil.parser.parse`, store as UTC-aware
    datetime; `None` if absent or unparseable.
  - `source_name = "tavily"`
- On HTTP / API errors, log the error and return an empty list (do not crash).

### T-07: Implement `deduper.py` — Deduplicator

`Deduplicator` class with method `deduplicate(results: list[SearchResult]) -> list[SearchResult]`:

1. **URL dedup**: normalise each URL (lowercase scheme+host, strip `www.`,
   strip trailing slash, strip query string). Keep first occurrence.
2. **Title similarity dedup**: for remaining results, compare all pairs using
   `difflib.SequenceMatcher(None, a, b).ratio()`. If ratio > 0.85, drop the
   later item.
3. Return deduplicated list preserving order of first occurrence.

### T-08: Implement `ranker.py` — RecencyPreFilter

`RecencyPreFilter` class with method
`prefilter(results: list[SearchResult], max_items: int) -> list[SearchResult]`:

- Sort by `published_at` descending. Items with `published_at = None` sort
  after all dated items.
- Return the first `max_items` entries.
- This is purely a cost-control gate before the LLM call — no scoring.

### T-09: Implement `llm.py` — LLMRanker

`LLMRanker(config: LLMConfig)` with method
`rank_and_summarize(results, query, agent_description) -> list[RankedItem]`:

**Prompt construction:**
- Build a numbered list of results: `[i] Title: ... URL: ... Date: ... Snippet: ...`
- System/user message instructs Claude to return a JSON array where each entry
  has `index` (int), `score` (0-100), `rationale` (one sentence),
  `bullets` (list of 1-2 strings).
- Instruct it to omit results with score < 20, and to never invent or modify URLs.
- Request only JSON output, no surrounding text.

**API call:**
- Use `anthropic.Anthropic()` client; read API key from env var.
- Call `client.messages.create(model=config.model, max_tokens=2048, ...)`.

**Response parsing:**
- Extract text content from the response.
- `json.loads()` the text; wrap in try/except.
- For each valid entry: look up `results[index]`, normalise score to `score / 100.0`,
  build `RankedItem`.
- Sort final list by score descending.

**Fallback on any error** (API error, JSON parse failure, missing index):
- Log the error at WARNING level.
- Return all input results as `RankedItem` with `score=0.5`,
  `rationale="LLM unavailable"`, `summary_bullets=[]`.

### T-10: Implement `renderer.py` — Renderer

`Renderer` class with method `render(report: Report) -> list[str]`:

- Returns a list of message strings, each ≤ 4096 characters (Telegram limit).
- Uses Telegram HTML formatting: `<b>`, `<i>`, `<a href="...">`.
- Per-item format:
  ```
  {rank}. <b><a href="{url}">{title}</a></b>
  • {bullet_1}
  • {bullet_2}
  📅 {published_date}
  <i>{rationale}</i>
  ```
  - Omit date line if `published_at` is None.
  - If `summary_bullets` is empty (LLM fallback), render first sentence of raw
    snippet instead (helper `_first_sentence(text) -> str`, max 200 chars).
  - HTML-escape all user-supplied text with `html.escape()`.
- Footer (appended to last message):
  ```
  ─────────────────────────
  🤖 <b>{agent_name}</b> | window: {window_days}d | {run_at:%Y-%m-%d %H:%M UTC}
  Fetched: {fetched_count} → deduped: {deduped_count} → shown: {len(items)}
  ```
- Splitting: accumulate items; when adding the next item would exceed 4096 chars,
  start a new message. Footer always goes on the last message.

### T-11: Implement `notifiers/base.py` and `notifiers/__init__.py`

- `base.py`: `BaseNotifier` ABC with abstract method `send(messages: list[str]) -> None`
- `__init__.py`: `notifier_from_config(cfg: NotifierConfig) -> BaseNotifier` factory

### T-12: Implement `notifiers/telegram.py` — TelegramNotifier

`TelegramNotifier(BaseNotifier)`:
- Constructor takes `TelegramNotifierConfig`; reads token and chat_id from
  env vars at instantiation; raises `RuntimeError` if either is missing.
- `send(messages: list[str])`: POST each message to
  `https://api.telegram.org/bot{token}/sendMessage` with
  `{"chat_id": ..., "text": ..., "parse_mode": "HTML",
   "disable_web_page_preview": true}`.
- Uses `httpx.Client` with `timeout=15.0`.
- On non-2xx response, log the status and body; raise `RuntimeError`.
- Never log or print the bot token.

---

## Milestone 3 — Pipeline + CLI

### T-13: Implement `pipeline.py` — `run_agent()`

```python
def run_agent(config: AgentConfig, dry_run: bool = False) -> Report:
```

Orchestrates the full flow:
1. Log run start (agent name, query, window).
2. `source = source_from_config(config.source)`
3. `results = source.search(config.query, config.window_days, config.source.max_results)`
4. `deduped = Deduplicator().deduplicate(results)`
5. `filtered = RecencyPreFilter().prefilter(deduped, config.llm.max_items_to_rank)`
6. `ranked = LLMRanker(config.llm).rank_and_summarize(filtered, config.query, config.description)`
7. Trim `ranked` to `config.max_results` items.
8. Build `Report(...)` with counts (`fetched_count=len(results)`,
   `deduped_count=len(deduped)`).
9. `messages = Renderer().render(report)`
10. If `dry_run`: print messages to stdout; skip notifiers.
11. Otherwise: for each notifier config, `notifier_from_config(cfg).send(messages)`.
12. Log run end with counts and timing.
13. Return `report`.

Use stdlib `logging` with format `%(asctime)s [%(levelname)s] %(name)s: %(message)s`.
Log at INFO by default.

### T-14: Implement `cli.py` — Typer CLI

```
bloompschmapf run <config_path> [--dry-run] [--verbose]
```

- `run(config_path: Path, dry_run: bool = False, verbose: bool = False)`
- If `verbose`: set root logger to DEBUG.
- Load config via `load_config(config_path)`.
- Call `run_agent(config, dry_run=dry_run)`.
- Exit with code 1 and error message if any unhandled exception escapes.

---

## Milestone 4 — Example config + documentation

### T-15: Create `agents/example.yaml`

An example agent config ready to run with real env vars set:

```yaml
agent:
  name: indie-folk-new-releases
  description: "New indie folk song releases and announcements from the last 7 days"
  query: "new indie folk songs releases 2026"
  window_days: 7
  max_results: 10

source:
  type: tavily
  max_results: 50

llm:
  model: claude-haiku-4-5-20251001
  max_items_to_rank: 30

notifiers:
  - type: telegram
    bot_token_env: BLOOMPSCHMAPF_TELEGRAM_BOT_TOKEN
    chat_id_env: BLOOMPSCHMAPF_TELEGRAM_CHAT_ID
```

### T-16: Update `AGENTS.md` / `CLAUDE.md` with repo structure and quick-start

Fill in the `<TBA>` sections:
- Repository structure (copy from PLAN.md layout)
- Quick start: install, configure, run once, environment variables
  (including `BLOOMPSCHMAPF_ANTHROPIC_API_KEY`)
- Key entry points: `run_agent()` in `pipeline.py`, `BaseSource` / `BaseNotifier`
- How to add a new source provider
- How to add a new notifier

---

## Milestone 5 — Tests

### T-17: `tests/test_config.py`

- Valid YAML loads to `AgentConfig` successfully.
- `LLMConfig` defaults are applied when `llm:` section is omitted.
- Invalid `window_days` (negative) raises `ValidationError`.
- Missing required fields raise `ValidationError`.
- `load_config()` with a non-existent file calls `sys.exit`.

### T-18: `tests/test_deduper.py`

- Exact URL duplicates (after normalisation) are removed.
- Items with different URLs but near-identical titles (ratio > 0.85) are deduped.
- Items with distinct titles and URLs are all retained.
- Empty input returns empty list.

### T-19: `tests/test_ranker.py`

- Items with a `published_at` are sorted before items without one.
- Items are returned in descending recency order.
- `max_items` is respected: output length ≤ `max_items`.
- Empty input returns empty list.

### T-20: `tests/test_llm.py`

Use `respx` to mock the Anthropic HTTP endpoint
(`https://api.anthropic.com/v1/messages`):

- **Happy path**: mock returns a valid JSON array with scores and bullets.
  Assert `ranked[0].score >= ranked[1].score` (sorted descending).
  Assert `ranked[0].summary_bullets` is non-empty.
  Assert `ranked[0].rationale` is non-empty.
  Assert no hallucinated URLs (all URLs in output exist in input).
- **Malformed JSON response**: mock returns non-JSON text.
  Assert fallback: all input items returned, `rationale == "LLM unavailable"`.
- **API error (500)**: mock returns HTTP 500.
  Assert fallback behaviour, no exception raised.
- **Empty results input**: returns empty list without calling the API.

### T-21: `tests/test_renderer.py`

- A report with 3 items renders to a list with at least one message.
- No single message exceeds 4096 characters.
- The footer line appears in the last message.
- A report whose total text exceeds 4096 chars is split into multiple messages.
- Items with `summary_bullets = []` fall back to raw snippet first sentence.
- HTML special characters in titles/snippets are escaped (`&`, `<`, `>`).

### T-22: `tests/test_pipeline.py` — integration test with mocked HTTP

Use `respx` to mock:
1. The Tavily search API response (return 5 fake `SearchResult`-shaped dicts).
2. The Anthropic messages endpoint (return a valid ranked JSON array).
3. The Telegram `sendMessage` endpoint (return `{"ok": true}`).

Test `run_agent()` end-to-end:
- Telegram `sendMessage` is called at least once.
- The request body contains `"chat_id"` and `"text"` and `"parse_mode": "HTML"`.
- `report.fetched_count == 5`.
- `report.items` is non-empty and sorted by score descending.
- No item in the report has a URL not present in the original mock results.

Test `dry_run=True`:
- Anthropic endpoint is still called (ranking still happens).
- Telegram endpoint is **not** called.
- Function still returns a valid `Report`.

---

## Definition of done

- [ ] `pip install -e ".[dev]"` completes without errors.
- [ ] `bloompschmapf run agents/example.yaml --dry-run` prints a report to stdout.
- [ ] `pytest -q` passes with all tests green (no live network calls).
- [ ] `ruff check .` reports no errors.
- [ ] `mypy src/` reports no errors.
