# AGENTS.md

## 📦 Project Overview

**bloompschmapf** is a small Python framework for running a *scheduled “search + summarize + notify” agent*.

A bloompschmapf agent is configured to:
1) search online for new items matching a topic (e.g., songs in a niche genre, calls for papers in a field, lawsuits in certain jurisdictions) from the last **X days**,
2) deduplicate and rank results by estimated relevance/importance,
3) produce a concise report with linked references,
4) deliver the report via **email** and/or **Telegram**.

- **Primary Language:** Python 3.11+ (preferred), 3.10 acceptable
- **Core concepts:** Agent config, Sources (search providers), Extractors, Ranker, Summarizer, Report renderer, Notifiers, Scheduler
- **Runtime modes:** CLI (run once), daemon scheduler (continuous), container-friendly
- **No secrets in repo:** all credentials via env vars or external secret manager

______________________________________________________________________

## 🗂️ Repository Structure

Typical layout (adjust if your repo differs; keep this file in sync):

<TBA>

______________________________________________________________________

## 🚦 Quick Start

### 1) Install (dev)

<TBA>

2) Configure an agent

<TBA>

3) Run once

<TBA>

4) Run on a schedule

<TBA>


⸻

🧑‍💻 Development Guidelines

1) Code Style & Quality
	•	Python 3.11+ preferred.
	•	Full type annotations for new code.
	•	Lint/format: ruff (format + lint).
	•	Type check: mypy.
	•	Tests: pytest.

Recommended commands:

ruff format .
ruff check .
mypy src/
pytest

2) Configuration Rules
	•	Agent configs are treated as stable API.
	•	Backward compatible changes only unless explicitly versioned.
	•	Support env var overrides for all secrets and deployment-specific values.
	•	Validate configs early; fail fast with actionable errors.

3) Networking & Scraping
	•	Always use timeouts and retries.
	•	Identify a clear User-Agent.
	•	Respect robots.txt where feasible and do not implement brittle scraping as the default.
	•	Prefer RSS/APIs over HTML scraping when available.
	•	Cache fetches during a run to avoid repeated requests.

4) Ranking

Ranking should be deterministic and explainable by default:
	•	recency score (within window)
	•	query match score (title/snippet)
	•	source authority heuristic (domain allowlist/denylist, or simple signals)
	•	novelty (dedupe + similarity suppression)

If an LLM is used:
	•	keep the prompt short and structured
	•	require a numeric score + short rationale
	•	never hallucinate URLs; URLs must come from sources

5) Summarization
	•	Output must be concise and link-first.
	•	Include dates when available (publish date, deadline, etc.).
	•	Each item should include: title, link, 1–2 bullets, optional key date(s).

6) Notification
	•	Email: support both plain text and HTML where possible.
	•	Telegram: keep within message length limits; split if needed.
	•	Never log secrets (SMTP passwords, bot tokens).

7) Observability
	•	Structured logs (JSON optional) with run id, agent name, timing, counts.
	•	Metrics (optional): fetched, deduped, ranked, delivered, failures.

⸻

🧩 Agent Code Integration

Key entry points
<TBA>

Adding a new source provider
<TBA>

Adding a new notifier
<TBA>

⸻

🔐 Secrets & Configuration
	•	Never commit secrets.
	•	Prefer .env locally; environment variables in CI/deploy.
	•	Common env vars (example naming):

BLOOMPSCHMAPF_SMTP_HOST
BLOOMPSCHMAPF_SMTP_PORT
BLOOMPSCHMAPF_SMTP_USER
BLOOMPSCHMAPF_SMTP_PASS
BLOOMPSCHMAPF_TELEGRAM_BOT_TOKEN
BLOOMPSCHMAPF_TELEGRAM_CHAT_ID
BLOOMPSCHMAPF_WEBSEARCH_API_KEY


⸻

🧪 Testing
	•	Unit tests for config validation, dedupe, ranking determinism, and rendering.
	•	Integration smoke test for pipeline with mocked HTTP.
	•	No live network calls in tests by default.

Suggested targets:

pytest -q
pytest -q -k pipeline


⸻

🏷️ Branching & Workflow
	•	Work in feature branches: feature/<desc> / bugfix/<desc>
	•	Keep PRs small and test-backed.
	•	Update README + example config when adding user-facing behavior.

⸻

🧠 Additional Instructions
	•	This file is committed to the repository and must not include secrets.
	•	Prefer explicit, readable code over cleverness.
	•	Never fabricate links or references in reports.
	•	Always include a “run metadata” footer in reports (agent name, window, run time, counts).
	•	Keep outputs concise; optimize for scanability.

⸻

🧭 Quick Reference

Task	Command
Install (dev)	pip install -e ".[dev]"
Run once	bloompschmapf run agents/x.yaml
Serve on schedule (optional)	bloompschmapf serve ...
Tests	pytest
Lint	ruff check .
Format	ruff format .
Type check	mypy src/

