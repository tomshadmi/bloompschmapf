# MVP Spec

Minimal viable bloompschmapf: one agent, one source (Tavily), one notifier (Telegram), LLM-powered ranking/summarization, CLI-only.

---

## Core Abstractions

```
AgentConfig      - YAML config defining: query, source, notifier, time window
Source           - Protocol: search(query, days) -> list[Item]
Item             - Dataclass: url, title, snippet, published_date, source_name, score, rationale
Ranker           - LLM-based: scores items with numeric score + short rationale
Summarizer       - LLM-based: produces concise per-item summaries
Renderer         - Formats ranked + summarized items into Telegram-friendly report
Notifier         - Protocol: send(report) -> None
Pipeline         - Orchestrates: source.search() -> rank() -> summarize() -> render() -> notifier.send()
```

---

## Data Flow

```
┌─────────────┐     ┌────────┐     ┌──────────┐     ┌────────────┐     ┌───────────┐
│ AgentConfig │────▶│ Source │────▶│  Ranker  │────▶│ Summarizer │────▶│ Renderer  │
└─────────────┘     └────────┘     └──────────┘     └────────────┘     └───────────┘
                         │              │                  │                  │
                    list[Item]    scored items      summarized items     report: str
                                  + rationales                                │
                                                                              ▼
                                                                       ┌───────────┐
                                                                       │ Notifier  │
                                                                       │(Telegram) │
                                                                       └───────────┘
```

1. Load `AgentConfig` from YAML
2. `Source.search(query, days)` → fetch items from Tavily
3. `Ranker.rank(items, query)` → LLM scores each item (0-10) + short rationale, dedupe, sort
4. `Summarizer.summarize(ranked_items)` → LLM produces 1-2 bullet summaries per item
5. `Renderer.render(items, config)` → Telegram-friendly markdown report with metadata footer
6. `Notifier.send(report)` → deliver via Telegram bot

---

## MVP Scope

### In Scope
- **1 source**: Tavily web search API
- **1 notifier**: Telegram bot
- **CLI only**: `bloompschmapf run config.yaml`
- **LLM ranking**: score (0-10) + rationale per item, dedupe by URL
- **LLM summarization**: 1-2 bullet summary per item
- **YAML config**: agent name, query, source config, notifier config, time window (days), LLM config
- **Telegram markdown output**: title, link, bullets, date, relevance score
- **Structured logging**: stdout, run id, item counts, LLM token usage
- **Tests**: unit, integration, and e2e (see Testing section)

### Out of Scope (for MVP)
- Multiple sources per agent
- Daemon/scheduler mode
- Email notifier
- Web UI or API server
- Persistence/history (seen items tracking)
- RSS/feed sources
- Extractors (full-page content extraction)
- Parallel fetching
- Rate limiting sophistication
- Streaming LLM responses

---

## Tech Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Language | Python 3.11+ | Type hints, match statements |
| HTTP | `httpx` | Async-ready, timeouts built-in |
| Config | `pyyaml` + `pydantic` | Validation with clear errors |
| CLI | `typer` | Simple, typed args |
| LLM | `anthropic` SDK (Claude) | Structured output, good at ranking tasks |
| Search | Tavily API via `httpx` | Simple API, good search quality, free tier |
| Telegram | `httpx` direct API calls | Avoid heavy bot frameworks |
| Testing | `pytest` + `respx` + `pytest-mock` | Mock HTTP and LLM calls |
| Lint/Format | `ruff` | Fast, all-in-one |
| Types | `mypy` | Strict mode |

---

## Minimal File Structure

