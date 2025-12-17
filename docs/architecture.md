# Jobsearcher Architecture (MVP)

## Overview
Jobsearcher tracks recent job openings from company career pages and surfaces them to authenticated users. The MVP prioritizes freshness, automatic removal of expired roles, and resilient parsing aided by an LLM fallback when deterministic scrapers are uncertain.

## Core Components

### 1. Scheduler
- Runs twice daily to trigger company crawls.
- Enqueues crawl jobs for each active company within user limits.
- Retries transient failures without blocking subsequent runs.

### 2. Crawler
- Fetches public career pages and relevant subpages (respecting robots.txt when feasible).
- Detects ATS providers (Greenhouse, Lever, Workday) to apply deterministic parsers when possible.
- Produces normalized HTML/text payloads for downstream parsing.

### 3. Parsing Pipeline
- **Deterministic parser**: Extracts job title, location, department, job type, job URL, and posted date when structure is recognized.
- **Confidence evaluator**: Marks extraction as low confidence when required fields are missing, structure diverges from expectations, or completeness is low.
- **LLM-assisted parser**: Invoked on low-confidence outputs to infer fields from raw HTML/text. LLM output is always subject to deterministic validation (e.g., required fields present, URLs valid).
- **Normalization**: Standardizes job type values, trims whitespace, and normalizes department/team names.

### 4. Job Lifecycle Manager
- **Creation**: Inserts new jobs detected during a crawl using posted date rules.
- **Removal**: Immediately deletes jobs that disappear from a company’s current crawl results.
- **Expiration**: Deletes jobs older than 30 days, using scraped posted date when available or system first-seen otherwise.
- **Uniqueness**: Jobs are uniquely identified by `(job_url, location)` to support multi-location roles.

### 5. API Layer
- Authenticated endpoints for adding companies, listing tracked jobs, and filtering.
- Enforces company tracking limits per subscription tier.
- Exposes posted date and posted date source to the client.

### 6. UI Layer
- Views: per-company and aggregated feed.
- Default sort: newest posted date.
- Filters: company, location, job type, and job title keywords.
- Indicates whether posted date is scraped or system-inferred.

## Data Model (Relational)

### users
- `id` (pk)
- `email`, `password_hash`
- `subscription_tier` (enum: FREE, PAID_X)
- `created_at`, `updated_at`

### companies
- `id` (pk)
- `name`
- `careers_url`
- `validated_at` (nullable if validation fails)
- `created_at`, `updated_at`

### user_companies
- `user_id` (fk users)
- `company_id` (fk companies)
- `created_at`
- Unique on `(user_id, company_id)` to avoid duplicates.

### jobs
- `id` (pk)
- `company_id` (fk companies)
- `title`
- `location`
- `department`
- `job_type` (enum: FULL_TIME, INTERN, CONTRACT, OTHER)
- `job_url`
- `posted_date`
- `posted_date_source` (enum: SCRAPED, SYSTEM_INFERRED)
- `first_seen_at`
- `created_at`, `updated_at`
- Unique on `(job_url, location)`

## Key Flows

### Adding a Company
1. User submits company name or careers URL via authenticated request.
2. System validates careers page reachability; on failure returns an error.
3. Upon success, company is stored and queued for crawl in next scheduler run.

### Twice-Daily Refresh
1. Scheduler enqueues crawl tasks for active companies.
2. Crawler fetches pages, applies deterministic parsing, and routes low-confidence results through LLM-assisted extraction.
3. Lifecycle manager inserts new jobs, removes missing ones, and expires listings older than 30 days.
4. Results are committed atomically per company to avoid partial updates.

### Job Retrieval
- API serves per-company or aggregated feeds, sorted by posted date (newest first) with optional filters.
- Responses include posted date source for transparency.

## Observability & Failure Handling
- Log crawl outcomes (success, partial, failure) with reasons.
- Record parsing confidence to monitor LLM usage effectiveness.
- Alert on sustained crawl failures for a company to avoid stale data.
- LLM failures fall back to deterministic results; lifecycle rules still apply.

## LLM Usage Controls
- LLM parsing is invoked only when deterministic extraction is low-confidence.
- Usage is gated by subscription tier; free users have limited LLM calls while paid tiers allow higher throughput.
- Bring-your-own-key support can be layered for advanced tiers without changing parsing flow.
- Deterministic validation (required fields present, URLs valid) always executes after LLM output and can discard malformed results.

## Freshness & Data Hygiene
- Crawls run twice per day; missed runs trigger requeueing rather than skipping the next window.
- Closed jobs disappear as soon as they are absent from a crawl result.
- Expiration deletes jobs older than 30 days using scraped posted date when available, otherwise the first-seen timestamp.
- Atomic per-company updates prevent partial refreshes from leaking stale jobs to users.

## Scalability Considerations
- Queue-based scheduling scales horizontally for crawls.
- Parsing pipeline is stateless and can run in workers.
- Deduplicating on `(job_url, location)` prevents duplicate entries when multiple crawls overlap.

