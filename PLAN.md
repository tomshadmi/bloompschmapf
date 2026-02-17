# bloompschmapf — MVP Implementation Plan

## Goal

Build the smallest working version of the framework: an agent that you configure
with a YAML file, runs from the CLI, searches for recent results via a single
provider, deduplicates, uses an LLM (via API) to rank and summarize results,
renders a Markdown report, and delivers it to a Telegram chat.

No scheduler daemon in MVP. No email. One search provider.

---

## Scope boundaries

| In scope (MVP)                                   | Out of scope (post-MVP)               |
|--------------------------------------------------|---------------------------------------|
| Single search provider (Tavily)                  | Email notifier                        |
| Telegram notifier only                           | Scheduler / daemon mode               |
| LLM ranking + per-item summarization (Anthropic) | Multiple parallel sources             |
| Recency pre-filter (limits items sent to LLM)    | Web UI / API server                   |
| CLI `run` command (run-once)                     | Persistent dedup store across runs    |
| YAML agent config                                | Non-Anthropic LLM providers           |
| Unit + integration tests (mocked HTTP)           |                                       |

---

## Repository layout

```
bloompschmapf/
├── src/
│   └── bloompschmapf/
│       ├── __init__.py          # package version
│       ├── cli.py               # typer CLI — `bloompschmapf run <config>`
│       ├── config.py            # AgentConfig, SourceConfig, LLMConfig, NotifierConfig (Pydantic v2)
│       ├── models.py            # SearchResult, RankedItem, Report
│       ├── pipeline.py          # run_agent() — wires all components together
│       ├── deduper.py           # URL normalisation + title-similarity dedup
│       ├── ranker.py            # recency pre-filter (limits items before LLM call)
│       ├── llm.py               # LLM-based ranker + summarizer (Anthropic)
│       ├── renderer.py          # renders Report → HTML string (Telegram-safe)
│       ├── sources/
│       │   ├── __init__.py      # source_from_config() factory
│       │   ├── base.py          # BaseSource ABC
│       │   └── tavily.py        # TavilySource
│       └── notifiers/
│           ├── __init__.py      # notifier_from_config() factory
│           ├── base.py          # BaseNotifier ABC
│           └── telegram.py      # TelegramNotifier (httpx, message splitting)
├── agents/
│   └── example.yaml             # example agent config (indie-folk songs)
├── tests/
│   ├── conftest.py
│   ├── test_config.py
│   ├── test_deduper.py
│   ├── test_ranker.py
│   ├── test_llm.py
│   ├── test_renderer.py
│   └── test_pipeline.py         # full pipeline with mocked HTTP (respx)
└── pyproject.toml
```

---

## Component design

### 1. Config (`config.py`)

Pydantic v2 models loaded from YAML. Secret values are **never** stored in the
config — configs carry the *name* of the environment variable to read.

```
LLMConfig
  provider: Literal["anthropic"] = "anthropic"
  model: str = "claude-haiku-4-5-20251001"     # fast + cheap for ranking tasks
  api_key_env: str = "BLOOMPSCHMAPF_ANTHROPIC_API_KEY"
  max_items_to_rank: int = 30                  # cap sent to LLM (cost control)

AgentConfig
  name: str
  description: str
  query: str
  window_days: int                    # look-back window
  max_results: int = 10               # items shown in final report
  source: SourceConfig
  llm: LLMConfig = LLMConfig()
  notifiers: list[NotifierConfig]

SourceConfig (discriminated union on `type`)
  TavilySourceConfig
    type: Literal["tavily"]
    api_key_env: str = "BLOOMPSCHMAPF_TAVILY_API_KEY"
    max_results: int = 50             # fetch more than shown; LLM will filter

NotifierConfig (discriminated union on `type`)
  TelegramNotifierConfig
    type: Literal["telegram"]
    bot_token_env: str = "BLOOMPSCHMAPF_TELEGRAM_BOT_TOKEN"
    chat_id_env: str   = "BLOOMPSCHMAPF_TELEGRAM_CHAT_ID"
```

### 2. Models (`models.py`)

```
SearchResult
  title: str
  url: str
  snippet: str
  published_at: datetime | None
  source_name: str                    # e.g. "tavily"

RankedItem
  result: SearchResult
  score: float                        # 0..1 (LLM score 0-100, normalised)
  rationale: str                      # LLM's 1-sentence reason for the score
  summary_bullets: list[str]          # LLM's 1-2 bullet summaries of the item

Report
  agent_name: str
  query: str
  window_days: int
  run_at: datetime
  items: list[RankedItem]
  fetched_count: int
  deduped_count: int
```

### 3. Source abstraction (`sources/base.py`)

```python
class BaseSource(ABC):
    @abstractmethod
    def search(self, query: str, window_days: int, max_results: int) -> list[SearchResult]:
        ...
```

