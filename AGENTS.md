# CLAUDE.md

## Project

SEO article generator plus AEO and brand-monitor utilities. FastAPI backend + Typer CLI. The core pipeline researches topics, analyzes competitors, and generates optimized articles with quality scoring, content humanization, and schema markup.

## Stack

- **Python 3.12+**, FastAPI, SQLAlchemy async (PostgreSQL prod, SQLite tests), Pydantic v2
- **LLM**: Quad backend via `app/llm.py` — Anthropic API (with tool use), Claude Agent SDK (`max_turns=50`), OpenAI Codex SDK, Google Gemini (with tool use)
- **CLI**: Typer + Rich (`autoseo` project script; use `uv run autoseo ...` in a synced checkout)
- **Cache**: Redis (`app/cache.py`)
- **Deps**: `textstat`, `google-genai`, `openai`, `openai-codex-sdk`, `firecrawl-py`, `playwright`, `spacy`, `voyageai`, `beautifulsoup4`, `sse-starlette`
- **Lint**: `ruff` (line-length=100, rules: E/F/I/N/W), `pyright` strict (`pyrightconfig.json`; third-party noise rules disabled — see config)
- **Tests**: pytest + pytest-asyncio (`asyncio_mode="auto"`), **398 tests** across 24 files

## Architecture

### Pipeline (`app/article/pipeline.py`)

State machine: `PENDING → RESEARCHING → PLANNING → GENERATING → SCORING → REVIEWING → [EDITING] → COMPLETED | FAILED`

- **Single-call generation**: Full article + FAQ in one `generate_text` call, parsed via `_parse_article_markdown`, then scrubbed via `scrub_article()`
- **Planning step**: Two-phase `planning_step` — multi-provider analysis (council fans out competitor analysis in parallel), then single-provider outline with embedded `ArticleBrief`
- **Brand voice**: Optional `BrandVoice` injected into outline/generate/edit prompts via `format_brand_voice()`
- **Hybrid scoring**: 7 algorithmic + 6 LLM = 13 dimensions, weighted average (`word_count_target` at 2x via `DIMENSION_WEIGHTS`)
- **Token tracking**: `LlmClient._usage` accumulates `TokenUsage` per call; usage extraction via `_record_sdk_usage`, `_record_gemini_usage`, `_record_codex_usage`; pipeline drains telemetry per step into `job.usage_data`
- **Multi-provider council**: `get_llm_council()` returns configured providers as equal peers (Anthropic if `ANTHROPIC_API_KEY`, Codex if `OPENAI_CODEX=true`, Gemini if `GOOGLE_API_KEY` — note: standard OpenAI API alone does **not** join the council); analysis, scoring, and review fan out in parallel, then merge/average results
- **Writer model routing**: `_resolve_article_writer()` routes article drafting and editing to `GEMINI_WRITING_MODEL` (default `gemini-3-pro-preview`) when `GOOGLE_API_KEY` is set, regardless of the primary LLM backend
- **Event system**: `job.events_data` stores structured events (LLM calls, scrub stats, timing, cache hits). Cleared on completion unless `PERSIST_EVENTS=true`
- **Edit loop**: Data-driven via `_EDIT_CYCLE` + `_run_step_safely()`; edit → re-score → re-review (capped at `max_revisions`)
- **Sub-step visibility**: Generate and score update `current_step` with sub-steps like `generating:article` and `scoring:llm`
- **Word count enforcement**: Prompt constraint (±20% target), steeper scoring curve on overshoot, and trim instructions in edit prompts

### Scoring dimensions

| Type | Dimensions | Source |
|------|------------|--------|
| Algorithmic | keyword_usage, heading_structure, word_count_target, readability_metrics, humanity, keyword_distribution, differentiation_delivery | `app/article/scorer.py` |
| LLM | content_depth, differentiation, accuracy, consistency, readability, actionability | 3 parallel `_ScorePair` calls |

### Content scrubber (`app/article/scrubber.py`)

Post-processes articles after generation and editing. Returns `(ArticleContent, ScrubStats)`:

