# FinSight AI

> An AI-ready personal finance backend that turns receipts and bank statements into structured expenses, insights, and forecasts.

FinSight AI is a portfolio project built to demonstrate practical backend engineering with Python and FastAPI first, then evolve into an AI application. The project intentionally starts with a small, reliable expense-management API before adding OCR, NLP, forecasting, Redis, and background workers.

## Why this project?

Most beginner portfolio projects stop at CRUD. FinSight AI is designed around problems that are useful in real systems and interesting in a technical interview:

- transactional consistency when creating an expense, consuming an AI credit, and writing an audit log;
- concurrent requests competing for the last available credit;
- rate limiting and short-lived caching with Redis;
- asynchronous document processing instead of keeping an HTTP request open;
- OCR and NLP pipelines with measurable rule-based baselines;
- query optimization, indexes, tests, and production-oriented API design.

The goal is not to add every technology at once. Each phase should be usable, tested, and explainable before the next phase begins.

## Product scope

A user can:

1. Register and sign in.
2. Record and manage expenses manually.
3. Upload a receipt image or bank-statement PDF.
4. Track the processing job while OCR and extraction run in the background.
5. Review and correct the extracted merchant, date, amount, and category.
6. View monthly summaries, category breakdowns, and spending trends.
7. Later, receive forecasts and anomaly warnings based on historical spending.

## Architecture

```text
Client
  |
  v
FastAPI API
  |
  +-- Auth / Expenses / Documents / Analytics
  |
  v
Service layer -----------------------------+
  |                                        |
  +-- Repository layer --> PostgreSQL      +--> AI pipeline
  |                         |                    |-- OCR
  |                         |-- transactions    |-- entity extraction
  |                         |-- locking         |-- classification
  |                         +-- indexes          +-- forecasting
  |
  +-- Redis
      |-- rate limiting
      |-- job status
      +-- short-lived, user-scoped cache
```

The dependency direction is kept simple:

```text
API -> Services -> Repositories -> Database
                  |
                  +-> AI / cache / worker adapters
```

- **API layer** handles HTTP input, authentication dependencies, and response codes.
- **Service layer** owns business rules and transaction boundaries.
- **Repository layer** contains database queries, not business decisions.
- **Schemas** are API contracts; **models** are database mappings. They are kept separate.
- **AI modules** expose normal Python interfaces so models can be replaced without rewriting endpoints.

## Planned database model

```text
User
 |-- Expense --------> Category
 |      |
 |      +------------> Document
 |
 |-- MonthlySummary
 |-- CreditBalance
 +-- UsageLog

Document
 +-- ProcessingJob
```

Suggested core tables:

| Table | Responsibility |
| --- | --- |
| `users` | identity and account state |
| `categories` | user or system expense categories |
| `expenses` | normalized financial transactions |
| `documents` | uploaded-file metadata and processing state |
| `processing_jobs` | asynchronous OCR/AI job progress |
| `credit_balances` | remaining AI-processing credits |
| `usage_logs` | immutable audit trail for credit usage |
| `monthly_summaries` | optional precomputed analytics |

Do not store uploaded financial documents in the database. Store an object key and metadata; use local storage in development and an object-storage adapter later.

## Transaction and concurrency case

When document analysis succeeds, one database transaction should:

```text
BEGIN
  consume one credit atomically
  create the expense
  update or invalidate the monthly summary
  append a usage log
  mark the document as completed
COMMIT
```

If any step fails, all steps roll back. To prevent two simultaneous uploads from spending the same final credit, use an atomic conditional update:

```sql
UPDATE credit_balances
SET remaining = remaining - 1
WHERE user_id = :user_id
  AND remaining > 0;
```

An affected-row count of zero means the user has no credit. This scenario should have an integration test with concurrent requests.

## Redis usage

Redis is introduced only where it has a clear purpose:

- per-user upload rate limiting;
- temporary processing progress with a TTL;
- short-lived analytics cache;
- idempotency keys for upload requests;
- optional task queue transport when a worker is introduced.

Financial extraction results must never be cached globally by document content alone. Any result cache must be scoped by user and document hash, contain no unnecessary raw data, and expire quickly.

## Project structure

```text
.
|-- app/
|   |-- main.py                 # FastAPI application factory
|   |-- api/
|   |   |-- deps.py             # shared FastAPI dependencies
|   |   +-- v1/
|   |       |-- router.py       # versioned API router
|   |       +-- endpoints/      # auth, expenses, documents, analytics
|   |-- core/
|   |   |-- config.py           # environment-based settings
|   |   +-- security.py         # password and token helpers
|   |-- db/
|   |   |-- base.py             # SQLAlchemy metadata/model imports
|   |   +-- session.py          # engine and session factory
|   |-- models/                 # SQLAlchemy persistence models
|   |-- schemas/                # Pydantic request/response models
|   |-- repositories/           # database access
|   |-- services/               # use cases and transaction boundaries
|   |-- ai/
|   |   |-- ocr/                # image/PDF-to-text adapters
|   |   |-- extraction/         # merchant/date/amount extraction
|   |   |-- classification/     # expense category prediction
|   |   +-- forecasting/        # spending forecast and anomalies
|   +-- workers/                # background job entry points
|-- tests/
|   |-- unit/
|   +-- integration/
|-- .env.example
|-- .gitignore
|-- pyproject.toml
+-- README.md
```

Folders that belong to later phases contain placeholders so the intended architecture is visible without pretending those features already exist.

## Current status

The repository currently contains the Phase 0 foundation:

