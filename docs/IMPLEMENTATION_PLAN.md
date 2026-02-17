# Implementation Plan

Task list for implementing the MVP as defined in [MVP_SPEC.md](./MVP_SPEC.md).

---

## Task Dependency Graph

```
#1 Set up project scaffolding
 └─► #2 Implement core models and config
      ├─► #3 Implement Tavily source ─────────┐
      ├─► #4 Implement LLM client and ranker  │
      │    └─► #5 Implement LLM summarizer ───┼─► #8 Implement pipeline and CLI
      ├─► #6 Implement renderer ──────────────┤         │
      └─► #7 Implement Telegram notifier ─────┘         │
                                                        ▼
      #9 Write unit tests ◄─────────────────────► #10 Write integration tests
                              │                         │
                              └────────┬────────────────┘
                                       ▼
                        #11 Add example config and e2e test
```

---

## Tasks

### Phase 1: Foundation

#### #1 Set up project scaffolding
- [ ] Create `pyproject.toml` with dependencies:
  - `httpx` (HTTP client)
  - `pydantic` + `pyyaml` (config validation)
  - `typer` (CLI)
  - `anthropic` (LLM)
  - Dev: `pytest`, `respx`, `pytest-mock`, `pytest-cov`, `ruff`, `mypy`
- [ ] Create `src/bloompschmapf/` package structure
- [ ] Create basic CLI entry point
- [ ] Set up `ruff` and `mypy` configs

#### #2 Implement core models and config
**Blocked by**: #1

- [ ] Create `models.py` with `Item` dataclass:
  - `url`, `title`, `snippet`, `published_date`, `source_name`
  - `score`, `rationale`, `summary` (populated later in pipeline)
- [ ] Create `config.py` with `AgentConfig` pydantic model:
  - Validate YAML structure
  - Support env var overrides for secrets
  - Fail fast with actionable errors

---

### Phase 2: Components (parallelizable)

#### #3 Implement Tavily source
**Blocked by**: #2

- [ ] Create `sources/base.py` with `Source` protocol
- [ ] Create `sources/tavily.py`:
  - `search(query: str, days: int) -> list[Item]`
  - Handle timeouts and retries
  - Respect rate limits
  - Map Tavily response to `Item` model

#### #4 Implement LLM client and ranker
**Blocked by**: #2

- [ ] Create `llm/client.py` wrapping `anthropic` SDK
- [ ] Create `llm/ranker.py`:
  - Input: `list[Item]` + query string
  - Prompt LLM for score (0-10) + short rationale per item
  - Dedupe by URL
  - Return sorted list (highest score first)
  - Keep prompt short and structured

#### #5 Implement LLM summarizer
**Blocked by**: #4

- [ ] Create `llm/summarizer.py`:
  - Input: ranked `list[Item]`
  - Prompt LLM for 1-2 bullet summary per item
  - Update `Item.summary` field
  - Never fabricate URLs (summaries reference existing item data only)

#### #6 Implement renderer
**Blocked by**: #2

- [ ] Create `renderer.py`:
  - Format items as Telegram-friendly markdown
  - Include: title, link, bullets, date, relevance score
  - Add metadata footer: agent name, time window, run time, item counts
  - Keep output concise and scannable

#### #7 Implement Telegram notifier
**Blocked by**: #2

- [ ] Create `notifiers/base.py` with `Notifier` protocol
- [ ] Create `notifiers/telegram.py`:
  - Use `httpx` to call Telegram Bot API directly
  - Handle message length limits (split if needed)
  - Never log secrets (bot token)

---

### Phase 3: Integration

#### #8 Implement pipeline and CLI
**Blocked by**: #3, #5, #6, #7

- [ ] Create `pipeline.py` orchestrating:
  1. Load `AgentConfig` from YAML
  2. `source.search()` → fetch items
  3. `ranker.rank()` → score and sort
  4. `summarizer.summarize()` → add summaries
  5. `renderer.render()` → format report
  6. `notifier.send()` → deliver
- [ ] Wire up `cli.py` with typer:
  - `bloompschmapf run <config.yaml>`
  - Structured logging (run id, counts, timing, token usage)

---

### Phase 4: Testing

#### #9 Write unit tests
**Blocked by**: #2, #4, #5, #6

- [ ] Create `tests/unit/` with tests for:
  - Config validation (valid/invalid YAML)
  - `Item` model
  - Deduplication logic
  - Renderer output format
  - Score parsing from LLM response
- [ ] Target >80% coverage on non-IO code

#### #10 Write integration tests
**Blocked by**: #8

- [ ] Create `tests/integration/` with tests for:
  - Full pipeline with mocked HTTP (`respx`) and mocked LLM
  - Tavily client with response fixtures
  - Telegram notifier with mocked API
- [ ] Create `tests/fixtures/` with sample data

#### #11 Add example config and e2e test
**Blocked by**: #8, #9, #10

- [ ] Create `agents/example.yaml` with working config
- [ ] Create `tests/e2e/test_full_run.py`:
  - Marked `@pytest.mark.e2e` (skipped by default)
  - Runs full pipeline with real APIs
  - Documents required env vars
- [ ] Update `README.md` with usage instructions