- **Modifies**: zero-width Unicode strip, AI filler opener removal, collapsed list normalization, unclosed code fence repair, long paragraph splitting (>6 sentences), spacing cleanup
- **Counts only**: em-dashes/double-hyphens, AI-favored words

### SEO outputs (`app/article/schema.py`)

- **JSON-LD**: Article + FAQPage schema markup, computed lazily in `build_result()`
- **Snippet detection**: List, table, definition, Q&A opportunities
- **Meta options**: 5 title tags + 5 meta descriptions via `SeoMetaOptions`

### AEO content scorer & fan-out (`app/aeo/`)

- **AEO scorer**: `POST /api/aeo/analyze` — 3 checks (direct answer, h-tag hierarchy, readability), each max 20 points, aggregated to 0-100 with band labels
- **Query fan-out**: `POST /api/aeo/fanout` — LLM decomposes a query into 10-15 sub-queries across 6 types, optional gap analysis via VoyageAI embeddings
- **Content parser** (`parser.py`): URL fetch via httpx or Firecrawl-backed full-page extraction path, HTML parsing, boilerplate stripping
- **Checks** (`checks.py`): spaCy `en_core_web_sm`, textstat
- **Fan-out** (`fanout.py`): `LlmClient.generate_structured()` for sub-query generation, Voyage `voyage-4-large` embeddings for gap analysis
- **Tests**: 49 unit tests across `test_aeo.py` and `test_fanout.py`

### Brand monitor (`app/brand/`)

- **Endpoints**: `POST /api/brand-monitor/analyze` (blocking), `POST /api/brand-monitor/analyze/stream` (SSE), `GET /api/brand-monitor/analyses` (history), `GET /api/brand-monitor/analyses/{id}`
- **Two modes**: Single-query (legacy: `query` param) or auto-discovery (new: `url` param → scrape → competitor ID → multi-prompt generation)
- **Fetch layer**: API mode via `app/brand/fetcher.py` (OpenAI, Perplexity, Gemini, Anthropic with web search), browser mode via `app/brand/browser_fetcher.py` (ChatGPT, Perplexity, Gemini, Grok)
- **Detection layer**: `detection.py` provides deterministic regex brand name matching (variations, negative context, confidence scoring) as fallback to LLM analysis
- **Scoring layer**: `scoring.py` computes visibility score, share of voice, sentiment score, position score, weighted overall score; competitor rankings and provider comparison matrix
- **Discovery layer**: `discovery.py` scrapes brand URL via Firecrawl, extracts `CompanyInfo` via LLM, identifies competitors, generates ~14 prompts across 4 categories (ranking, comparison, alternatives, recommendations)
- **Persistence**: `store.py` — `BrandAnalysis` ORM model with JSONB payload, auto-saved after each analysis
- **SSE streaming**: `stream.py` orchestrates the full pipeline with real-time events: scraping → identifying-competitors → generating-prompts → fetching-responses → analyzing → scoring → finalizing
- **Operational split**: Fetch and analysis are separate; a successful fetch can still end in `503` if the analyzer LLM backend fails
- **Playwright loading**: Browser mode is lazy-imported so app startup/test collection does not require Playwright unless that path is used

### Database bootstrap (`app/db.py`)

- `init_db()` wraps `Base.metadata.create_all()` in a Postgres transaction-scoped advisory lock
- Fresh `uvicorn --workers 2` startup is safe against an empty Postgres database

## Key files