```
bloompschmapf/
├── src/
│   └── bloompschmapf/
│       ├── __init__.py
│       ├── cli.py           # CLI entry point
│       ├── config.py        # AgentConfig + validation
│       ├── models.py        # Item dataclass
│       ├── pipeline.py      # Orchestration
│       ├── sources/
│       │   ├── __init__.py
│       │   ├── base.py      # Source protocol
│       │   └── tavily.py    # Tavily search implementation
│       ├── llm/
│       │   ├── __init__.py
│       │   ├── client.py    # LLM client wrapper
│       │   ├── ranker.py    # LLM-based ranking
│       │   └── summarizer.py # LLM-based summarization
│       ├── renderer.py      # Telegram markdown output
│       └── notifiers/
│           ├── __init__.py
│           ├── base.py      # Notifier protocol
│           └── telegram.py  # Telegram bot notifier
├── tests/
│   ├── unit/                # Fast, isolated tests
│   │   ├── test_config.py
│   │   ├── test_models.py
│   │   ├── test_ranker.py
│   │   ├── test_summarizer.py
│   │   └── test_renderer.py
│   ├── integration/         # Component interaction, mocked HTTP/LLM
│   │   ├── test_pipeline.py
│   │   ├── test_tavily.py
│   │   └── test_telegram.py
│   ├── e2e/                 # Full pipeline, real APIs (optional, slow)
│   │   └── test_full_run.py
│   ├── fixtures/            # Shared test data
│   │   ├── sample_items.py
│   │   └── sample_configs.py
│   └── conftest.py          # Shared fixtures, mocks
├── agents/                  # Example configs
│   └── example.yaml
├── pyproject.toml
└── README.md
```

---

## Example Config (MVP)

```yaml
name: ml-papers-weekly
query: "machine learning transformers research papers"
days: 7
max_items: 10

source:
  type: tavily
  # API key from env: BLOOMPSCHMAPF_TAVILY_API_KEY

llm:
  provider: anthropic
  model: claude-sonnet-4-20250514
  # API key from env: ANTHROPIC_API_KEY

notifier:
  type: telegram
  # From env: BLOOMPSCHMAPF_TELEGRAM_BOT_TOKEN, BLOOMPSCHMAPF_TELEGRAM_CHAT_ID
```

---

## Testing Strategy

### Unit Tests (`tests/unit/`)
- **What**: Individual functions/classes in isolation
- **Mocking**: All external dependencies (HTTP, LLM)
- **Speed**: Fast (<1s total)
- **Examples**:
  - Config validation (valid/invalid YAML)
  - Item deduplication logic
  - Renderer output format
  - Score parsing from LLM response

### Integration Tests (`tests/integration/`)
- **What**: Component interactions
- **Mocking**: HTTP via `respx`, LLM responses via fixtures
- **Speed**: Medium (<10s total)
- **Examples**:
  - Pipeline: source → ranker → summarizer → renderer
  - Tavily client with mocked responses
  - Telegram notifier with mocked API

### E2E Tests (`tests/e2e/`)
- **What**: Full pipeline with real APIs
- **Mocking**: None (real Tavily, real LLM, test Telegram chat)
- **Speed**: Slow, costs money
- **Run**: Manual or CI with `pytest -m e2e` (skipped by default)
- **Examples**:
  - `bloompschmapf run agents/test.yaml` succeeds end-to-end

### Test Commands

```bash
# All fast tests (unit + integration)
pytest

# Unit only
pytest tests/unit/

# Integration only
pytest tests/integration/

# E2E (requires API keys, costs money)
pytest tests/e2e/ -m e2e

# With coverage
pytest --cov=bloompschmapf --cov-report=term-missing
```

---

## Environment Variables (MVP)

```bash
# Required
ANTHROPIC_API_KEY=sk-ant-...
BLOOMPSCHMAPF_TAVILY_API_KEY=tvly-...
BLOOMPSCHMAPF_TELEGRAM_BOT_TOKEN=123456:ABC...
BLOOMPSCHMAPF_TELEGRAM_CHAT_ID=123456789
```

---

## Done When

- [ ] `bloompschmapf run agents/example.yaml` executes end-to-end
- [ ] Fetches results from Tavily API
- [ ] LLM ranks items with score + rationale
- [ ] LLM summarizes top items (1-2 bullets each)
- [ ] Outputs Telegram-friendly markdown report
- [ ] Sends via Telegram bot
- [ ] Fails fast with clear error on bad config or missing env vars
- [ ] Unit tests pass with >80% coverage
- [ ] Integration tests pass with mocked HTTP/LLM
- [ ] E2E test documented and runnable manually