`TavilySource` wraps the `tavily-python` client. The `search()` call passes
`days=window_days` (Tavily's built-in recency filter) so results are already
bounded without post-processing.

Adding a new source = subclass `BaseSource`, register in the `source_from_config`
factory in `sources/__init__.py`.

### 4. Deduplicator (`deduper.py`)

Two-pass approach (fast → accurate):

1. **URL normalization**: strip scheme, `www.`, trailing slash, query params.
   Drop exact-match duplicates.
2. **Title similarity**: for remaining items, compare title pairs with
   `difflib.SequenceMatcher`. Drop if ratio > 0.85 (keep first occurrence).

### 5. Recency pre-filter (`ranker.py`)

Before calling the LLM, sort results by recency and keep the top
`llm.max_items_to_rank` to control cost and prompt size. Items without a
`published_at` are placed after dated items.

```python
class RecencyPreFilter:
    def prefilter(self, results: list[SearchResult], max_items: int) -> list[SearchResult]:
        ...  # sort by published_at desc (None last), return [:max_items]
```

### 6. LLM ranker + summarizer (`llm.py`)

`LLMRanker(config: LLMConfig)` with method:

```python
def rank_and_summarize(
    self,
    results: list[SearchResult],
    query: str,
    agent_description: str,
) -> list[RankedItem]:
    ...
```

**Prompt design** (per AGENTS.md guidelines — short, structured, no hallucinated URLs):

```
You are ranking search results for an agent monitoring: "{agent_description}"
Query: "{query}"

Below are {N} search results. For each, output a JSON array entry with:
- "index": the result index (0-based, as given)
- "score": integer 0-100 (100 = most relevant/important)
- "rationale": one sentence explaining the score
- "bullets": list of 1-2 short summary bullets

Only include results that are genuinely relevant (score >= 20).
Do NOT invent or modify URLs. Output ONLY valid JSON, no other text.

Results:
[0] Title: ...  URL: ...  Date: ...  Snippet: ...
[1] ...
```

**Response parsing:**
- Parse JSON array; ignore malformed entries.
- Normalise score: `score / 100.0`.
- Map each entry back to its `SearchResult` by index.
- Sort by score descending.
- Items absent from the LLM response (score < threshold) are dropped.

**Error handling:**
- On API error or JSON parse failure, fall back to returning all items with
  `score=0.5`, `rationale="LLM unavailable"`, `summary_bullets=[]`.
  Log the error prominently. Do not crash.

### 7. Renderer (`renderer.py`)

Produces HTML strings suitable for Telegram. Format per item:

```
1. <b><a href="{url}">{title}</a></b>
   • {bullet_1}
   • {bullet_2}
   📅 {published_date}
   <i>{rationale}</i>
```

Footer (last message):
```
─────────────────────────
🤖 <b>{agent_name}</b> | window: {window_days}d | {run_at:%Y-%m-%d %H:%M UTC}
Fetched: {fetched_count} → deduped: {deduped_count} → shown: {len(items)}
```

The renderer handles Telegram's 4096-char message limit by splitting at item
boundaries. If `summary_bullets` is empty (LLM fallback), falls back to first
sentence of the raw snippet.

### 8. Telegram notifier (`notifiers/telegram.py`)

Uses `httpx`. Bot API endpoint:
`POST https://api.telegram.org/bot{token}/sendMessage`

- Reads token and chat_id from environment variables at send time.
- Sends with `parse_mode=HTML` (more reliable than MarkdownV2 escaping).
- If report is split, sends parts sequentially with a short delay.
- Never logs the token.

### 9. Pipeline (`pipeline.py`)

```python
def run_agent(config: AgentConfig, dry_run: bool = False) -> Report:
    source   = source_from_config(config.source)
    results  = source.search(config.query, config.window_days, config.source.max_results)
    deduped  = Deduplicator().deduplicate(results)
    filtered = RecencyPreFilter().prefilter(deduped, config.llm.max_items_to_rank)
    ranked   = LLMRanker(config.llm).rank_and_summarize(
                   filtered, config.query, config.description)
    ranked   = ranked[: config.max_results]   # trim to report size
    report   = Report(
                   fetched_count=len(results),
                   deduped_count=len(deduped),
                   ...)
    messages = Renderer().render(report)
    if not dry_run:
        for notifier_cfg in config.notifiers:
            notifier_from_config(notifier_cfg).send(messages)
    return report
```

### 10. CLI (`cli.py`)

Typer app, single `run` subcommand:

```
bloompschmapf run agents/example.yaml [--dry-run] [--verbose]
```

`--dry-run` prints the rendered report without sending it.
`--verbose` enables DEBUG logging.

---

## Dependencies

```toml
[project]
dependencies = [
    "pydantic>=2.0",
    "pyyaml>=6.0",
    "httpx>=0.27",
    "typer>=0.12",
    "python-dateutil>=2.9",
    "tavily-python>=0.3",
    "anthropic>=0.40",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "respx>=0.21",          # httpx mock library
    "ruff>=0.4",
    "mypy>=1.10",
    "types-PyYAML",
    "types-python-dateutil",
]
```

---

## Key design decisions & rationale

| Decision | Rationale |
|---|---|
| Tavily as default search provider | Built for agents, native recency filter, Python SDK, free tier |
| Anthropic Claude (Haiku) for ranking | Fast, cheap, reliable JSON output; Haiku is sufficient for scoring/summarizing short snippets |
| Recency pre-filter before LLM | Caps prompt size and API cost; ensures LLM sees only plausibly recent results |
| LLM fallback on error | Agent should always deliver a report, even if degraded; fail-open is better than crashing |
| `httpx` for Telegram (not `python-telegram-bot`) | Minimal dependency, no async required for run-once CLI |
| Pydantic v2 for config | Fast validation, good error messages, discriminated union for provider/notifier types |
| Env var names in config, not values | Secrets never touch the config file; easy to override per-deployment |
| `respx` for test mocking | Designed for `httpx`, clean fixture-based API; also used to mock Anthropic HTTP calls |

---

## Environment variables

| Variable | Used by |
|---|---|
| `BLOOMPSCHMAPF_TAVILY_API_KEY` | TavilySource |
| `BLOOMPSCHMAPF_ANTHROPIC_API_KEY` | LLMRanker |
| `BLOOMPSCHMAPF_TELEGRAM_BOT_TOKEN` | TelegramNotifier |
| `BLOOMPSCHMAPF_TELEGRAM_CHAT_ID` | TelegramNotifier |