| Path | Purpose |
|------|---------|
| `app/article/pipeline.py` | Pipeline runner, step functions, markdown parser, merge logic |
| `app/article/prompts.py` | Prompt templates + `format_brand_voice`, `meta_options_prompt` |
| `app/article/models.py` | Pydantic models (BrandVoice, SeoMetaOptions, KeywordDistribution, etc.) |
| `app/article/scorer.py` | 7 algorithmic scoring functions + `full_text()` helper |
| `app/article/scrubber.py` | Content post-processor |
| `app/article/schema.py` | JSON-LD generation, snippet opportunity detection |
| `app/article/constants.py` | Shared regex/patterns: AI filler phrases, sentence regex, zero-width RE, fence toggling |
| `app/article/tools.py` | Research tools for LLM tool-use in the planning step |
| `app/serp/fetcher.py` | Firecrawl Python SDK integration; use snake_case SDK kwargs like `only_main_content`, not raw API camelCase |
| `app/serp/client.py` | SERP provider abstraction |
| `app/brand/routes.py` | Brand monitor endpoints (analyze, stream, history) |
| `app/brand/fetcher.py` | API-mode platform fetches (OpenAI, Perplexity, Gemini, Anthropic) |
| `app/brand/browser_fetcher.py` | Browser-mode platform fetches; raises helpful `LlmError` if Playwright or Chromium is missing |
| `app/brand/analyzer.py` | Structured brand analysis + aggregate summary + detection fallback |
| `app/brand/detection.py` | Deterministic regex brand name matching with variations and negative context |
| `app/brand/scoring.py` | Visibility scoring: visibility, share of voice, sentiment, position, overall |
| `app/brand/discovery.py` | URL scraping, competitor identification, multi-category prompt generation |
| `app/brand/store.py` | Brand analysis persistence — `BrandAnalysis` ORM model + CRUD |
| `app/brand/stream.py` | SSE streaming orchestrator for real-time brand analysis progress |
| `app/brand/gather.py` | Partial-success gathering utility for parallel fetches |
| `app/aeo/checks.py` | Direct answer, h-tag hierarchy, readability checks |
| `app/aeo/fanout.py` | Fan-out prompt, sub-query generation, gap analysis |
| `app/aeo/parser.py` | URL fetch + HTML parsing/boilerplate stripping (BeautifulSoup) |
| `app/aeo/routes.py` | AEO analyze and fan-out endpoints |
| `app/aeo/store.py` | AEO persistence — `AeoAnalysis` ORM model + Redis cache for fetches/fan-out |
| `app/db.py` | Engine/session setup + advisory-lock init |
| `app/errors.py` | Error hierarchy (`SeoAgentError` → `JobNotFoundError`/`StepError`/`LlmError`/`SerpError`/`ContentFetchError`) + HTTP helpers |
| `app/job/service.py` | Job CRUD, `claim_job_for_resume`, persisted resume behavior |
| `app/job/routes.py` | Job API endpoints (CRUD + resume) |
| `app/cli.py` | Typer CLI: generate, status, result, list, watch, resume, export, brand, brand-history, aeo, fanout |
| `app/main.py` | App lifespan, DB init, Redis connect, orphaned-job recovery |
| `app/cache.py` | Redis cache client with graceful degradation |
| `app/config.py` | Settings via pydantic-settings |

## Commands

```bash
uv sync --extra dev --python 3.12
uv run ruff check app/ tests/
uvx pyright
uv run pytest tests/ -x -q                      # Full suite (398 tests; requires en_core_web_sm)
uv run pytest tests/ -x -q -k "not e2e"         # Unit tests only (346 tests; skip E2E)
uv run pytest tests/test_pipeline_e2e.py -x -v  # E2E pipeline (~13m, needs GOOGLE_API_KEY)
uv run pytest tests/test_brand_e2e.py -x -v     # E2E brand (~7m, needs Google+Perplexity+Firecrawl)
uv run pytest tests/test_aeo_e2e.py -x -v       # E2E AEO (~15s, Firecrawl optional)
uv run pytest tests/test_fanout_e2e.py -x -v    # E2E fanout (~1m, needs Google+Voyage)
uv run python -m spacy download en_core_web_sm  # Required for AEO checks/full suite
uv run playwright install chromium              # Required for brand browser mode
uv run uvicorn app.main:app --workers 2         # Agent SDK blocks the event loop
uv run autoseo generate "topic" --brand-voice brand.json
uv run autoseo -v generate "topic"
uv run autoseo status <id>
uv run autoseo result <id>
uv run autoseo list
uv run autoseo watch <id>
uv run autoseo resume <id>
uv run autoseo export <id> article.md
uv run autoseo aeo tests/fixtures/article_good.html --json
uv run autoseo fanout "best CRM for startups" --content https://example.com --json
uv run autoseo brand "Notion" "best note-taking app" --json
uv run autoseo brand "Notion" --url https://notion.so --mode api --json
uv run autoseo brand-history
uv run autoseo brand-history --brand "Notion"
```

