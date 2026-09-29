# AI Test Generator

[![pytest](https://github.com/TanishMhalsekar01/AI-test-generator/actions/workflows/pytest.yml/badge.svg)](https://github.com/TanishMhalsekar01/AI-test-generator/actions/workflows/pytest.yml)
[![Deploy to Render](https://github.com/TanishMhalsekar01/AI-test-generator/actions/workflows/live-check.yml/badge.svg)](https://github.com/TanishMhalsekar01/AI-test-generator/actions/workflows/live-check.yml)

AI Test Generator reviews source code in any programming language, generates unit tests for it, executes those tests in a sandbox, and reports defects with exact line numbers. It also validates API contracts (OpenAPI/Swagger, GraphQL, JSON/YAML) including live checks against running servers.

**Production URL:** <https://aitestge.stream>  
**Model:** Google Gemini `gemini-2.5-pro` (with automatic fallback to Flash models)  
**Stack:** FastAPI · SQLAlchemy (Supabase Postgres / SQLite) · vanilla HTML/CSS/JS · Docker on Render

---

## Table of Contents

- [What It Does](#what-it-does)
- [How a Run Works](#how-a-run-works)
- [Language Support](#language-support)
- [API & Spec Checks](#api--spec-checks)
- [Project Layout](#project-layout)
- [Configuration](#configuration)
- [Local Development](#local-development)
- [GitHub OAuth Setup](#github-oauth-setup)
- [Supabase (Database)](#supabase-database)
- [Deployment on Render](#deployment-on-render)
- [Security Model](#security-model)
- [HTTP API](#http-api)
- [Claude Code Tooling](#claude-code-tooling)
- [Limitations](#limitations)

---

## What It Does

| Area | Description |
|------|-------------|
| **Code Review** | Upload up to 10 files or paste code in any language. Each file is checked by real compilers/linters, then reviewed by Gemini for defects (logic errors, null dereferences, security flaws, resource leaks, edge cases). Findings include exact line numbers and suggested fixes. |
| **Test Execution** | Gemini writes a test file in the language's standard framework. Tests assert intended behavior from names, docstrings, and types. Tests run in a sandbox for **Python, JavaScript, Go, Ruby, and Rust**. Failures are triaged as code defects, wrong test expectations, or sandbox limitations. |
| **Repository Analysis** | Analyze any repository the signed-in GitHub account can read (including private/org repos), pinned to a specific commit. Source files in all languages are selected; vendor, build, generated, and minified files are skipped. |
| **API & Data Contracts** | Lint OpenAPI 3.x/Swagger 2.0, validate GraphQL SDL, and validate JSON/YAML against JSON Schema. With a base URL, perform live contract checks: status codes documented, JSON bodies match schema, required parameters enforced, GraphQL introspection compared to SDL. Gemini also generates a pytest suite per operation. |
| **History & Dashboards** | Overview shows each project's latest result and findings trend. History lists every run, filterable by project, type, and scope. Team dashboards combine live GitHub organization data with runs made by organization members. |

> **Nothing is simulated.** If the AI step fails (e.g., quota limit), the report says so and shows "—" instead of zero. Tests for languages without a sandbox runner are marked "not executed" and never counted as passed.

---

## How a Run Works

```text
Sign in with GitHub ─► submit code / repository / spec ─► run is queued (stored in DB)
                                                               │
         ┌────────────────────────────────────────────────────┘
         ▼
   1. Detect language (extension, shebang; unknown extensions identified by model)
   2. Static checks ...... real compilers / linters, line + column diagnostics
   3. AI review .......... Gemini: findings (severity, category, lines, fix) + test file
   4. Execute tests ...... sandboxed runner, per-test pass/fail + failure output + coverage
   5. Triage failures .... Gemini: code defect vs wrong expectation, with source line
   6. Store report ....... summary + full report in runs table ─► History / dashboards
```

The browser polls `GET /api/runs/{id}` until completion. Runs execute on a background thread pool so large repositories don't hold an HTTP request open.

---

## Language Support

Every language can be reviewed by the model. Deterministic checks and test execution depend on toolchains installed on the server (the Docker image installs all tools below).

| Language | Static Checks | Tests Executed in Sandbox |
|----------|---------------|---------------------------|
| Python | `ast` compile + pylint (errors & warnings) + AST facts | pytest + coverage |
| JavaScript (CJS/ESM) | `node --check` | `node --test` + built-in coverage |
| TypeScript | `tsc --noEmit --strict` | generated (Vitest), run locally |
| Go | `gofmt -e`, `go vet` | `go test -json -cover` |
| Rust | `rustc` (JSON diagnostics) | `rustc --test` |
| Ruby | `ruby -wc` | Minitest |
| C / C++ | `gcc` / `g++ -fsyntax-only -Wall -Wextra` | generated, run locally |
| Java | `javac -Xlint:all` | generated (JUnit 5), run locally |
| PHP | `php -l` | generated (PHPUnit), run locally |
| Shell | `bash -n`, shellcheck | generated (bats), run locally |
| Kotlin, C#, Swift, Scala, Dart, SQL, R, Lua, Perl, Haskell, Elixir, Solidity, Terraform, … | AI review only | generated in conventional framework, run locally |

When a toolchain is missing, the report says "unavailable" rather than silently skipping. Diagnostics caused by missing third-party dependencies are downgraded to *info* with the note "dependency not available in the analysis sandbox".

---

## API & Spec Checks

### OpenAPI / Swagger Lint
- Unsupported version; missing `info.title` / `info.version`; empty `paths`
- Unresolved local `$ref`; path template variables without matching `in: path` parameter
- Path parameters not marked `required`; duplicate parameters and `operationId`s
- Missing `responses`; no 2xx/3xx response; invalid status codes; responses without description
- `requestBody` without content or schema; bodies on GET/HEAD/DELETE
- Undefined security schemes; `required` properties not defined; missing `servers`
- Every issue carries a JSON pointer and, where possible, a line number

### GraphQL
Syntax errors and SDL/schema validation from `graphql-core` (unknown types, invalid root types, etc.), with line and column.

### JSON / YAML
Syntax errors with line and column, duplicate keys (which most parsers silently drop), JSON Schema validation (Draft 4 to 2020-12), and meta-validation when the document is itself a JSON Schema.

### Live Contract Checks (optional `base_url`)
Each operation is called with values from examples, defaults, enums, or the schema. The checker verifies:
- Returned status code is documented
- JSON response validates against documented schema (OpenAPI 3.0 `nullable` handled)
- Omitting required query parameters returns 4xx
- Write methods (POST/PUT/PATCH/DELETE) are sent only when explicitly opted in
- For GraphQL: server introspection compared with SDL type-by-type, field-by-field

---

## Project Layout

```
backend/
  main.py            FastAPI app: pages, /auth, /api routes, security headers
  config.py          settings from environment / backend/.env
  auth.py            GitHub OAuth, server-side sessions, encrypted GitHub tokens
  db.py              SQLAlchemy Core schema + queries (Supabase Postgres or SQLite)
  jobs.py            background run execution + progress
  gemini_client.py   Gemini REST client: strict model, retries, quota circuit breaker
  languages.py       language detection + test framework per language
  static_checks.py   compiler / linter diagnostics
  sandbox.py         isolated subprocess execution (scrubbed env, rlimits, privilege drop)
  executors.py       run generated tests and parse results (pytest, node, go, ruby, rust)
  code_review.py     per-file pipeline: static → AI review → tests → triage
  repo_analyzer.py   repository file selection + parallel analysis
  github_api.py      GitHub REST client (repos, orgs, members, trees, blobs)
  spec_parser.py     OpenAPI / GraphQL operation extraction
  spec_checks.py     lint, validation, live contract checks, test generation for specs
  spec_llm.py        prompts for API test generation
  parser.py          Python AST fact extraction
  dashboards.py      overview + team aggregations
  test_*.py          pytest suite (network mocked; language runners are real)
frontend/
  index.html         sign-in page
  app.html           application shell
  assets/            CSS, ES-module views, icons (no build step)
supabase/migrations/ database schema with Row Level Security
examples/            sample specs and deliberately buggy files for demos
Dockerfile, render.yaml
```

---

## Configuration

All settings are environment variables. For local development, copy `backend/.env.example` to `backend/.env` (gitignored; never commit real keys).

| Variable | Required | Description |
|----------|----------|-------------|
| `GEMINI_API_KEY` | yes (for AI) | Google AI Studio key. Used only by the server, sent in `x-goog-api-key` header. |
| `GEMINI_MODEL` | no | Main model, tried first. Default: `gemini-2.5-pro`. |
| `GEMINI_FALLBACK_MODELS` | no | Comma-separated fallbacks tried in order. Default: `gemini-3.6-flash,gemini-3-flash-preview,gemini-3.1-flash-lite`; `none` = no fallback. |
| `GEMINI_MAX_RETRIES` | no | Retries on 429/5xx with exponential backoff (default 4). |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | yes | GitHub OAuth App credentials. |
| `SESSION_SECRET` | production | Random string; encrypts stored GitHub tokens. Auto-generated for local dev. |
| `DATABASE_URL` | production | Supabase Postgres connection string. Empty = SQLite in `backend/data/`. |
| `APP_BASE_URL` | yes | Public URL (e.g. `https://aitestge.stream`). Builds the OAuth callback URL. |
| `APP_ENV` | no | `production` enables HTTPS/www redirects, HSTS, secure cookies. |
| `ALLOWED_HOSTS` | no | Comma-separated Host allowlist (TrustedHostMiddleware). |
| `ALLOW_PRIVATE_TARGETS` | no | Allow live checks/spec URLs on localhost/private IPs. Default `true` in dev, `false` in prod. |
| `MAX_REPO_FILES` | no | Upper bound for files per repository run (default 25, max 100). |
| `JOB_WORKERS` | no | Concurrent runs (default 3). |

### Gemini Models, Availability & Quota

Each AI call tries `GEMINI_MODEL` first, then each fallback in order. A model is skipped when Google answers:

| Response | Meaning | Skipped For |
|----------|---------|-------------|
| HTTP 404 "no longer available to new users" | Key's project cannot use the model | 15 min |
| HTTP 429 with `limit: 0` | Model has no free-tier quota; needs billing | 15 min |
| HTTP 429, daily quota | Key's daily requests for model used up | 15 min |
| HTTP 429, per-minute quota | Too many requests this minute | 20 s or Google's retry delay |
| HTTP 503 "high demand" | Google overloaded (after one quick retry) | 1 min |

The last model gets the full retry budget. Every report records the model that actually answered. If no model answers, the file's AI step says why for each model; compiler/linter results and live contract checks still complete. Nothing is invented when the AI step fails.

> One file uses one request, or two when tests fail and need triage. The free tier has small per-model daily limits — enable billing on the Google AI project for real use. Pro models "think" before answering; those thinking tokens are billed as output tokens.

---

## Local Development

**Prerequisites:** Python 3.11+. Node.js, Go, Ruby, rustc, a JDK, gcc, and PHP are optional; each one you install enables its checks and runners.

```bash
# Clone and set up
git clone https://github.com/TanishMhalsekar01/AI-test-generator.git
cd AI-test-generator/backend
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                   # then fill in the values
python -m uvicorn main:app --reload
```

Open <http://localhost:8000> and sign in with GitHub. For local sign-in, create a separate OAuth App with callback URL `http://localhost:8000/auth/github/callback`.

### Tests

```bash
cd backend
python -m pytest -q
```

183 tests cover auth and sessions, the Gemini client (retries, quota handling, key redaction), compiler and linter parsing, real sandbox runners for every supported language, the review pipeline, repository selection, spec linting, live contract checks (HTTP mocked with `responses`), and history/team access control. Gemini and GitHub are always mocked; the test suite never uses the key in `backend/.env`. CI runs the suite on every pull request ([workflow](.github/workflows/pytest.yml)).

---

## GitHub OAuth App

1. GitHub → **Settings → Developer settings → OAuth Apps → New OAuth App**
2. **Homepage URL:** `https://aitestge.stream`
3. **Authorization callback URL:** `https://aitestge.stream/auth/github/callback`
4. Copy **Client ID**, generate **Client secret**, and set both as environment variables.

The app requests `read:user`, `read:org`, and `repo` scopes — needed to list organizations and analyze private repositories. If an organization restricts third-party OAuth apps, an organization owner must approve the app before its repositories and members appear on the team dashboard.

---

## Supabase (Database)

1. Create a Supabase project.
2. Apply the schema: open the **SQL editor** and run [`supabase/migrations/20260928000000_init.sql`](supabase/migrations/20260928000000_init.sql), or run `supabase db push` with the Supabase CLI.
3. **Project Settings → Database → Connection string → Session pooler**. Copy the URI, fill in the database password, and set it as `DATABASE_URL`. The session pooler works over IPv4, which Render requires.

Row Level Security is enabled on all tables with no policies. The public Supabase API (anon/authenticated keys) therefore cannot read sessions or reports; only the backend's direct database connection can.

---

## Deployment on Render

1. Push this repository to GitHub. In Render: **New → Blueprint** → select the repository. [`render.yaml`](render.yaml) creates the Docker web service `ai-test-generator` with a health check at `/healthz` and domains `aitestge.stream` and `www.aitestge.stream`.
2. Enter the environment variables marked `sync: false` in the Render dashboard: `GEMINI_API_KEY`, `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `DATABASE_URL`, and `APP_BASE_URL` (e.g. `https://aitestge.stream`). `SESSION_SECRET` is generated automatically.
3. **Register the domain** `aitestge.stream` with any registrar if you haven't already.
4. **DNS:** In Render, open the service → **Settings → Custom Domains** and create the records it shows at your DNS provider:

| Host | Type | Value |
|------|------|-------|
| `aitestge.stream` (apex) | `A` (or `ALIAS`/`ANAME` → `ai-test-generator.onrender.com`) | `216.24.57.1` |
| `www` | `CNAME` | `ai-test-generator.onrender.com` |

Remove any conflicting `AAAA` records for the apex. Render issues TLS certificates automatically once DNS resolves. The app forces HTTPS in production (HSTS).
5. Update the GitHub OAuth App's callback URL to `https://aitestge.stream/auth/github/callback`.

> Until DNS is live, set `APP_BASE_URL` to the service's `https://<name>.onrender.com` URL and use `https://<name>.onrender.com/auth/github/callback` as the OAuth App's callback URL. In production, `www.aitestge.stream` redirects to `aitestge.stream` and HTTP redirects to HTTPS.

---

## Security Model

- **Secrets stay on the server.** The Gemini key is read from the environment, sent only in a request header, redacted from any error text, and never included in API responses. `backend/.env` is gitignored.
- **Sessions.** The browser holds only a random session ID in an HttpOnly, `SameSite=Lax` cookie (`Secure` in production). The database stores a SHA-256 of that ID. The GitHub access token is encrypted with Fernet using a key derived from `SESSION_SECRET`.
- **Access control.** Every `/api/*` route requires a session. Users see their own runs, plus runs on repositories owned by GitHub organizations they belong to (membership checked live against GitHub).
- **Sandbox.** Submitted code and generated tests run in a temporary directory with a scrubbed environment (no API keys, OAuth secrets, or database URL), CPU, file-size, and core-dump limits, and a timeout that kills the whole process group. In the Docker image, each process drops to the unprivileged `sandbox` user, so it cannot read the server's environment or files. Network access from the sandbox is not blocked (live API tests need it). For hostile multi-tenant use, run the service in its own isolated container or VM.
- **SSRF guard.** In production, spec URLs and live-check base URLs that resolve to private, loopback, link-local, or reserved addresses are rejected.
- **Browser hardening.** Strict Content-Security-Policy (no inline scripts/styles, no third-party origins except GitHub avatars), `X-Frame-Options: DENY`, `nosniff`, HSTS in production. All user-controlled and model-generated text is rendered with `textContent`, never as HTML.

---

## HTTP API

All endpoints except sign-in and `/healthz` require the session cookie. Interactive docs at `/api/docs`.

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/auth/github/login` | Start GitHub sign-in |
| GET | `/auth/github/callback` | OAuth callback |
| POST | `/auth/logout` | End the session |
| GET | `/api/me` | Signed-in user, model name, available runners |
| POST | `/api/runs/code` | Multipart: `files[]` or `code` + `filename`; optional `project`, `run_tests` |
| POST | `/api/runs/repo` | JSON: `{ "repo": "owner/name" \| URL, "branch"?, "max_files"? }` |
| POST | `/api/runs/spec` | Multipart: `file` \| `spec_text` \| `spec_url`; optional `schema_file`, `base_url`, `allow_mutations`, `generate_tests` |
| GET | `/api/runs` | History: `scope=mine\|all\|team:<org>`, `kind`, `project`, `limit`, `offset` |
| GET / DELETE | `/api/runs/{id}` | Run with full report / delete your own run |
| GET | `/api/github/repos`, `/api/github/orgs` | Live GitHub data for the signed-in user |
| GET | `/api/dashboard/overview` | Per-project latest results and trends |
| GET | `/api/dashboard/team/{org}` | Organization dashboard (members only) |

`POST /api/runs/*` endpoints return `202 {"id": ...}`; poll `GET /api/runs/{id}` until `status` is `completed` or `failed`.

---

## Claude Code Tooling

This repository ships project-level Claude Code configuration:

- **Plugins** (declared in [`.claude/settings.json`](.claude/settings.json), from `anthropics/claude-plugins-official`): **Supabase** (database/auth management via MCP), **Playwright** (browser automation MCP), **Context7** (up-to-date library documentation), and **frontend-design** (UI implementation skill).
- **Playwright CLI** skill in `.claude/skills/playwright-cli/` (from `@playwright/cli`), for driving a browser from the terminal.
- **Strix** security-testing skill in `.claude/skills/strix-scan/`, with instructions for running [Strix](https://github.com/usestrix/strix) against a local instance. Strix needs Docker and a billed LLM key.
- [`scripts/setup-dev-tools.sh`](scripts/setup-dev-tools.sh) installs all of the above on a developer machine.

---

## Limitations

- AI findings are model output: they are specific and line-referenced, but review them before acting. Failing tests plus triage are the strongest evidence a defect is real.
- Tests execute only for Python, JavaScript, Go, Ruby, and Rust. Other languages get generated tests to run locally.
- Generated tests run against the single file under test. Code that depends on other project files or third-party packages may need those dependencies; such failures are triaged as *sandbox limitation*.
- Repository runs analyze up to `MAX_REPO_FILES` files per run (application code before tests).
- `gemini-2.5-pro` answers only for keys from Google AI projects that still have access to it (Google no longer offers it to new users); other keys are served by the fallback models.