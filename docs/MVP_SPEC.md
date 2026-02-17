# MVP Spec

Minimal viable bloompschmapf: one agent, one source (Tavily), one notifier (Telegram), LLM-powered ranking/summarization, CLI-only.

---

## Core Abstractions

```
AgentConfig      - YAML config defining: query, source, notifier, time window
Source           - Protocol: search(query, days) -> list[Item]
Item             - Dataclass: url, title, snippet, published_date, source_name, score, rationale
Ranker           - LLM-based: scores items with numeric score + short rationale
Summarizer       - LLM-based: produces 2-3 bullet summary of the entire result set
Notifier         - Protocol: render(items) -> formatted_msg, send(msg) -> None (coupled rendering)
Pipeline         - Orchestrates: source.search() -> rank() -> summarize() -> notifier.render() -> notifier.send()
```

---

## Data Flow

```
┌─────────────┐     ┌────────┐     ┌──────────┐     ┌────────────┐     ┌───────────────────┐
│ AgentConfig │────▶│ Source │────▶│  Ranker  │────▶│ Summarizer │────▶│     Notifier      │
└─────────────┘     └────────┘     └──────────┘     └────────────┘     │ (render + send)   │
                         │              │                  │           └───────────────────┘
                    list[Item]    scored items       2-3 bullet              │
                                  + rationales        summary           formatted_msg
                                                                        for channel
```

1. Log verbose startup info (config loaded, run id, query, time window)
2. Load `AgentConfig` from YAML
3. `Source.search(query, days)` → fetch items from Tavily
4. `Ranker.rank(items, query)` → LLM scores each item (0-10) + short rationale, dedupe, sort
5. `Summarizer.summarize(ranked_items)` → LLM produces 2-3 bullet summary of entire list
6. `Notifier.render(items, summary, config)` → channel-specific formatted message
7. `Notifier.send(msg)` → deliver via Telegram bot

---

## MVP Scope

### In Scope
- **1 source**: Tavily web search API
- **1 notifier**: Telegram bot (with coupled rendering)
- **CLI only**: `bloompschmapf run config.yaml`
- **LLM ranking**: score (0-10) + rationale per item, dedupe by URL
- **LLM summarization**: 2-3 bullet summary of the entire result list (not per-item)
- **YAML config**: agent name, query, source config, notifier config, time window (days), LLM config
- **Telegram markdown output**: title, link, date, relevance score, holistic summary
- **Structured logging**: verbose startup logs (config, run id, query), item counts, LLM token usage
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
│       │   └── summarizer.py # LLM-based summarization (holistic)
│       └── notifiers/
│           ├── __init__.py
│           ├── base.py      # Notifier protocol (render + send)
│           └── telegram.py  # Telegram notifier (renders markdown + sends)
├── tests/
│   ├── unit/                # Fast, isolated tests
│   │   ├── test_config.py
│   │   ├── test_models.py
│   │   ├── test_ranker.py
│   │   ├── test_summarizer.py
│   │   └── test_telegram_notifier.py  # Notifier render + send
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
  - Notifier render output format (Telegram markdown)
  - Score parsing from LLM response

### Integration Tests (`tests/integration/`)
- **What**: Component interactions
- **Mocking**: HTTP via `respx`, LLM responses via fixtures
- **Speed**: Medium (<10s total)
- **Examples**:
  - Pipeline: source → ranker → summarizer → notifier (render + send)
  - Tavily client with mocked responses
  - Telegram notifier render + send with mocked API

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
- [ ] Verbose startup logs (config loaded, run id, query, time window)
- [ ] Fetches results from Tavily API
- [ ] LLM ranks items with score + rationale
- [ ] LLM summarizes result list (2-3 bullets for entire set)
- [ ] Notifier renders channel-specific output (Telegram markdown)
- [ ] Sends via Telegram bot
- [ ] Fails fast with clear error on bad config or missing env vars
- [ ] Unit tests pass with >80% coverage
- [ ] Integration tests pass with mocked HTTP/LLM
- [ ] E2E test documented and runnable manually