## Config (env vars / .env)

- `ANTHROPIC_API_KEY` — Anthropic API key (if set, uses API backend)
- `GOOGLE_API_KEY` — Enables Gemini in the provider council and brand API fetches
- `OPENAI_API_KEY` — Enables OpenAI API fetches for brand monitoring
- `PERPLEXITY_API_KEY` — Enables Perplexity API fetches for brand monitoring
- `LLM_MODEL` — Anthropic model (default `claude-sonnet-4-6`)
- `GEMINI_MODEL` — Default Gemini model for non-writing tasks (default `gemini-3-flash-preview`)
- `GEMINI_WRITING_MODEL` — Gemini model for article draft + edit generation (default `gemini-3-pro-preview`)
- `OPENAI_MODEL` — OpenAI model (default `o3-mini`)
- `OPENAI_CODEX` — Set `true` to use Codex SDK backend (ChatGPT subscription)
- `SERP_PROVIDER` — `mock` (default) or `serpapi`
- `SERPAPI_KEY` — Required if `SERP_PROVIDER=serpapi`
- `FIRECRAWL_API_KEY` — Firecrawl API key for SERP content fetching and URL-backed fan-out analysis
- `VOYAGE_API_KEY` — VoyageAI API key for fan-out gap-analysis embeddings
- `VOYAGE_EMBEDDING_MODEL` — Voyage embedding model (default `voyage-4-large`)
- `CONTENT_FETCH_TOP_N` — Number of top SERP results to fetch content for (default `10`)
- `DATABASE_URL` — PostgreSQL connection string
- `REDIS_URL` — Redis connection string
- `QUALITY_THRESHOLD` — Minimum overall score (default `0.8`)
- `MAX_REVISIONS` — Edit-loop cap (default `10`)
- `AEO_SIMILARITY_THRESHOLD` — Cosine similarity threshold for fan-out gap analysis (default `0.72`)
- `DEBUG` — Enable SQLAlchemy debug logging
- `BRAND_MONITOR_MAX_PROMPTS` — Max auto-generated brand prompts (default `14`)
- `BRAND_MONITOR_BATCH_SIZE` — Concurrent prompt analysis batch size (default `3`)
- `PERSIST_EVENTS` — Keep pipeline events after completion (default `false`)

## DB migration

Existing databases still need manual column addition for newer job payload fields:

```sql
ALTER TABLE jobs ADD COLUMN IF NOT EXISTS meta_options_data JSON;
ALTER TABLE jobs ADD COLUMN IF NOT EXISTS brand_voice_data JSON;
ALTER TABLE jobs ADD COLUMN IF NOT EXISTS usage_data JSON;
ALTER TABLE jobs ADD COLUMN IF NOT EXISTS events_data JSON;

UPDATE jobs SET status='planning' WHERE status IN ('analyzing', 'outlining');
```

Tables `aeo_analyses` (ORM in `app/aeo/store.py`) and `brand_analyses` (ORM in `app/brand/store.py`) auto-create alongside `jobs` on fresh databases.

Fresh databases auto-create on startup, and Postgres startup is serialized with an advisory lock so multi-worker boot is safe.

## CLI internals

- **Verbose mode** (`--verbose/-v`): renders pipeline events streamed via `events_data` polling
- **`ctx.obj`**: dict `{"url": ..., "verbose": ...}` — accessed via `_api_url(ctx)` and `_is_verbose(ctx)`
- **Polling**: `_poll_job` uses a long httpx timeout and retries on `ReadTimeout`
- **Sub-step display**: `current_step` with `:` separator renders as `Generate (article)...`
- **Resume**: `409` auto-switches to watch mode; `400` prints the API error message
- **Watch**: reconnects to running jobs and prints completed/failed state immediately for finished jobs
- **Brand**: CLI only surfaces the top-level `detail.message` from a `503`; inspect the API response for the nested analyzer failure