- FastAPI application factory;
- versioned `/api/v1` router;
- health-check endpoint;
- environment-based configuration;
- initial test setup and code-quality configuration;
- package boundaries for future backend and AI features.

No database, authentication, Redis, or AI functionality is claimed as implemented yet.

## Getting started

### Requirements

- Python 3.12+
- `pip`

### Local setup

```bash
python -m venv .venv
```

Activate the virtual environment:

```powershell
# Windows PowerShell
.venv\Scripts\Activate.ps1
```

```bash
# macOS/Linux
source .venv/bin/activate
```

Install the project with development dependencies:

```bash
python -m pip install -e ".[dev]"
```

Create the local environment file:

```powershell
Copy-Item .env.example .env
```

Run the API:

```bash
uvicorn app.main:app --reload
```

Open:

- API documentation: <http://127.0.0.1:8000/docs>
- Health check: <http://127.0.0.1:8000/api/v1/health>

### Quality checks

```bash
pytest
ruff check .
ruff format --check .
```

## API roadmap

### Phase 1 — Backend MVP

```text
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh

POST   /api/v1/expenses
GET    /api/v1/expenses
GET    /api/v1/expenses/{expense_id}
PATCH  /api/v1/expenses/{expense_id}
DELETE /api/v1/expenses/{expense_id}

GET    /api/v1/categories
GET    /api/v1/analytics/monthly
```

Deliverables:

- PostgreSQL and async SQLAlchemy;
- Alembic migrations;
- JWT access/refresh tokens and password hashing;
- ownership checks, filtering, pagination, and error handling;
- unit and API integration tests.

### Phase 2 — Document ingestion

```text
POST /api/v1/documents
GET  /api/v1/documents/{document_id}
GET  /api/v1/jobs/{job_id}
POST /api/v1/documents/{document_id}/confirm
```

Start with a rule-based baseline:

1. Validate file type and size.
2. Extract text with OCR.
3. Extract amount/date/merchant using regex and rules.
4. Classify using merchant keywords.
5. Require user confirmation before the expense becomes final.

Record precision, recall, latency, and representative failure cases before adding a model.

### Phase 3 — Transactions and concurrency

- credit balance and immutable usage log;
- atomic credit consumption;
- one transaction for finalized AI results;
- idempotent confirmation endpoint;
- rollback and concurrent-request tests.

### Phase 4 — Redis

- upload rate limiter returning HTTP `429`;
- analytics cache with invalidation after expense changes;
- temporary job progress;
- upload idempotency keys.

### Phase 5 — Production engineering

- Docker and Docker Compose;
- PostgreSQL/Redis integration tests;
- CI for linting, tests, and migration checks;
- structured logs, request IDs, and basic metrics;
- database indexes verified with `EXPLAIN ANALYZE`.

At this point the project is strong enough for a backend internship portfolio. The remaining phases add the AI story.

### Phase 6 — NLP classification

- preserve rules as a baseline;
- build and version a labeled dataset;
- train a scikit-learn text-classification baseline;
- optionally fine-tune a Hugging Face model;
- compare accuracy, macro F1, latency, and memory use;
- load the selected model behind an interface owned by `app/ai`.

### Phase 7 — Forecasting and anomaly detection

- only forecast when enough history exists;
- compare against a naive previous-month baseline;
- return confidence/error information rather than false precision;
- detect category-level spending anomalies;
- prevent data leakage with time-based train/test splits.

### Phase 8 — Background worker and deployment

- move OCR/inference out of the request process;
- add retries, timeouts, dead-letter handling, and idempotency;
- store uploads outside the API container;
- deploy API, worker, PostgreSQL, and Redis;
- document operational trade-offs and measured performance.

## Definition of done for each feature

A feature is complete when it has:

- an explicit request/response contract;
- validation and authorization;
- business logic outside the route handler;
- expected error cases;
- tests at the appropriate level;
- a migration when persistence changes;
- logs without secrets or raw financial content;
- README or API documentation updates.

## Security and privacy checklist

- Never commit secrets or real financial documents.
- Validate MIME type, extension, and file size; generate server-side filenames.
- Reject encrypted or unsupported PDFs clearly.
- Keep user data isolated in every query and cache key.
- Do not log access tokens, passwords, raw OCR text, or bank details.
- Encrypt traffic in production and define retention/deletion behavior.
- Treat extracted results as suggestions until the user confirms them.
- Use synthetic or explicitly anonymized data for demos and model training.

This is a portfolio system, not financial advice, accounting software, or a production banking platform.

## Interview demo script

A concise demo should show engineering decisions, not only screens:

1. Create a user and add a manual expense.
2. Upload a synthetic receipt and receive `202 Accepted` with a job ID.
3. Poll the job, inspect the extracted draft, and correct one field.
4. Confirm it and show the expense, credit balance, and usage log updated together.
5. Send two requests against one remaining credit and show only one succeeds.
6. Trigger the upload rate limit.
7. Show a cached analytics request and explain invalidation.
8. Run the tests and discuss one measured AI baseline improvement.

## Suggested resume bullet — only after implementation

> Built an AI-ready expense-processing SaaS with FastAPI, PostgreSQL, and Redis, using transactional credit accounting, concurrency-safe updates, rate limiting, asynchronous OCR processing, and tested expense analytics.

Replace broad claims with measured results such as test coverage, p95 latency, cache hit rate, classification F1, or query-time improvements. Never put roadmap-only features on a resume.

## License

Choose a license before publishing the repository. MIT is a common option for a public portfolio project.