## Known issues / gotchas

- **Agent SDK blocks the event loop**: Long Claude Agent SDK calls can block for minutes; keep `uvicorn --workers 2`
- **Orphaned jobs are failed on startup, not auto-resumed**: `app.main._recover_orphaned_jobs()` marks active jobs as `failed` with `Recovered: server restarted mid-pipeline`; users then `resume`
- **AEO/full tests require a separate spaCy model install**: `en_core_web_sm` is not installed by `uv sync`
- **Brand defaults to browser mode**: The default browser target set is ChatGPT, Perplexity, Gemini, and Grok
- **Brand browser mode needs Chromium binaries**: The Playwright Python package is in project deps, but `playwright install chromium` is still required
- **Brand API fetches use provider SDKs directly**: `app/brand/fetcher.py` uses `openai.AsyncOpenAI` for both OpenAI and Perplexity-compatible APIs
- **`claim_job_for_resume` requires `session.refresh()`**: Without it, `_job_to_response` can trigger `MissingGreenlet`
- **`from google import genai` needs `# type: ignore[reportAttributeAccessIssue]`**: The `google` namespace package doesn't expose `genai` in stubs; all `google.genai` imports require inline suppression
- **pyright runs outside venv via `uvx`**: Third-party type noise is disabled in `pyrightconfig.json` (`reportMissingImports`, `reportMissingTypeStubs`, `reportUntypedBaseClass`, `reportUntypedFunctionDecorator`, `reportUntypedClassDecorator`, `reportMissingParameterType`, `reportPrivateUsage`)

## Testing patterns

### Unit tests (346 tests, 19 files)

- In-memory SQLite via `create_async_engine("sqlite+aiosqlite:///:memory:")`
- Pipeline tests mock `generate_text`/`generate_structured` with `_smart_generate_structured`
- Pipeline tests must include `SeoMetaOptions: _make_meta_options()` in model maps
- Pipeline tests patch `get_llm_council` (return `[mock_llm]`) and `settings` (control threshold)
- Mock LLM requires `drain_usage = MagicMock(return_value=[])` and `drain_call_log = MagicMock(return_value=[])`
- Threshold for edit-loop tests: use `0.6` because `word_count_target` is double-weighted
- Single-provider council dimension count assertion: `9 = 7` algorithmic + `2` merged LLM

### E2E tests (52 tests, 4 files) — real API calls

| File | Tests | Time | Keys required |
|------|-------|------|---------------|
| `test_pipeline_e2e.py` | 9 | ~13m | `GOOGLE_API_KEY` |
| `test_brand_e2e.py` | 18 | ~7m | `GOOGLE_API_KEY`, `PERPLEXITY_API_KEY`, `FIRECRAWL_API_KEY` |
| `test_aeo_e2e.py` | 17 | ~15s | `FIRECRAWL_API_KEY` (URL tests only) |
| `test_fanout_e2e.py` | 8 | ~1m | `GOOGLE_API_KEY`, `VOYAGE_API_KEY` |

- Shared helpers in `conftest.py`: `write_example()`, `write_log()`, skip markers (`skip_no_llm`, `skip_no_fetch`, `skip_no_firecrawl`, `skip_no_voyage`)
- Each test file has `EXAMPLES_DIR` (showcase) + `INTERNALS_DIR` (test artifacts in `_internals/`)
- Generated articles write to `examples/articles/{slug}.md` via `_write_article()`; pipeline JSON stays in `examples/pipeline/`
- Pipeline E2E uses `MockSerpProvider` + real Gemini LLM (no SERP key needed)
- `Job.status` is a plain string after SQLite `session.refresh()` — compare with `str(job.status)`, not `JobStatus` enum
- Voyage gap analysis threshold (0.72) is strict; short content often yields 0% coverage — assert on valid scores, not coverage counts
